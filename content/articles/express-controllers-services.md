---
date: "2026-08-15"
title: "Decoupling business logic from Express with a service layer"
tags: ["Express"]
---

In early development of an Express application, it is tempting to write business loginc in route handlers: fetch the data, validate it, apply business rules, send the response — all in one function. It works, but it also means your business logic is heavily coupled to Express. Every function takes `(req, res)`, reads from `req.params` or `req.body`, and writes directly to `res`. None of that logic can be reused outside an HTTP request — not from a CLI script, a cron job, a test, or a future GraphQL resolver — without dragging Express along with it.

The fix is a service layer: plain functions that contain the actual business logic and know nothing about HTTP. Controllers become a thin adapter between Express and the services.

## Before: everything in the route handler

```ts
app.get("/users/:id", async (req, res) => {
  const user = await db.users.findById(req.params.id);
  if (!user) throw createHttpError(404, `User ${req.params.id} not found`);
  res.json(user);
});
```

This is fine as a single route. But the "not found" rule, the response shape, and the status code are all inline — copy-pasted into every handler that needs the same check. Test this logic and you're testing an Express route, `req`/`res` mocks included, for what is really just "does this repository lookup handle a missing id."

## After: services and controllers

### Services: framework-agnostic logic

A service function takes and returns plain values, not `req`/`res`:

```ts
export async function readUser(id: string): Promise<User> {
  const user = await db.users.findById(id);
  if (!user) throw new NotFoundError("User", id);
  return user;
}
```

There's no Express in sight. This function could be called from a route handler, a background job, or a unit test with equal ease.

### Controllers: thin adapters

The controller's only job is translating between the HTTP world and the service call — pull inputs out of `req`, call the service, put the result on `res`:

```ts
export async function getUser(req, res) {
  const user = await readUser(req.params.id);
  res.json(user);
}
```

That's it. No business rules, no validation logic, nothing that would need to change if you swapped Express for Fastify or added a non-HTTP entry point.

### Error handling

One thing this split forces you to solve properly: since services can't reach for `res.status()`, they need another way to signal failure. Throwing typed errors works well:

```ts
export class NotFoundError extends Error {
  constructor(resource: string, id: string | number) {
    super(`${resource} ${id} not found`);
    this.name = new.target.name;
  }
}

export class ValidationError extends Error {
  constructor(message: string) {
    super(message);
    this.name = new.target.name;
  }
}
```

The controller doesn't catch these — it just lets them propagate. In Express 5, a rejected promise from an async handler is automatically forwarded to the error-handling middleware, so no extra wiring is needed. A single middleware, registered last, maps error types to HTTP responses:

```ts
app.use((err, req, res, next) => {
  if (err instanceof ValidationError) {
    return res.status(400).json({ error: err.message });
  }

  if (err instanceof NotFoundError) {
    return res.status(404).json({ error: err.message });
  }

  console.error(err);
  res.status(500).json({ error: "Internal server error" });
});
```

This keeps the framework coupling contained to exactly two places: the controller (adapting inputs/outputs) and this one middleware (adapting errors to status codes). Everything in between — the actual logic — stays portable.

## Why it's worth the extra layer

It's one more file, one more indirection, for what a single-function route handler could do inline. But the payoff shows up as the app grows: services are testable without spinning up Express, reusable outside HTTP entry points, and the business logic doesn't erode every time the web framework's API changes.
