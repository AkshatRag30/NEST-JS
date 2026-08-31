# 9. REST API and HTTP Methods

## What an API is, in the most basic terms

API stands for Application Programming Interface. It is any defined way for two pieces of software to talk to each other. In a web backend context, the "API" is the set of URLs your server exposes, and the rules for what you can send to them and what you get back. Your frontend (a website or mobile app) is the client, and it talks to your NestJS app over HTTP using that API.

## What makes an API a REST API specifically

REST stands for Representational State Transfer. It is a set of conventions (not a strict protocol like some others) for designing APIs. The core ideas relevant to a beginner are:

Everything is a resource, addressed by a URL. In this repo, `/student`, `/category`, `/customer`, `/product` are all resources.

You use standard HTTP methods to say what you want to do to that resource, rather than inventing your own verbs in the URL. This is the biggest thing that separates a REST API from a poorly designed one. A non-REST style API might have URLs like `/getStudents`, `/createStudent`, `/deleteStudent`. A REST style API instead has one URL per resource, `/student`, and expresses the action through the HTTP method: `GET /student` to read, `POST /student` to create, `DELETE /student/:id` to remove.

Communication is stateless. Each request from the client to the server must carry everything the server needs to understand it; the server does not remember anything about "the client's session" between separate requests purely from the connection itself (any session-like behavior has to be built explicitly, for example with tokens, which is what the `AuthGuard` in [11-guards-and-authorization.md](11-guards-and-authorization.md) is a small taste of).

## The five HTTP methods, and exactly what each one means

The slide deck defines these cleanly, and `src/student/student.controller.ts` gives you a working example of every single one:

`GET`, used to read or fetch data, and it should never change anything on the server. `getAll()` and `getOne()` in `StudentController` are both `GET` routes, and both only read from the `students` array, never modify it.

`POST`, used to create new data. `create()` is a `POST` route; it adds a brand new student to the array.

`PUT`, used to update existing data completely, meaning you are expected to send the entire new representation of the resource, replacing what was there before. Look at `updateStudent` in `src/student/student.service.ts`:

```ts
updateStudent(id: number, data: {name: string; age: number}){
    const index = this.students.findIndex((s) => s.id === id);
    if(index === -1) throw new NotFoundException('Student not found!');
    this.students[index] = { id, ...data};
    return this.students[index];
}
```

Notice `this.students[index] = { id, ...data}` completely replaces the object at that index. If `data` were missing a field, that field would simply not exist in the new object, this is the literal meaning of "update completely."

`PATCH`, used to partially update existing data, meaning you only send the fields you actually want to change. Look at `patchStudent`:

```ts
patchStudent(id: number, data: Partial<{ name: string; age: number}>){
    const student = this.getStudentById(id);
    Object.assign(student, data);
    return student;
}
```

`Object.assign(student, data)` merges `data`'s fields onto the existing `student` object, leaving any field not present in `data` untouched. This is the practical difference between `PUT` and `PATCH`, and it is a very common interview question: `PUT` replaces the whole resource, `PATCH` only changes the parts you send.

`DELETE`, used to remove data. `deleteStudent` finds the index of the matching student and calls `this.students.splice(index, 1)`, removing exactly one element starting at that index.

## Mapping HTTP methods to their decorators and their meaning together

| HTTP verb | Decorator | Meaning | Example route in this repo |
|---|---|---|---|
| GET | `@Get()` | Read data, no side effects | `GET /student`, `GET /student/:id` |
| POST | `@Post()` | Create a new resource | `POST /student`, `POST /customer` |
| PUT | `@Put()` | Replace a resource entirely | `PUT /student/:id` |
| PATCH | `@Patch()` | Partially update a resource | `PATCH /student/:id` |
| DELETE | `@Delete()` | Remove a resource | `DELETE /student/:id` |

(A small note on formatting here: this table uses a Markdown table, which renders as neat rows and columns rather than a plain hyphenated bullet list, purely for readability.)

## Why organizing an app around REST actually helps you

The slide deck's "Importance of REST API" points come alive once you look at `StudentController` as a whole. Every route follows the exact same shape: extract the id and/or body, hand it straight to the matching service method, return the result. Because every route follows this same shape, once you understand one route in this controller, you understand all six of them instantly. Any other developer opening this file for the first time can predict what `PATCH /student/5` does without reading a single line of the service, just from knowing REST conventions. That predictability is the real value: it turns "understanding this specific API" into "already knowing REST, so I already mostly understand this API."

## HTTP status codes, briefly

Although this repo's controllers mostly just `return` plain values and let NestJS pick a sensible default status code (`200 OK` for a normal successful response, `201 Created` by default for a successful `POST`), the server also needs a way to signal that something went wrong. `src/student/student.service.ts` uses NestJS's built in `NotFoundException`:

```ts
getStudentById(id: number){
    const student = this.students.find((s) => s.id === id);
    if(!student) throw new NotFoundException('Student not Found!');
    return student;
}
```

Throwing `NotFoundException` automatically produces an HTTP response with status code `404`, meaning "the resource you asked for does not exist," along with the message you passed in. This connects directly to [13-exception-filters.md](13-exception-filters.md), where you will see how a custom filter can reshape exactly what this error response looks like.

## Key terms recap

API: a defined way for two programs to communicate.

REST: an architectural style for APIs built around resources (URLs) and standard HTTP methods.

Stateless: each request is handled independently, with no memory of previous requests baked into the protocol itself.

Idempotent (a term you will encounter once you read more about REST): an operation that produces the same end result no matter how many times you repeat it. `PUT` and `DELETE` are meant to be idempotent (replacing a resource with the same data twice, or deleting an already-deleted resource, should not cause additional side effects), while `POST` is typically not (calling it twice usually creates two separate resources).
