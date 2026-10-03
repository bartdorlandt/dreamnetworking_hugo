---
name: hugo-content
description: Add or edit content on the dreamnetworking.nl Hugo site — blog posts, presentations/talks, podcast appearances. Use when asked to write a new blog post, add a talk or podcast to the speaker page, or fix front matter on existing content.
---

# Hugo content for dreamnetworking.nl

## Where things go

| Type | Command | Lands in |
|---|---|---|
| Blog post | `hugo new content blogs/YYYYMMDD_slug/index.md` | `content/blogs/` |
| Presentation | `hugo new content speaker/YYYYMMDD_slug/index.md` | `content/speaker/` |
| Podcast | `hugo new content --kind podcast speaker/YYYYMMDD_slug/index.md` | `content/speaker/` |

Always scaffold with `hugo new` so front matter comes from `archetypes/`. Every item is a page bundle: one folder named `YYYYMMDD_slug` (date of publication / talk / episode, slug in kebab-case), with `index.md` and an `images/` subfolder.

## Front matter rules

- YAML (`---`), never TOML. No `draft:` field.
- `date:` plain `YYYY-MM-DD`, matching the folder prefix.
- `title:` the archetype guesses from the slug; always rewrite it to the real title.
- `description:` one or two sentences, ~200 chars max. Used for og:description and list cards, so never leave it empty.
- `image:` `/<section>/<folder>/images/<file>` (e.g. `/blogs/20261003_why-not-yang/images/top.png`). Shown as the featured image. Remove the line if there is no image.
- `images:` (plural, list) — blog posts only, and only when the image is **at least 1200px wide**. Drives og:image for LinkedIn previews (`layouts/partials/head.html`). Same path as `image:`. Check width with `sips -g pixelWidth <file>`.
- `tags:` lowercase for blogs; speaker entries use the existing mixed style.

## Speaker entries

- `speakerType:` `presentation` or `podcast`.
- `location:` event or show name; `locationUrl:` its site (or the episode page).
- `links:` order: Video, Slides for talks; Episode page, Blog post, YouTube, Apple Podcasts, Spotify, others for podcasts. Drop entries without a URL.
- Body: short summary, then details (hosts, chapters, etc.) as needed.

## Writing

- Keep the author's voice and text when given a draft; strip any assistant chatter around it.
- Link related content with site paths, e.g. `/speaker/20260508_json-freedom-or-chaos-how-to-trust-your-data/`.

## Verify

```sh
hugo --quiet && ls public/<section>/<folder>/
```

Must build without errors and produce `index.html`. Do not commit unless asked.
