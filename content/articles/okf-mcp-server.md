---
date: "2026-07-09"
title: "OKF MCP server"
tags: ["AI", "OKF", "MCP", "Projects"]
---

OKF (Open Knowledge Format) is Google's spec for representing a knowledge base as a directory of linked Markdown files — YAML frontmatter for structured metadata (`type`, `title`, `tags`), a Markdown body for the actual content, and regular `[link](path.md)` references between files forming a graph. It's a plain-text, git-friendly way to store the kind of internal knowledge that used to live in a wiki or a pile of Confluence pages, and it maps cleanly onto how an LLM wants to consume information: bounded chunks with explicit relationships, not a search index.

### What the server does

Five tools, all read-only:

- `okf_list_concepts` — enumerate every concept ID in the bundle
- `okf_read_concept` — return one concept's frontmatter and body as text
- `okf_get_links` — follow a concept's outgoing links to its neighbors
- `okf_find_by_tag` / `okf_find_by_type` — filter the bundle by metadata

Underneath, a small loader (`okf.py`) walks the bundle directory, parses each file's frontmatter with PyYAML, and resolves the Markdown links in the body into concept IDs — skipping images, anchors, and external URLs — so the link graph is available as data, not just prose. It's genuinely simple: no embeddings, no vector store, no chunking heuristics. The graph structure the format already provides is the retrieval mechanism.

### The one design decision that mattered: the bundle isn't baked in

The obvious way to ship this is to `COPY` the bundle into the Docker image alongside the code. I deliberately didn't do that — the bundle is excluded via `.dockerignore`, and the server reads it from a path set by a `BUNDLE_PATH` environment variable at runtime instead.

That one choice is what makes the server bundle-agnostic rather than a single-purpose tool wired to one dataset. The image itself knows nothing about what knowledge it's serving; point it at a different `BUNDLE_PATH` and it's a different assistant. In the actual deployment at JTEKT, the bundle isn't baked in or manually copied at all — an `initContainer` clones it from a GitLab repo into a shared volume before the main container starts, so updating the knowledge base is a `git push`, and the container image never needs rebuilding for a content change. That same portability is why the same image also works unmodified at my home lab, pointed at a completely different bundle.

### Built on FastMCP

The server itself is a thin `FastMCP` app — five `@mcp.tool`-decorated functions over `streamable-http` transport, which is what let it plug into JTEKT's existing setup without extra glue. Compared to the K8s agent, there's no agent loop here at all: this project doesn't run its own LLM or make any decisions. It's pure protocol — tools in, MCP out — and the reasoning happens entirely on the client side, wherever the server is plugged in.

The source code is available on GitHub: [github.com/maximemoreillon/okf-mcp-server](https://github.com/maximemoreillon/okf-mcp-server)
