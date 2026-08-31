# 11. Guards and Authorization

## What "protecting a route" actually means

The slides frame this well: protecting a route means restricting access to it so only authorized requests get through, for example only logged in users, or only users with a specific role like admin. A guard is the NestJS building block designed specifically for this decision: should this particular request be allowed to continue, or should it be stopped right here?

## The `CanActivate` interface

Every guard is a class decorated with `@Injectable()` that implements the `CanActivate` interface, meaning it must have a method called `canActivate` that returns `true` (let the request through), `false` (block it), or a `Promise`/`Observable` that eventually resolves to `true` or `false` (for when the decision requires asynchronous work, like checking a database or calling an external auth service).

## Example 1: a simple token check with `AuthGuard`

`src/guards/auth/auth.guard.ts`:

```ts
@Injectable()
export class AuthGuard implements CanActivate {
  canActivate(
    context: ExecutionContext,
  ): boolean | Promise<boolean> | Observable<boolean> {
    const request = context.switchToHttp().getRequest();
    const authHeader = request.headers['authorization'];

    return authHeader === 'Bearer my-secret-token';
  }
}
```

`context: ExecutionContext` is an object NestJS gives every guard, representing the current request in a protocol-agnostic way (the same guard system works for HTTP, WebSockets, and microservices, `ExecutionContext` is the abstraction that makes that possible). `context.switchToHttp()` narrows that generic context down specifically to the HTTP case, and `.getRequest()` then gives you the actual underlying request object (the same kind of `request` object Express normally hands you).

`request.headers['authorization']` reads the `Authorization` header the client sent with their request. The guard's entire decision is a single line: `return authHeader === 'Bearer my-secret-token'`. If the client sent exactly the header `Authorization: Bearer my-secret-token`, the guard returns `true` and the request proceeds. Otherwise it returns `false`, and NestJS automatically responds with a `403 Forbidden` error, without the controller method ever running. This is obviously a toy example (a real app would check the token against a database or decode and verify a JWT), but it demonstrates the exact mechanism real authentication guards use.

This guard is applied in `src/product/product.controller.ts`:

```ts
@Get()
@UseGuards(AuthGuard)
getProducts(){
    return this.productService.getAllProducts();
}
```

`@UseGuards(AuthGuard)` attaches the guard to just this one route. Only `GET /product` requires the header; `GET /product/:id` right below it in the same controller has no guard and remains open to anyone.

## Example 2: role based access with `RolesGuard`, a custom decorator, and `Reflector`

This is a more advanced and more realistic pattern: instead of hardcoding "only admins can access this route" logic separately into every route that needs it, you build one reusable guard plus a small custom decorator that lets you declare which roles are allowed, right on each route.

First, the enum of possible roles, `src/guards/roles/roles.enums.ts`:

```ts
export enum Role {
    User = 'user',
    Admin = 'admin'
}
```

An `enum` (short for enumeration) is a TypeScript feature for defining a fixed set of named constant values. Using `Role.Admin` everywhere instead of the raw string `'admin'` prevents typos (misspelling `'admni'` would be caught by TypeScript, misspelling a raw string would not be).

Next, the custom decorator, `src/guards/roles/roles.decorator.ts`:

```ts
import { SetMetadata } from "@nestjs/common";

export const ROLES_KEY = 'roles';

export const Roles = (...roles: string[]) => SetMetadata(ROLES_KEY, roles);
```

This is your first look at building your own decorator, and it is simpler than it looks. `SetMetadata(key, value)` is a NestJS helper function that attaches a small piece of arbitrary metadata to whatever it is used on (a class or a method). `Roles` is defined as a function that takes any number of role strings (`...roles: string[]` is JavaScript's "rest parameter" syntax, collecting every argument into an array) and immediately calls `SetMetadata('roles', roles)`. Calling `Roles('admin')` therefore produces a decorator that, when placed above a route, tags that route with the metadata `{ roles: ['admin'] }`. `ROLES_KEY` is just the string `'roles'` pulled out into a shared constant, so both the decorator and the guard that reads it later refer to the exact same key and can never accidentally drift apart from a typo.

Now, the guard that reads that metadata back out, `src/guards/roles/roles.guard.ts`:

```ts
@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector){}
  canActivate(
    context: ExecutionContext,
  ): boolean | Promise<boolean> | Observable<boolean> {
    const requiredRoles = this.reflector.getAllAndOverride<Role[]>(
      ROLES_KEY, [
        context.getHandler(),
        context.getClass(),
      ]
    );
    if(!requiredRoles) return true;
    const request = context.switchToHttp().getRequest<{ headers: Record<string, string>}>();
    const userRole = request.headers['x-user-role'] as Role;
    return requiredRoles.includes(userRole);
  }
}
```

`Reflector` (injected through the constructor, exactly the way any other provider would be, see [07-dependency-injection.md](07-dependency-injection.md)) is the tool that reads metadata that was attached with `SetMetadata`. `this.reflector.getAllAndOverride<Role[]>(ROLES_KEY, [context.getHandler(), context.getClass()])` looks for metadata stored under the key `'roles'`, checking first the specific route handler method (`context.getHandler()`) and falling back to the whole controller class (`context.getClass()`) if the method itself has none. This lets you put `@Roles(...)` either on one specific route, or on an entire controller to cover every route inside it.

`if(!requiredRoles) return true;` is an important detail: if a route has no `@Roles(...)` decorator at all, `requiredRoles` will be `undefined`, and the guard lets the request through unconditionally. This means `RolesGuard` only restricts routes that were explicitly marked, it does nothing on its own to a route with no `@Roles` decorator attached to it.

If there are required roles, the guard reads a custom header, `x-user-role`, off the request, and checks whether that role is included in the list of roles allowed for this route. This is, again, a simplified stand-in for how a real app would determine the current user's role (typically decoded from an authentication token rather than trusted directly from a raw header a client could set to anything), but the authorization logic pattern itself (declare allowed roles on a route, check the current user's role against them in a guard) is exactly how production role based access control works.

Finally, seeing it all wired together in `src/user-roles/user-roles.controller.ts`:

```ts
@Controller('user-roles')
export class UserRolesController {
    @Get('admin-data')
    @UseGuards(RolesGuard)
    @Roles(Role.Admin)
    getAdminData(){
        return { message: 'Only admin can access'}
    }
    @Get('user-data')
    getUserData(){
        return { message: 'Anyone can access'}
    }
}
```

`GET /user-roles/admin-data` has both `@UseGuards(RolesGuard)` (activate the guard on this route) and `@Roles(Role.Admin)` (tell the guard the required role is `'admin'`). Sending this request with header `x-user-role: admin` succeeds; sending it with `x-user-role: user`, or no such header at all, gets rejected with `403 Forbidden`. `GET /user-roles/user-data` has neither decorator, so it is open to anyone.

## Guards versus middleware, briefly

The slide deck draws this distinction directly, and it will make more sense once you read [12-middleware.md](12-middleware.md), but it is worth previewing here: middleware runs earlier and more generically (logging, decoding a token into a usable form), while guards run specifically to make an allow-or-deny decision immediately before a particular route handler executes, and guards have access to rich, route-specific information (via `ExecutionContext`, and metadata via `Reflector`) that plain middleware does not.

## Key terms recap

Guard: a class implementing `CanActivate`, deciding whether a request may proceed to its route handler.

`ExecutionContext`: the object a guard (and other request-scoped mechanisms) receives, giving protocol-agnostic access to the current request.

`@UseGuards()`: the decorator that attaches one or more guards to a route or an entire controller.

`SetMetadata`: a NestJS helper for attaching custom metadata to a class or method, the foundation for building custom decorators like `@Roles()`.

`Reflector`: a NestJS provider used to read metadata (attached via `SetMetadata`) back out at runtime, most commonly inside a guard.
