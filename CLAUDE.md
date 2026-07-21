# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

Documentation site for [Atomize](https://github.com/Anatoly1010/Atomize), a modular Python software for controlling scientific/industrial instruments (primarily EPR spectrometers). This repo is **only the docs** — it does not contain Atomize itself. Published to GitHub Pages at https://anatoly1010.github.io/atomize_docs/.

Built with **MkDocs + the [Material for MkDocs] theme** (`mkdocs-material==9.7.6`, see `requirements.txt`; CI uses Python 3.12).

> **Legacy Jekyll tree:** an older site (Just-the-Docs theme) still lives in this repo under `pages/`, `_config.yml`, `_layouts/`, `_includes/`, `_sass/`. It is **not deployed** — the live site is built from `docs/` by MkDocs. Don't edit the Jekyll files expecting changes on the published site; they are kept only for reference and can be removed.

## Common commands

```bash
pip install -r requirements.txt   # install MkDocs + the Material theme
mkdocs serve                      # preview locally at http://localhost:8000
mkdocs build                      # build into ./site
mkdocs build --strict             # build, failing on warnings (what CI runs)
```

CI (`.github/workflows/ci.yml`) runs `mkdocs build --strict` on every push to `main` and on PRs; `pages.yml` ("Deploy MkDocs site to Pages") builds with MkDocs and deploys `./site` to GitHub Pages on push to `main`. There are no tests or linters beyond the strict build.

## Site structure

- `mkdocs.yml` — site config, theme, markdown extensions, and the explicit `nav:` tree. **Navigation is defined here**, not via per-page front matter.
- `docs/` — all documentation content (`docs_dir: docs`):
  - `docs/index.md` — home page
  - `docs/functions/` — one Markdown file per **instrument category** (`awg`, `digitizer`, `lock_in`, `magnet`, `oscilloscope`, `temp_controller`, …), plus `general_functions/` (concurrency, data management) and `plotting_functions/` (liveplot usage)
  - `docs/projects/` — per-spectrometer pages (xband, qband, endstation)
  - `docs/script_examples/` — example experiment scripts (e.g. cw_epr)
  - `docs/images/` — figures; `docs/stylesheets/extra.css` — extra styles (referenced from `mkdocs.yml`)
- `site/` — build output; do not edit or commit.

## Authoring conventions

- Pages are plain Markdown — **no front matter required**; the title comes from the first `# H1` (and the `nav:` label in `mkdocs.yml`).
- To add a page: create the `.md` under `docs/`, add it to the `nav:` tree in `mkdocs.yml`, and — for a new instrument category — add it to the table in `docs/instruments.md`. Mirror the structure of a sibling page (e.g. `docs/functions/digitizer.md`).
- Internal links are **relative to the `docs/` tree**, e.g. `protocol_settings.md`, `functions/general_functions/general_functions.md`, `images/figure_2.png`. Do **not** use the old Jekyll absolute `/atomize_docs/...` paths.
- Callouts use MkDocs admonitions: `!!! note`, `!!! warning`, `!!! important` (`admonition` + `pymdownx.details`), **not** Jekyll's `{: .note }`.
- Custom heading anchors / TOC labels use `attr_list`, e.g. `### tc_name() { #tc_name data-toc-label="tc_name" }`.
- Enabled extensions (see `mkdocs.yml`): `admonition`, `footnotes`, `attr_list`, `md_in_html`, `pymdownx.details/superfences/highlight/inlinehilite/snippets/tabbed`, and `toc` (permalinks).

## Notes for editing

- `mkdocs serve` live-reloads on content changes, but **restart it after editing `mkdocs.yml`**.
- Keep the build warning-free — CI runs `--strict`, so a broken link or a page missing from `nav:` fails the build.
- The theme is pinned (`mkdocs-material==9.7.6`); bumping it can shift layout/CSS.
- Some migrated `docs/` pages still carry leftover Jekyll/Kramdown attributes such as `{: .enum }` (e.g. in `functions/temp_controller.md`); these do **not** render under MkDocs — replace them with `attr_list`/admonition equivalents when you touch those pages.

## EPR automation (`epr_auto`) pages

`docs/projects/epr_auto/` documents the Atomize_ITC protocol runner
(`atomize/epr_auto/`, which lives in the separate Atomize_ITC repo, not here).
Two maintenance rules apply when that code changes:

- **`steps.md` is auto-generated — never hand-edit it.** It is emitted from the
  runner's own step registry by a generator in the Atomize_ITC repo. Whenever a
  step or parameter registration changes in `steps.py` (a new step, a changed
  default/range/choice, edited help text), regenerate the page from the
  Atomize_ITC checkout:

  ```bash
  # run from the Atomize_ITC repo root
  python3 -m atomize.epr_auto.docgen /home/anatoly/atomize_docs/docs/projects/epr_auto/steps.md
  ```

  The page carries a do-not-edit banner and this command at its top.
  Regeneration is **manual** — it is not run at site-build time — so it must be
  redone by hand whenever the `steps.py` registrations change.

- **Protocol-schema changes must touch `protocols.md` in the same session.**
  The YAML dialect (top-level keys, per-step `retries`/`on_fail`/`checkpoint`,
  the `foreach` block, value syntax, autonomy levels) is documented by hand in
  `projects/epr_auto/protocols.md`; a schema change in the runner that the
  generated `steps.md` does not capture must be reflected there in the same
  change. Prose about preset expectations, judges, and failure messages lives
  in `presets.md` / `tuning.md` / `troubleshooting.md` — keep those in step with
  the code (verbatim error strings especially) when it changes.

[Material for MkDocs]: https://squidfunk.github.io/mkdocs-material/
