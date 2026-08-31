# 07. The Supabase Auth Guard

## What Supabase is, in one sentence

Supabase is a backend as a service platform, built on top of a real PostgreSQL database, that bundles together a hosted database, a file storage system, and a built in authentication service, and that authentication service is what issues a signed JSON Web Token, a JWT, to a user the moment they successfully log in.

## The full guard, `supabase-auth.guard.ts`

```ts
import { CanActivate, ExecutionContext, Injectable, UnauthorizedException } from '@nestjs/common';
import { Observable } from 'rxjs';
import * as jwt from 'jsonwebtoken';
import { Request } from 'express';
import { ConfigService } from '@nestjs/config';

@Injectable()
export class SupabaseAuthGuard implements CanActivate {
  constructor(private configService: ConfigService){}

  canActivate(
    context: ExecutionContext,
  ): boolean | Promise<boolean> | Observable<boolean> {
    const request = context.switchToHttp().getRequest<Request>();
    const authHeader = request.headers['authorization'];
    if(!authHeader || !authHeader.startsWith('Bearer ')) {
      throw new UnauthorizedException('No Token Provided!');
    }
    const token = authHeader.split(' ')[1];
    const jwtSecret = this.configService.get<string>('SUPABASE_JWT_SECRET');
    if(!jwtSecret){
      throw new UnauthorizedException('JWT Secrete not found');
    }
    try{
      const decode = jwt.verify(token, jwtSecret);
      request['user'] = decode;
      return true;
    }
    catch(error) {
      throw new UnauthorizedException('Invalid Token');
    }
  }
}
```

## `implements CanActivate`, and how this guard makes its decision

`SupabaseAuthGuard` is a class decorated with `@Injectable()` that implements NestJS's `CanActivate` interface, meaning it has a `canActivate` method that NestJS calls right before a protected route's handler runs, and whose return value (or thrown exception) decides whether the request is allowed to continue. `context.switchToHttp().getRequest<Request>()` narrows the generic, protocol agnostic `ExecutionContext` object down to the real underlying Express `Request` object, exactly the same first step used by every other guard covered in the sibling NestJS course project's guards note.

## What it checks, verified by reading the actual code

This guard does two distinct things, in order, and it is worth being precise about both, since the question of exactly how a token gets validated matters a lot in practice.

First, it checks for a bearer token on the `Authorization` header. `request.headers['authorization']` reads the header, and `!authHeader || !authHeader.startsWith('Bearer ')` rejects the request immediately with a `401 Unauthorized` (`UnauthorizedException`) if that header is missing entirely, or present but not in the expected `Bearer <token>` format. If it passes, `authHeader.split(' ')[1]` pulls out just the token part, discarding the literal word `Bearer`.

Second, and this is the detail worth verifying carefully rather than assuming, it verifies that token as a JWT entirely locally, using the `jsonwebtoken` package's `jwt.verify(token, jwtSecret)` function, it does not make any network call out to Supabase's servers to ask whether the token is valid. The secret it verifies against, `SUPABASE_JWT_SECRET`, is read through `ConfigService.get<string>('SUPABASE_JWT_SECRET')`, meaning it comes from an environment variable, the same shared signing secret that a real Supabase project uses to sign every JWT it issues, and confirmed by checking `package.json`, there is no `@supabase/supabase-js` client library anywhere in this project's dependencies, only the generic `jsonwebtoken` library. `jwt.verify()` checks the token's cryptographic signature against that shared secret and checks that it has not expired, if verification succeeds, the decoded payload (whatever claims Supabase originally encoded into the token, typically things like the user's id and email) is attached directly onto the request object as `request['user'] = decode`, making it available to anything running after this guard, such as the route handler itself. If the secret is missing from the environment entirely, or if verification throws for any reason (an invalid signature, an expired token, a malformed token), the guard throws `UnauthorizedException` and the request never reaches the controller.

## Where this guard is actually applied, confirmed by searching the codebase

Searching this entire `src` folder for every reference to `UseGuards` and `SupabaseAuthGuard` turns up exactly one place it is used as an actual guard: `employees.controller.ts`, on the `findAll` method:

```ts
@UseGuards(SupabaseAuthGuard)
@Get()
async findAll(): Promise<Employee[]> {
    return this.employeeService.findAll();
}
```

That is the only route in the entire application protected by this guard. Every other route on `EmployeesController`, creating, searching, reading one employee by id, updating, and deleting, has no `@UseGuards()` at all, and `UserModule` has no controller for the guard to be applied to in the first place, see [06-user-entity-and-module-gap.md](06-user-entity-and-module-gap.md). This is worth stating plainly rather than assuming the guard is broadly wired in just because it exists and looks complete: `SupabaseAuthGuard` is a fully implemented, working piece of authentication logic, but its actual footprint in this codebase, the set of real HTTP routes it currently protects, is exactly one route, `GET /employees`.

## The environment variable this guard depends on

Just like `DATABASE_URL` covered in [01-project-setup-and-postgres-connection.md](01-project-setup-and-postgres-connection.md), `SUPABASE_JWT_SECRET` is read from the environment and is not present anywhere in this repo, there is no `.env` file shipped, and `.gitignore` confirms that is deliberate. Unlike a missing `DATABASE_URL`, though, a missing `SUPABASE_JWT_SECRET` does not stop the app from starting, `ConfigService.get()` simply returns `undefined` at request time, and the guard's own `if(!jwtSecret)` check catches that and throws a clean `401` rather than letting `jwt.verify(token, undefined)` blow up unpredictably. In other words, until someone sets a real `SUPABASE_JWT_SECRET` matching an actual Supabase project's signing secret, `GET /employees` would reject every single request, even ones carrying what looks like a well formed bearer token, since the guard would fail at the missing secret check before it ever gets to verifying the token itself.
