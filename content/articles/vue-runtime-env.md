---
date: "2026-09-21T00:00:00+09:00"
title: "Runtime environment variables in a Vue.js app"
tags: ["Vue.js", "Docker", "Vite", "Tutorials"]
---

In a Vue.js app built with Vite, environment variables such as `VITE_API_URL` are read from `import.meta.env`. This is convenient, but Vite replaces the `VITE_` environment variables with their values during the build. This means they cannot be changed at runtime.

This article presents a workaround: when the container starts, a small JavaScript file is generated to copy the container's environment variables onto `window`. The app then reads its configuration from there.

## Why not just replace a placeholder?

A common approach, which I described in [an earlier article](https://moreillon.medium.com/environment-variables-for-containerized-vue-js-applications-f0aa943cb962), is to build the app with a placeholder such as `VITE_API_URL_PLACEHOLDER` and to run `sed` over the built files when the container starts. It works, but it rewrites minified, content-hashed bundles, and it relies on the placeholder surviving the build untouched. A separate configuration file leaves the bundle as it is.

## Setup

### 1. Keep dev values in the usual place

`.env.development` and `import.meta.env` continue to work for `npm run dev`:

```
VITE_API_URL=http://127.0.0.1:8080
```

### 2. Load a runtime config file

In `public/env.js`, an empty object serves as a placeholder during development:

```js
window.__ENV__ = {};
```

Vite serves it as is in development and copies it into `dist` at build time. It is loaded from `index.html`:

```diff
+<script src="/env.js"></script>
 <script type="module" src="/src/main.ts"></script>
```

If the project uses TypeScript, the compiler needs to be told about the object that will hold the runtime values. Without it, `window.__ENV__` fails with `Property '__ENV__' does not exist on type 'Window'`, even though the property will exist at runtime. This is done by adding a `Window` declaration to the project's declaration file, such as `env.d.ts` or `shims-vue.d.ts`:

```diff
+interface Window {
+  __ENV__?: Record<string, string>;
+}
```

### 3. Read the values in a single place

`src/runtimeEnv.ts` merges the build-time and runtime values, the runtime ones taking precedence:

```ts
export const env = {
  ...import.meta.env,
  ...window.__ENV__,
};
```

The rest of the app then uses `env.VITE_API_URL` instead of `import.meta.env.VITE_API_URL`:

```diff
 import axios from "axios";
+import { env } from "./runtimeEnv";

-axios.defaults.baseURL = import.meta.env.VITE_API_URL;
+axios.defaults.baseURL = env.VITE_API_URL;
```

In development, `window.__ENV__` is empty, so the value comes from `.env.development`. In production, it comes from the container.

### 4. Generate the file when the container starts

The official nginx image runs every executable script found in `/docker-entrypoint.d/` before starting nginx. This one, saved as `40-env-config.sh` at the root of the project, writes `env.js` from all the `VITE_*` environment variables:

```sh
#!/bin/sh
# 40-env-config.sh
{
  echo "window.__ENV__ = {"
  env | grep '^VITE_' | while IFS='=' read -r key value; do
    echo "  \"$key\": \"$value\","
  done
  echo "};"
} > /usr/share/nginx/html/env.js
```

Because it exports every variable with the `VITE_` prefix, adding a new setting later doesn't require touching this script.

Values are not escaped: one containing a double quote or a line break would produce invalid JavaScript. This is fine for URLs, but arbitrary values would call for a proper JSON encoder in the script.

### 5. Dockerfile and nginx

```dockerfile
FROM node:24 AS build-stage
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx AS production-stage
COPY --from=build-stage /app/dist /usr/share/nginx/html
COPY default.conf /etc/nginx/conf.d/default.conf
COPY 40-env-config.sh /docker-entrypoint.d/40-env-config.sh
RUN chmod +x /docker-entrypoint.d/40-env-config.sh
```

There is no `ENTRYPOINT` in this Dockerfile: overriding it would prevent the base image's own entrypoint, and therefore the script above, from running.

The nginx configuration is in `default.conf` at the root of the project, and the Dockerfile copies it over the image's default site configuration. It starts from the [standalone server configuration of the Vue Router documentation](https://router.vuejs.org/guide/essentials/history-mode.html), for applications with a router in history mode, which is not the object of this article. It has been modified to deal with the caching of `env.js`:

```diff
 server {
   listen 80;
   server_name localhost;
   root /usr/share/nginx/html;
   index index.html;
   location / {
     try_files $uri $uri/ /index.html;
   }
+  location = /env.js {
+    add_header Cache-Control "no-cache";
+  }
 }
```

The `location = /env.js` block prevents browsers from caching the file, which could otherwise leave them with the previous values for a while after a variable is changed.

## Result

```bash
docker build -t my-app .
docker run --rm -p 8080:80 -e VITE_API_URL=https://api.example.com my-app
curl localhost:8080/env.js
```

```js
window.__ENV__ = {
  "VITE_API_URL": "https://api.example.com",
};
```

Starting a new container with a different `VITE_API_URL` is enough to change the API the app talks to, without rebuilding the image. The same works with the `environment` section of Docker Compose, or the `env` section of a Kubernetes container.
