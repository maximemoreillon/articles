---
date: "2026-03-21"
title: "Shared Countdown: a Small Firebase + SvelteKit App"
tags: ["Svelte", "SvelteKit", "Firebase"]
---

[shared-countdown](https://github.com/maximemoreillon/shared-countdown) is a pet project built to poke at Firestore's real-time sync rather than to fill some gap in the countdown-timer market: a countdown, plus the ability to share it with other people instead of it living only in your own browser tab. It's a SvelteKit frontend on Firebase — Firebase Authentication for Google sign-in, Firestore for storage — and it reuses the same stack as [encrypted-chat](/encrypted-chat/), another small project built around SvelteKit and Firebase.

## What it does

A countdown is just a name and a target date and time, picked with a calendar and time input and stored in Firestore as a `Timestamp`. The countdown page recomputes the remaining years, months, days, hours, minutes, and seconds once a second using `dayjs`'s duration plugin, and renders them as a simple breakdown rather than a single "time remaining" string.

Sharing is handled with a `users` field on each countdown document: a list of email addresses. Adding someone appends their email to that array; removing them (aside from yourself) splices it back out. Firestore security rules — not part of the client code — are what actually enforce that only the users listed on a countdown can read or write it; the app itself only ever reads and updates that array.

## Stack

Same shape as the other Firebase-based projects in this series: SvelteKit with Svelte 5, Tailwind CSS, a shadcn-svelte-style component set (`bits-ui`, `lucide-svelte`), `sveltekit-superforms` and `zod` for form handling, built with `adapter-static`, and deployed to Firebase Hosting through GitHub Actions — a preview channel on pull requests, production on merge to `master`.

## Why it exists

Nothing about a countdown timer needs a backend, but sharing state between multiple people does, so this is mostly an excuse to play with Firestore listeners and real-time synced state, wrapped around whatever UI made that interesting to build. It's not aimed at competing with any of the many polished countdown apps out there, or even at being generally useful — it's a pet project that happens to work well enough to actually use for something like agreeing on an event date with a few other people.
