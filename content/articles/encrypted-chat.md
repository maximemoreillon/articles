---
date: "2026-03-21"
title: "Building a Client-Side Encrypted Chat App with SvelteKit and Firebase"
tags: ["Svelte", "SvelteKit", "Firebase", "Cryptography", "Projects"]
---

[encrypted-chat](https://github.com/maximemoreillon/encrypted-chat) is a small chat application built not as a secure messaging product but as a way to get hands-on with the Web Crypto API: what it actually takes to keep message content unreadable to the backend itself, rather than merely encrypting data in transit and at rest. The app is a SvelteKit frontend backed by Firebase: Firebase Authentication (Google sign-in) handles who can access the app, and Firestore stores chats and messages, but the messages Firestore holds are already ciphertext by the time they leave the browser.

## Where encryption happens

All encryption and decryption is done client-side using the browser's native [Web Crypto API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Crypto_API), with AES-GCM as the cipher. Each chat has its own 256-bit symmetric key, generated with `crypto.subtle.generateKey` and exported to a base64 string so it can be stored and pasted around. Encrypting a message generates a random 12-byte IV, encrypts the plaintext with `crypto.subtle.encrypt`, and writes the resulting ciphertext and IV (both base64-encoded) to the message document in Firestore. Decryption reverses the process using the same key and the IV stored alongside the ciphertext. Firestore never sees the key or the plaintext, only opaque ciphertext and an IV.

## Key handling

Since Firestore isn't a party to the encryption, the key has to reach every participant some other way. The app never transmits it: a per-chat key dialog lets a user either generate a fresh key or paste in one shared out of band, and the key is then kept in the browser's `localStorage`, indexed by chat ID. This also means the app has no way to recover a chat if the key is lost, and no server-side mechanism to reset it.

To catch the common mistake of a participant holding the wrong key, the chat view tries to decrypt the most recent message as soon as a key is entered. If that fails, the UI flags the key as incorrect instead of silently showing garbled text, and disables the message composer until a working key is provided.

## Encrypted/decrypted toggle

Each chat view has a lock/unlock switch that toggles between showing messages as their decrypted plaintext or as raw stored ciphertext, which makes it easy to see, message by message, that what's leaving the browser really is encrypted rather than trusting the implementation blindly.

## Stack and deployment

The frontend is built with SvelteKit and Svelte 5, styled with Tailwind CSS and a shadcn-svelte-style component set (`bits-ui`, `lucide-svelte` icons). It's built with `adapter-static` and deployed to Firebase Hosting, with GitHub Actions workflows publishing a preview on pull requests and deploying to production on merge to `master`.

## Limitations

This is a learning project, not a security product, and it was never meant to be one. Its value is in making the primitives concrete: generating a key, using it to encrypt and decrypt, and seeing what happens when Firestore only ever stores ciphertext. Real secure messaging systems (Signal being the reference example) also have to solve key exchange, forward secrecy, multi-device support, and metadata protection, none of which this project attempts. Key distribution here is entirely manual and there is no key rotation, both of which would be immediate problems for anything beyond a toy.
