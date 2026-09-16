---
date: "2026-09-15"
title: "Nuxt with Vuetify's v-data-table-server"
tags: ["Nuxt", "Vue", "Vuetify"]
---

This article showcases the usage Vuetify's `v-data-table-server` in a Nuxt application.

## Back-end

The example used throughout this example involves fetching movies from a PostgreSQL database using the [Drizzle ORM](https://orm.drizzle.team/) while validation is implemented using [Zod](https://zod.dev/).
Data is queried in paginated fashion where the page number and size can be adjusted via URL query parameters.
Here is `server/api/movies/index.get.ts`

```ts
import { db } from "~~/server/db";
import { Movies } from "~~/server/db/schema";
import { z } from "zod";
import { asc, count, desc, like } from "drizzle-orm";

const querySchema = z.object({
  itemsPerPage: z.coerce.number().default(10),
  page: z.coerce.number().gt(0).default(1),
  sort: z.union([z.literal("title"), z.literal("id")]).default("id"),
  order: z.union([z.literal("asc"), z.literal("desc")]).default("asc"),
});

export default defineEventHandler(async (event) => {
  const { itemsPerPage, page, sort, order } = await getValidatedQuery(
    event,
    querySchema.parse,
  );

  const offset = (page - 1) * itemsPerPage;
  const orderMap = { asc, desc };

  const items = await db
    .select()
    .from(Movies)
    .orderBy(orderMap[order](Movies[sort]))
    .limit(itemsPerPage)
    .offset(offset);

  const [{ total }] = await db.select({ total: count() }).from(Movies);

  return { items, total, itemsPerPage, page, sort, order };
});
```

## Front-end

The front-end consumes the aforementioned API and displays the data in a `v-data-table-server`.

```html
<template>
  <h2>Movies</h2>
  <v-data-table-server
    :loading="pending"
    :headers="headers"
    :items="data.items"
    :items-length="data.total"
    v-model:itemsPerPage="itemsPerPage"
    v-model:page="page"
  />
</template>

<script setup lang="ts">
  import { useRouteQuery } from "@vueuse/router";

  const route = useRoute();
  const query = computed(() => route.query);

  const page = useRouteQuery("page", 1, { transform: Number });
  const itemsPerPage = useRouteQuery("itemsPerPage", 10, { transform: Number });

  const { data, pending } = await useFetch("/api/movies", {
    query,
    key: JSON.stringify(query.value),
  });

  const headers = [
    { title: "ID", key: "id" },
    { title: "Title", key: "title" },
  ];
</script>
```

Having the page number or page size reset every time the user navigates away would be inconvenient, so those values are synchronized into the URL query parameters using [VueUse](https://vueuse.org/)'s `useRouteQuery`.
Since `query` is a `computed`, any changes of URL query parameter triggers a re-execution of `useFetch`.

`key: JSON.stringify(query.value)` solves `<no response> Request aborted as another request to the same endpoint was initiated` errors.
