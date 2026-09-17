---
date: "2026-09-18T08:28:42+09:00"
draft: true
title: "Synchronizing a local user table with an OIDC identity provider"
tags: ["Nuxt", "OIDC", "Authentication", "Drizzle ORM"]
---

Applications authenticating via OIDC might not need a local user table at all; the session can simply hold whatever claims the identity provider hands back for the user currently logged in. The trouble starts as soon as the application needs to display data about users other than the one currently logged in, for example a list of items with the name of their owner. Other tables can only reference a user by an identifier such as `sub`, and the OIDC flow only ever hands back claims for whoever is logging in at that moment, so there is no way to look up another user's name without keeping a local copy of it. This article shows a pattern for solving this in a [Nuxt](https://nuxt.com/) application using [`nuxt-auth-utils`](https://github.com/atinux/nuxt-auth-utils) and [Drizzle ORM](https://orm.drizzle.team/), where a minimal local copy of each user is upserted into a PostgreSQL table on every successful login.

## The users table

Here is the [Drizzle](https://orm.drizzle.team/) schema for the local `users` table. It only stores what the application actually needs: the issuer and subject claims from the OIDC token, which together uniquely identify the user, and a display name. The combination of `issuer` and `sub` is declared unique, since the same `sub` value could in theory be reused across different identity providers.

```ts
export const users = pgTable(
  "users",
  {
    id: serial().primaryKey(),
    issuer: text().notNull(),
    sub: text().notNull(),
    name: text(),
  },
  (table) => [unique().on(table.issuer, table.sub)],
);
```

Having a local, auto-incrementing `id` also means other tables can reference the user with a plain integer foreign key instead of a composite `(issuer, sub)` pair, for example:

```ts
export const items = pgTable("items", {
  id: serial().primaryKey(),
  owner_id: integer()
    .notNull()
    .references(() => users.id),
  // ...
});
```

## The OIDC callback route

`nuxt-auth-utils` exposes `defineOAuthOidcEventHandler`, which handles the OIDC redirect flow and hands back the provider's `user` claims and `tokens` once the login succeeds. The route reads the issuer out of the ID token itself (via [`jose`](https://github.com/panva/jose)'s `decodeJwt`), upserts a local user record, and stores that local record in the session instead of the raw OIDC claims.

```ts
import { decodeJwt } from "jose";

export default defineOAuthOidcEventHandler({
  config: {
    scope: ["openid", "profile"],
  },
  async onSuccess(event, { user, tokens }) {
    const issuer = tokens.id_token && decodeJwt(tokens.id_token).iss;
    if (!issuer) throw new Error("ID token is missing an issuer");

    const dbUser = await upsertUser(issuer, user);

    await setUserSession(event, {
      user: dbUser,
    });

    return sendRedirect(event, "/");
  },
  onError(event, error) {
    console.error("OIDC error:", error);
    return sendRedirect(event, "/");
  },
});
```

Storing `dbUser` in the session, rather than the OIDC `user` object, means the rest of the application only ever deals with the shape of the local `users` table (including its local `id`), regardless of which identity provider was used to authenticate.

## Upserting the user

The actual upsert lives in a small utility function. It relies on Drizzle's `onConflictDoUpdate` against the `(issuer, sub)` unique constraint to either insert a new user or refresh the existing one's `name`.

```ts
type User = { sub: string; name?: string | null };

export async function upsertUser(issuer: string, { sub, name }: User) {
  const [user] = await db
    .insert(schema.users)
    .values({ issuer, sub, name: name ?? null })
    .onConflictDoUpdate({
      target: [schema.users.issuer, schema.users.sub],
      set: { name: name },
    })
    .returning();

  if (!user) throw new Error("Failed to upsert user");

  return user;
}
```

Because this runs on every login, the local copy self-heals: if a user's display name changes at the identity provider, the change is picked up the next time they sign in, without any separate sync job.

## Why this works well

- The local `id` gives the rest of the schema a stable, simple foreign key to reference, instead of a composite `(issuer, sub)` pair or the OIDC `sub` alone.
- The unique constraint on `(issuer, sub)` makes the upsert idempotent and keeps the door open to supporting more than one identity provider without risking `sub` collisions.
