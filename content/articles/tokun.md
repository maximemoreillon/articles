---
date: "2025-04-02"
title: "Tokun: Tracking Japanese Vocabulary Across Texts"
tags: ["Svelte", "SvelteKit", "Japanese", "Drizzle ORM", "Projects"]
---

[tokun](https://github.com/maximemoreillon/tokun) is a pet project for learning Japanese from real texts: paste some Japanese in, and the app splits it into words and lets you mark which ones you know. It is a SvelteKit application storing its data in PostgreSQL through Drizzle ORM, and relying on [kuromoji.js](https://github.com/takuyaa/kuromoji.js) to do the actual language processing.

## How it works

Japanese is written without spaces, so the first step is tokenization. When a text is submitted, kuromoji breaks it into tokens, each with its surface form (the word as written), part of speech and reading. The server stores the text, then registers every token. Tokens belong to a user and are deduplicated by surface form, so a word that appears in ten different texts is a single token linked to each text through a join table that also keeps its position. This is what makes the tracking useful: marking a word once applies to every text containing it.

Not every token is worth tracking. Only nouns, verbs, interjections and adverbs that have a reading are treated as vocabulary, and everything else, particles for instance, is rendered as plain text.

## Reading a text

A text is displayed with its unknown words in red, in bold if they were flagged as important. Toggles can additionally highlight known words in green and ignored words in grey. Clicking a word opens a dialog showing it with its reading above it as furigana, along with checkboxes to mark it as known, important or ignored. A details page for each word lists how many times it occurs and in which texts. There is also a token list page with search, known/important filters and pagination, which is a more direct way to go through what has been collected so far.

## Auth and deployment

Login goes through Auth.js with Keycloak or Auth0 as the identity provider, and the user's email is used as the user ID. Every query is scoped to it, so each user only ever sees their own texts and words. The app itself is deployed like the other small services of my homelab: a Node Dockerfile, a GitLab CI pipeline pushing the image to Docker Hub, and a manifest applied to the Kubernetes cluster, with settings coming from Vault through an `ExternalSecret`.

## Rough edges

A few things are simple by design, and some are marked as TODOs in the code. The database schema has no unique constraints on texts or tokens, although the application logic deduplicates tokens. Tokens are matched by surface form only, so two words written identically but read differently share a single entry. The kuromoji dictionary is also reloaded every time a text is submitted, which is slow but of little consequence at this scale.
