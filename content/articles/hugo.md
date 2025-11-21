+++
date = '2025-11-21'
title = "Migrating from my CMS to Hugo"
tags = []
+++

For several years, I have been managing articles such as this one using a custom CMS built around Neo4j.
Although functional, this approach required operational overhead such as server administration, authentication and maintenance.

To reduce this burden, I decided to export all articles as markdown files and use [Hugo](https://gohugo.io/), a popular static site generator, to render and serve those.

Adding articles now involves adding mardkown files to the Hugo project's `content` folder and pushing the code to GitHub for CI/CD to compile it into the newest version of the site to be served.

Although this means I can no longer edit or add articles directly from the CMS's web UI, the simplicity and security that it provides feels worth it.
