# Fail. Learn. Repeat.

Source for [blog.faillearnrepeat.net](https://blog.faillearnrepeat.net/) — a
[Hugo](https://gohugo.io/) blog using the
[hugo-bearblog](https://github.com/janraasch/hugo-bearblog) theme.

## Develop

The theme is a git submodule, so clone with submodules (or init them after cloning):

```bash
git clone --recurse-submodules <repo-url>
# or, in an existing clone:
git submodule update --init
```

Run a local preview (Google Analytics is disabled in the development environment):

```bash
hugo server
```

Production build (what Netlify runs):

```bash
hugo --gc --minify
```

## Structure

- `hugo.yaml` — site config (theme, menu, Google Analytics, params).
- `content/_index.md` — home page intro; the post list is generated below it.
- `content/blog/<slug>/index.md` — one page bundle per post (`title`, `date`,
  `description`); post URLs are `/blog/<slug>/`.
- `layouts/` — overrides: `index.html` (home = intro + posts grouped by year),
  `partials/custom_head.html` (Analytics, production only),
  `partials/footer.html` (CC BY-NC-SA + RSS).
- `netlify.toml` — Hugo-only build.

## Writing a post

Create `content/blog/my-post/index.md`:

```yaml
---
title: My post title
date: 2026-07-07
description: One-line summary used for the meta description.
---
```

Put images alongside it in the same folder and reference them by filename.

## License

Content is licensed under
[CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/).
