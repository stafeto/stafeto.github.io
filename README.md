# stafeto.github.io

The site and blog of [stafeto](https://github.com/stafeto/stafeto), served at https://stafeto.github.io.

The package index of stafeto lives under https://stafeto.github.io/packages/ and comes from another repository. This site never creates a `packages/` path.

## Layout

| Path | Contents |
|---|---|
| `config.toml` | Zola configuration |
| `content/` | the home page, the `try` page and the blog posts (Markdown) |
| `templates/` | the page templates of the site's own theme, `robots.txt` and the 404 page |
| `static/` | the stylesheet and the favicon |
| `.github/workflows/pages.yml` | build and deploy to GitHub Pages |

The theme has no JavaScript, no analytics, no external fonts and no CDN. It follows the colour scheme of the reader (`prefers-color-scheme`).

## Build

The site is built with [Zola](https://www.getzola.org/) 0.23.6, a single static binary. The workflow downloads that exact release and checks its SHA-256 before it runs.

```sh
zola build            # writes the site to public/
zola check            # checks the internal and external links
zola serve            # local preview at http://127.0.0.1:1111
zola serve --drafts   # the preview with the draft posts
```

## Writing a post

Add a Markdown file to `content/blog/` with this front matter:

```toml
+++
title = "Title of the post"
description = "One sentence for the list and the feed."
date = 2026-01-01
+++
```

A post with `draft = true` is left out of the build, the list, the feed and the sitemap. Remove the line to publish it.

## Deploy

A push to `main` runs the workflow, which builds the site and deploys it with `actions/upload-pages-artifact` and `actions/deploy-pages`. In the repository settings, Pages must use the source "GitHub Actions". The workflow pins each action by its full commit SHA.

## Licence

The text is under CC BY 4.0 and the code under the MIT licence. See [LICENSE](LICENSE).
