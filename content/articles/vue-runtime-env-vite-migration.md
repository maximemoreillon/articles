---
title: "Adapting the environment variable substitution approach to Vue 3 and Vite"
date: "2022-04-18"
lastmod: "2022-04-18"
tags: ["Vue.js", "Docker", "DevOps", "Tutorials"]
---

[The original article](/articles/vue-runtime-env-substitution/) on setting environment variables at runtime for a containerized Vue.js application was written for a Vue 2 project built with Vue CLI, which uses webpack. Since then, Vite has become the standard build tool for Vue 3 projects, and it changes two things the original approach depends on: how variables are named and read, and where the built files end up. This article covers what has to change.

## Reading the variable

Vue CLI exposed environment variables on `process.env`, polyfilled by webpack at build time. Vite does not polyfill `process.env` in the browser bundle; instead, it exposes variables on `import.meta.env`, which is native to the ES module the browser (or, at build time, Vite itself) evaluates. The example component from the original article becomes:

```vue
<template>
  <div>VITE_MY_VARIABLE: {{my_variable}}</div>
</template>

<script>
export default {
  name: 'App',
  data: () => ({
    my_variable: import.meta.env.VITE_MY_VARIABLE
  })
}
</script>
```

## The variable prefix

Vue CLI only exposed variables prefixed with `VUE_APP_`. Vite has the equivalent rule, but with its own prefix: as its documentation states, "Variables prefixed with `VITE_` will be exposed in client-side source code after Vite bundling", and variables without that prefix stay out of the bundle, "to prevent accidentally leaking env variables to the client". The prefix can be changed with the [`envPrefix`](https://vite.dev/config/shared-options.html#envprefix) configuration option, but this article keeps the default.

The `.env` and `.env.<mode>` file convention used for the development and production environments in the original article is unchanged, so `.env.development` and `.env.production` still work as before, just with the variable renamed:

```
VITE_MY_VARIABLE=dev
```

## Build output layout

Vue CLI, through webpack, split the build output into a `js` directory, a `css` directory, and files such as `precache-manifest.<hash>.js` when its PWA plugin was used. This is what the original substitution script's file glob targeted:

```sh
for file in $ROOT_DIR/js/*.js* $ROOT_DIR/index.html $ROOT_DIR/precache-manifest*.js;
```

Vite's build defaults are different. Its `outDir` still defaults to `dist`, but `assetsDir`, the directory built files are nested under, defaults to `assets` rather than being split by file type. A production build now looks like this:

```
dist/
├── assets/
│   ├── index-4ba79cbc.js
│   └── index-a5b8f6c1.css
└── index.html
```

There is no `precache-manifest*.js` unless a PWA plugin for Vite, such as `vite-plugin-pwa`, is used and configured to produce one; this article assumes it is not. The substitution script's glob has to be updated accordingly:

```sh
for file in $ROOT_DIR/assets/*.js* $ROOT_DIR/index.html;
```

## Putting it together

The rest of the original approach, generating the placeholder in `.env.production`, substituting it with `sed` when the container starts, and copying the script into `/docker-entrypoint.d/`, still applies unchanged, once the variable is renamed and the glob updated:

```sh
#!/bin/sh

ROOT_DIR=/app

# Replace env vars in files served by NGINX
for file in $ROOT_DIR/assets/*.js* $ROOT_DIR/index.html;
do
  sed -i 's|VITE_MY_VARIABLE_PLACEHOLDER|'${VITE_MY_VARIABLE}'|g' $file
  # Your other variables here...
done
```

This still substitutes one variable at a time, and is still vulnerable to the special characters discussed in a follow-up article on making this script generic.
