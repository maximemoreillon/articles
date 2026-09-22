---
title: "Showing the application version in a Vite-based Vue.js app"
date: "2026-09-22"
lastmod: "2026-09-22"
tags: ["Vue.js", "Docker", "Vite", "Tutorials"]
---

This article shows how to set the version of a Vite-based Vue.js app automatically at build time, using a Docker build argument fed by an environment variable provided by CI/CD, such as the Git tag being built.

## The Dockerfile

A build argument is declared with `ARG`, and made available to the build steps that follow it in the same stage:

```dockerfile
FROM node:24 AS build-stage
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .

ARG APP_VERSION=dev
ENV VITE_APP_VERSION=$APP_VERSION

RUN npm run build
```

`ARG` alone is not enough, because Vite only exposes environment variables prefixed with `VITE_` to client code, and a build argument is not automatically an environment variable to begin with. The `ENV` line does both jobs at once: it turns the build argument into an environment variable, under the name Vite expects, before `npm run build` runs.

The default, `dev`, is used whenever the image is built without specifying a version, for instance during local development.

## Passing the version at build time

To set the version explicitly, `--build-arg` is passed to `docker build`, with its value taken from a variable the CI/CD platform already provides. On GitLab CI/CD, `$CI_COMMIT_TAG` holds the tag being built:

```bash
docker build --build-arg APP_VERSION=$CI_COMMIT_TAG .
```

This is only set when the pipeline is triggered by a tag, so it fits a job that runs on tagged releases. Any other variable works equally well, such as a commit SHA for untagged builds.

## Reading it in the app

With the environment variable set at build time, the version is read like any other Vite environment variable, `import.meta.env.VITE_APP_VERSION`. Because it is undefined when the app is run directly with `npm run dev`, outside of the Docker build, a fallback is worth keeping:

```ts
export const version: string = import.meta.env.VITE_APP_VERSION || "dev"
```

This can then be displayed anywhere in the app, for instance in an about page:

```vue
<template>
  <v-list-item title="Version" :subtitle="version" />
</template>

<script setup lang="ts">
import { version } from "./version"
</script>
```

## Verifying the result

Building the image with an explicit version and inspecting the built files confirms that the value has been substituted, and not left as a reference to be resolved later:

```bash
docker build --build-arg APP_VERSION=v1.2.3-abc1234 -t my-app .
docker run --rm my-app grep -l v1.2.3-abc1234 /usr/share/nginx/html/assets/*.js
```

This prints the name of the bundled file containing the version, the same way any other build-time constant would be inlined by Vite.
