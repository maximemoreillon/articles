---
title: "Wiring the generic substitution script into the Dockerfile"
date: "2022-09-19"
lastmod: "2022-09-19"
tags: ["Vue.js", "Docker", "DevOps", "Tutorials"]
---

In [a previous article](/articles/generic-vite-runtime-env-script/), I made the environment variable substitution script generic, so that it no longer needs to be edited when a new variable is added. This article covers how that script is wired into the Dockerfile.

## Dockerfile

The Dockerfile copies the script into `/docker-entrypoint.d/`, where nginx's own entrypoint picks it up and runs it before starting nginx:

```dockerfile
COPY 40-env-config.sh /docker-entrypoint.d/40-env-config.sh
RUN chmod +x /docker-entrypoint.d/40-env-config.sh
```

The original article's Dockerfile has a mismatch here: it copies the script to `/docker-entrypoint.d/substitute_environment_variables.sh`, but runs `chmod +x` on `/substitute_environment_variables.sh`, a different path. The copied script is therefore never made executable, and nginx's entrypoint silently skips it. Keeping the two paths identical, as above, avoids the issue.

## No more `exec "$@"`

Unlike the original article's version, the generic script does not end with `exec "$@"`. That line was needed because the original script replaced the image's `ENTRYPOINT` entirely, and had to hand off to the container's command itself. Saved as `40-env-config.sh` and placed under `/docker-entrypoint.d/` as above, it is instead run by the image's own entrypoint, which starts nginx once every script there has finished.

The rest of the setup, the rest of the Dockerfile and the placeholder values in `.env.production`, is unchanged from the original article.
