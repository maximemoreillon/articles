# Image migration from img.maximemoreillon.com

As of 2026-08-30, the 200 images this blog used from `https://img.maximemoreillon.com/images/*`
have been copied into their respective article's Hugo page bundle (`content/articles/<slug>/<id>.<ext>`),
and articles now reference them by relative path instead of the external URL.

The full list of migrated image IDs, their extensions, and which article(s) used each one is
tracked in the `img` repo itself: [`blog_usage.json`](https://github.com/maximemoreillon/img/blob/master/blog_usage.json).

These images are no longer required on `img.maximemoreillon.com` for this blog's sake, but they
have **not** been deleted from that repo and should not be, without checking `blog_usage.json`
against other known consumers first (other GitHub project READMEs, Medium cross-posts, etc.).
