# AGENTS.md

Repository instructions for Codex and other coding agents working in atomize_docs. Adapted from `CLAUDE.md`; keep shared project facts consistent when updating either guide.

## Working rules

- Make the smallest change that meets the user's request. Follow existing page conventions; avoid unrelated rewriting, dependency changes, or site redesigns.
- Keep progress and final reports concise. For multi-step tasks, show a short checklist using the plan/todo tool when available, otherwise a progress update.
- Read the relevant source code in the sibling Atomize repository when documenting API behavior. Preserve technical meaning when changing prose or formatting.
- When reviewing or merging documentation commits, inspect every changed page for writing style as well as build errors. A successful build does not establish that the prose follows the site's conventions.
- Preserve unrelated user changes. When committing, group the requested corrections into one cohesive commit for this repository.

## What this repository is

Documentation site for [Atomize](https://github.com/Anatoly1010/Atomize), a modular Python software for controlling scientific/industrial instruments (primarily EPR spectrometers). This repo is **only the docs** — it does not contain Atomize itself. Published to GitHub Pages at https://anatoly1010.github.io/atomize_docs/.

Built with **MkDocs + the [Material for MkDocs] theme** (`mkdocs-material==9.7.6`, see `requirements.txt`; CI uses Python 3.12).

> **Legacy Jekyll tree:** an older site (Just-the-Docs theme) still lives in this repo under `pages/`, `_config.yml`, `_layouts/`, `_includes/`, `_sass/`. It is **not deployed** — the live site is built from `docs/` by MkDocs. Don't edit the Jekyll files expecting changes on the published site; they are kept only for reference. Leave their removal to a task that explicitly calls for it.

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
- Internal links are relative to the current source page within `docs/`, e.g. `protocol_settings.md` from a root page or `pulse_programmer.md#pulser_close` from another instrument page. Do **not** use the old Jekyll absolute `/atomize_docs/...` paths.
- Callouts use MkDocs admonitions: `!!! note`, `!!! warning`, `!!! important` (`admonition` + `pymdownx.details`), **not** Jekyll's `{: .note }`.
- Custom heading anchors / TOC labels use `attr_list`, e.g. `### tc_name() { #tc_name data-toc-label="tc_name" }`. Always set `data-toc-label` to the bare function name — without it the sidebar carries the whole argument list.
- **The instrument pages under `docs/functions/` are the style reference for every page, including the `math_modules/` ones.** Prose is **not hard-wrapped**: one paragraph, list item or admonition body per source line, however long. A function section opens with a `python` block giving the call forms with inline `# -> type` comments, then flowing "This function …" prose; bullet lists are for calls with many parameters (e.g. `math_modules/deer.md`), not for ordinary description. Prefer untitled `!!! note` / `!!! warning`.
- Function descriptions begin with "This function …" and state arguments, results, and behavior before device-specific details. Keep measured examples and device differences in separate paragraphs; avoid fragmentary openings or implementation-log language. Use `# -> type` for returned values and short action comments for commands.
- Use untitled `!!! note` boxes for availability and device-specific qualifications, and `!!! warning` boxes for existing failure or recovery requirements. Keep the device scope explicit. Do not invent stronger recovery requirements while restyling a page.
- In usage instructions, format visible controls consistently in **bold**, keyboard shortcuts as code (for example, `Ctrl + End`), and API names, values, and exceptions as code. Link API references to their function anchors.
- Reflowing an existing page can be verified mechanically: render it before and after with `python -m markdown` using the `mkdocs.yml` extension set, collapse whitespace, and diff — joining wrapped lines must leave the HTML identical.
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

## Validation and delivery

- For changes to site content, configuration, or theme, run `mkdocs build --strict`. To keep the output separate from an existing preview, use `mkdocs build --strict --site-dir /tmp/atomize-docs-check` on Linux.
- Inspect the generated section when changing callouts, heading anchors, code examples, or other rendering-sensitive markup. Check relative links and navigation as part of the build review.
- For instructions-only changes outside `docs/`, review the Markdown and run a whitespace check; a site rebuild is unnecessary.
- Run `git diff --check` and review the complete task diff before committing. Do not commit `site/` or temporary validation artifacts.
- Report what changed and which checks passed. A push to `main` triggers deployment; do not claim the live site has updated without confirming deployment.
