# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

Documentation site for [Atomize](https://github.com/Anatoly1010/Atomize), a modular Python software for controlling scientific/industrial instruments (primarily EPR spectrometers). This repo is **only the docs** — it does not contain Atomize itself. Published to GitHub Pages at https://anatoly1010.github.io/atomize_docs/.

Built with Jekyll + the [Just the Docs] theme (pinned to `just-the-docs 0.10.1`, Jekyll `~> 4.4.1`, Ruby 3.3).

## Common commands

```bash
bundle install                  # install gems (first time / after Gemfile change)
bundle exec jekyll serve        # preview locally at http://localhost:4000
bundle exec jekyll build        # build into ./_site (same command CI runs)
```

CI (`.github/workflows/ci.yml`) runs `bundle exec jekyll build` on every push/PR; `pages.yml` builds and deploys to GitHub Pages on push to `main`. There are no tests or linters.

## Site structure

- `index.md` — home page (top-level intro to Atomize)
- `pages/` — all documentation content, organized via Just-the-Docs `nav_order` + `parent`/`has_children` front matter (not via folder layout). Subdirectories:
  - `pages/functions/` — one Markdown file per instrument category (AWG, digitizer, lock-in, magnet, oscilloscope, etc.), plus `general_functions/` (concurrency, data management) and `plotting_functions/` (liveplot usage)
  - `pages/projects/` — per-spectrometer pages (xband, qband, endstation)
  - `pages/script_examples/` — example experiment scripts (e.g. cw_epr)
- `_config.yml` — site config; theme is `just-the-docs`, color scheme `skhep` (custom, defined in `_sass/color_schemes/skhep.scss`; dark variant `darkskhep.scss` also present)
- `_layouts/page.html`, `_includes/` (`head.html`, `head_custom.html`, `toc.html`, `package_table.html`) — local overrides of theme defaults
- `_sass/custom.scss` — site-specific style overrides
- `assets/`, `images/` — figures referenced from pages (paths are absolute, prefixed with `/atomize_docs/`)
- `_site/`, `.jekyll-cache/`, `vendor/` — build output / caches; do not edit

## Authoring conventions

- Every page needs Jekyll front matter. Existing pages use: `title`, `layout: page`, and either `nav_order` (top-level) or `parent:` + `nav_order` (child). Parent sections use `has_children: true`.
- Internal links use absolute paths prefixed with the site baseurl, e.g. `/atomize_docs/pages/instruments` and `/atomize_docs/images/figure_1.png`. Match this style when adding cross-references so links work both on GitHub Pages and in local preview.
- Custom callouts are defined in `_config.yml`: `{: .warning }`, `{: .note }`, `{: .important }`, `{: .highlight }`.
- When adding a new instrument page under `pages/functions/`, mirror the front matter of a sibling (e.g. `digitizer.md`) and add the entry to `pages/instruments.md`'s table-of-contents list.

## Notes for editing

- SASS deprecation warnings are silenced in `_config.yml` (`sass.silence_deprecations: ["import"]`) — leave that alone unless upgrading the theme.
- `_config.yml` changes are **not** picked up by `jekyll serve` automatically; restart the server.
- The Gemfile pins `just-the-docs 0.10.1`; bumping it can shift layout/CSS and may require updating the local overrides in `_layouts/` and `_includes/`.

[Just the Docs]: https://just-the-docs.github.io/just-the-docs/
