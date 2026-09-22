---
title: "A generic environment variable substitution script for Vite-based Vue.js containers"
date: "2022-09-12"
lastmod: "2022-09-12"
tags: ["Vue.js", "Docker", "DevOps", "Tutorials"]
---

In [a previous article](/articles/vue-runtime-env-vite-migration/), I adapted a method to set environment variables at runtime for a containerized Vue.js application to Vue 3 and Vite: build the app with placeholders such as `VITE_MY_VARIABLE_PLACEHOLDER`, then have a script replace them with real values when the container starts, using `sed`.

That script had two problems I have since run into in practice. This article addresses both.

## Every variable has to be added by hand

The original script substitutes one variable, and adding another means adding another line:

```sh
sed -i 's|VITE_MY_VARIABLE_PLACEHOLDER|'${VITE_MY_VARIABLE}'|g' $file
sed -i 's|VITE_ANOTHER_VARIABLE_PLACEHOLDER|'${VITE_ANOTHER_VARIABLE}'|g' $file
# and so on
```

This is easy to forget, and the script ends up listing every variable the application uses in two places: once in `.env.production`, and once here.

Since every variable follows the same `<NAME>_PLACEHOLDER` convention, the substitutions can be generated from the environment instead of being written out:

```sh
env | grep '^VITE_' | while IFS='=' read -r key value; do
  echo "s|${key}_PLACEHOLDER|${value}|g"
done > /tmp/env.sed

sed -i -f /tmp/env.sed $file
```

Every environment variable whose name starts with `VITE_` becomes one line of a `sed` script, which is then applied in a single pass. Adding a variable no longer touches this script at all: it only needs to exist in `.env.production` as `VITE_NEW_VARIABLE=VITE_NEW_VARIABLE_PLACEHOLDER`.

## Some characters break the substitution

The original command has a second issue, which only shows up with certain values:

```sh
sed -i 's|VITE_MY_VARIABLE_PLACEHOLDER|'${VITE_MY_VARIABLE}'|g' $file
```

`sed` gives special meaning to a few characters inside the replacement part of a substitution. `&` is replaced with the whole matched text, and the delimiter itself, here `|`, ends the substitution early if it appears in the value. A URL such as `https://api.example.com/x?a=1&b=2` is corrupted, because `&` is expanded rather than inserted literally:

```
$ VITE_URL='https://api.example.com/x?a=1&b=2'
$ sed -i 's|PLACEHOLDER|'${VITE_URL}'|g' file
$ cat file
https://api.example.com/x?a=1VITE_MY_VARIABLE_PLACEHOLDERb=2
```

The fix is to escape `&`, the delimiter, and the escape character itself before the value is inserted:

```sh
env | grep '^VITE_' | while IFS='=' read -r key value; do
  escaped=$(printf '%s' "$value" | sed 's/[&|\\]/\\&/g')
  printf 's|%s_PLACEHOLDER|%s|g\n' "$key" "$escaped"
done > /tmp/env.sed

sed -i -f /tmp/env.sed $file
```

`printf` is used instead of `echo` to build both the substitution and the escaped value, since some shells interpret backslash sequences in `echo`'s argument, which would undo the escaping.

## The full script

Combining this with the original's structure gives a script that never needs to be edited as the application's variables change:

```sh
#!/bin/sh

ROOT_DIR=/app

# Build a sed script with one substitution per VITE_* variable
env | grep '^VITE_' | while IFS='=' read -r key value; do
  escaped=$(printf '%s' "$value" | sed 's/[&|\\]/\\&/g')
  printf 's|%s_PLACEHOLDER|%s|g\n' "$key" "$escaped"
done > /tmp/env.sed

# Replace env vars in files served by NGINX
sed -i -f /tmp/env.sed $ROOT_DIR/assets/*.js* $ROOT_DIR/index.html
```

Unlike the original article's version, this one does not end with `exec "$@"`. [A follow-up article](/articles/vue-runtime-env-dockerfile/) covers how the script is wired into the Dockerfile, and why that line is no longer needed there.
