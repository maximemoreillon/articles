+++
title = "Countries tracker"
date = '2024-02-18'
lastmod = '2024-03-04'
tags = ['Firebase', 'Projects', 'Svelte']
+++

Nagoya Global Friends is an international community in Nagoya whose events gather people from all over the world. In order to visualize the various countries represented, I designed a simple Firebase application. The application allows users to register their home country, which gets stored in a Firestore database. The data is then displayed as a 3D globe with markers for each record.

![Article image](https://img.maximemoreillon.com/images/65d163cf12a3120013538c56)

The interface is built using Svelte and the [Threlte](https://threlte.xyz/) library for 3D rendering. Its source code is available on [GitHub](https://github.com/maximemoreillon/countries-tracker).