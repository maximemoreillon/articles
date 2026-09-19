---
date: "2026-05-25"
title: "MQTT Alarm Clock: a Cron Scheduler with a Web UI"
tags: ["Svelte", "SvelteKit", "MQTT", "IoT", "Drizzle ORM", "Kubernetes", "Projects"]
---

[mqtt-alarm-clock](https://github.com/maximemoreillon/mqtt-alarm-clock) is a small home project born from a practical worry: I often sleep with earplugs and fear I won't hear a regular alarm clock. My bedroom lights, on the other hand, are controlled over MQTT, so instead of making a sound, an alarm in this app publishes a message to an MQTT broker and the lights do the waking up. The app itself stays down to a scheduler with a settings page, and what happens on a given message is up to whatever is subscribed to that topic.

## How it works

An alarm is a name, a time, a set of weekdays, and an enabled flag. The UI is a time input and a row of weekday toggles, and it turns that selection into a cron expression of the form `minute hour * * weekdays` (for example `30 7 * * 1,2,3,4,5`), which is what gets stored. Alarms live in a PostgreSQL table managed with Drizzle ORM, with just `id`, `name`, `cron` and `enabled` columns. When an alarm is displayed or edited, the time and weekdays are parsed back out of the stored cron string.

The scheduling itself is done in the SvelteKit server process with `node-cron`. On startup, the server hook connects to the MQTT broker and registers a cron job for every enabled alarm. Every create, update or delete then calls `recreateCronJobs()`, which stops all registered jobs and registers them again from the database. It's a blunt approach, but with a handful of alarms it is simple and always leaves the in-memory schedule matching what is in the database. It also means the scheduler state lives in a single process, so running more than one replica would publish every alarm more than once, and the manifest sets `replicas: 1`.

When a job fires, it publishes a message to the broker. The broker URI, credentials, topic and payload all come from environment variables (`MQTT_BROKER_URI`, `MQTT_USERNAME`, `MQTT_PASSWORD`, `MQTT_TOPIC`, `MQTT_PAYLOAD`), defaulting to a public test broker, the topic `test` and the payload `ON`. Two things follow from that: every alarm publishes the same topic and payload, so an alarm carries no information beyond when it fires, and the cron jobs are scheduled in a hardcoded `Asia/Tokyo` timezone.

## Authentication

Login goes through Auth.js (`@auth/sveltekit`), with Keycloak and Auth0 available as providers depending on which issuer variables are set. The JSON API under `/api/alarms` is used by the UI and checks for a session on every request, returning a 401 otherwise.

## Deployment

The app is built with `adapter-node` and packaged in a small Node 22 Dockerfile. A GitLab CI pipeline builds the image, pushes it to Docker Hub, and applies a Kubernetes manifest to a home cluster. The manifest contains the Deployment, a ClusterIP Service and a TLS Ingress, along with an `ExternalSecret` that pulls the MQTT credentials and the Keycloak client settings from Vault into the container's environment. The database connection is read from a `DATABASE_URL` environment variable.

## Limitations

Since it is a personal tool, several things are deliberately left simple: there is no per-alarm topic or payload, the timezone is not configurable, the UI only supports a time plus weekdays rather than arbitrary cron expressions, and there is no handling of missed alarms if the pod happens to be down at the scheduled minute. The last point follows directly from using in-process cron jobs rather than persisting any record of executions.
