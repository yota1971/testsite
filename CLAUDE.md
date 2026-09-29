# CLAUDE.md

## What this is

A personal static site published via GitHub Pages (`git@github.com:yota1971/testsite.git`, no CNAME — served at the default `github.io` URL). `index.html` is a single landing page whose "Tools" section links out to a grid of standalone, dependency-free audio/DSP demo tools under `tools/`. No framework, no bundler, no package.json.

## Commands

No build step and no test suite. To preview locally, open `index.html` (or any file under `tools/`) directly in a browser, or serve the directory with any static file server. Deployment is `git push` to whatever branch GitHub Pages is configured to serve — there's no CI workflow in this repo (no `.github/workflows/`), so Pages builds straight from the pushed HTML/CSS.

## Architecture

- **`index.html` + `styles.css`** — the landing page. `styles.css` is layered: the base rules come first and later blocks at the end of the file (card-link colours, visibility tweaks) override them (e.g. `.card { padding: 0 }` overrides the earlier `.card` padding), so edit the last matching rule rather than the first. The Tools grid is a flat list of hardcoded `<div class="card"><a href="tools/X.html">Label</a></div>` entries. **Adding a tool means two edits, not one**: drop the file in `tools/` and add its card to `index.html` — a file placed only in `tools/` is reachable by direct URL but won't show up on the site.
- **`tools/*.html`** — each file is a fully self-contained page (own `<style>`/`<script>`, no shared JS/CSS between tools). Treat each as independent; there's no shared component or build step that would propagate a fix from one tool to another.
- **`tools/loudness.md`** — after-the-fact spec/reference doc for the loudness-normalize workflow (`loudness.html`). Not a deployed page, not read by any tool — reference only when working on that tool. (A `console.md` was once documented here but does not exist in the repo.)

## Relationship to `../教材/`

Most of these tools originated as copies from the versioned tool series under `../教材/<name>/` (see `../MEMORY.md` for that folder's naming convention). **They have since diverged**: fixes have been committed here directly (e.g. `console.html`, `absorp.html` — see `git log`) that were never ported back to `教材/`. Do not assume `教材/` and `tools/` hold the same code for a given tool, and do not assume a fix made in one is reflected in the other — check before treating either as the source of truth for a specific bug.

## External CDN scripts/styles

All tools share one origin (`yota1971.github.io`), so any third-party script loaded by one tool can read/write every tool's `localStorage` (including `ac-calc.html`'s saved inputs and 計算書). Therefore every external `<script>`/`<link>` must use an **exact version** (no `@latest`, no version-less `npm/pkg` or `gh/user/repo` URLs) plus `integrity="sha384-…"` and `crossorigin="anonymous"`. Compute the hash from the pinned URL: `curl -s URL | openssl dgst -sha384 -binary | openssl base64 -A`. When upgrading a library, change the URL and the hash together. (Pinned on 2026-09-29: chart.js 4.5.1 / 4.4.1 / 3.9.1, katex 0.18.9, katex-copytex 1.0.2, soundfont-player 0.12.0, vscode markdown CSS @1.139.1.)
