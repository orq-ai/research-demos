# Conventions for Adding a Demo

This document describes how to ingest a new research artifact into `research-demos`.

---

## 1. Manifest Schema

Each demo directory must contain a `manifest.json` file at `demos/<slug>/manifest.json`.

```json
{
  "schemaVersion": 1,
  "slug": "<url-safe-kebab-case string>",
  "title": "<display title>",
  "authors": ["<name>"],
  "published": "YYYY-MM-DD",
  "updated": "YYYY-MM-DD",
  "summary": "<one-sentence summary for gallery card and OG description>",
  "tags": ["<tag>"],
  "source": {
    "repo": "<org/repo>",
    "path": "<path inside source repo>",
    "commit": "<full SHA of source commit>",
    "ingested": "YYYY-MM-DD"
  },
  "artifacts": [
    {
      "kind": "<kind>",
      "file": "<filename relative to demo dir>",
      "icon": "<icon id>",
      "label": "<human label>"
    }
  ]
}
```

**`kind` closed enum:** `doc | slides | explainer | pdf | notebook | video`

**Required top-level fields:** `schemaVersion`, `slug`, `title`, `authors`, `published`, `summary`, `tags`, `source`, `artifacts`.

**`source` provenance block:** All four fields (`repo`, `path`, `commit`, `ingested`) are required. `commit` must be the full 40-character SHA. `ingested` is the date you ran the ingest, not the source publication date.

**`artifacts` array:** Each entry requires `kind`, `file`, `icon`, and `label`. `file` is relative to the demo directory. `icon` must be one of `doc`, `slides`, `spark`, `pdf` (maps to inline SVG in the cover page).

---

## 2. Pandoc Render

To render a Markdown source file to `doc.html`, run from the repo root:

```bash
pandoc /path/to/source.md \
  -f markdown+tex_math_dollars+pipe_tables+footnotes \
  -t html5 \
  --mathjax \
  --template assets/pandoc-template.html \
  --metadata title="<document title>" \
  -o demos/<slug>/doc.html
```

**Flags rationale:**

- `+tex_math_dollars` — enables `$...$` for inline math and `$$...$$` for display math, emitting `\(...\)` and `\[...\]` which MathJax handles.
- `+pipe_tables` — GitHub-flavored pipe tables.
- `+footnotes` — Pandoc-style footnotes.
- `--mathjax` — tells pandoc to emit MathJax-compatible delimiters instead of plain text.
- `--template assets/pandoc-template.html` — wraps the rendered body with our shared shell.css and MathJax CDN.

**MathJax CDN** (`https://cdn.jsdelivr.net/npm/mathjax@3/es5/tex-mml-chtml.js`) is the **one external dependency we allow** in this repo. Rendering TeX math offline requires either a full Node.js + npm install (katex-cli) or a LaTeX distribution — both are heavy for a static-first repo. MathJax via CDN adds ~300 KB on first load but keeps the build step to a single `pandoc` call with no extra toolchain.

Verify after rendering:

```bash
wc -c demos/<slug>/doc.html   # must be > 0
grep -c "table" demos/<slug>/doc.html   # check tables rendered
```

---

## 3. Provenance Comment Format

Every cover `index.html` must begin with an HTML comment block immediately after `<!doctype html>` (before `<html>`), providing source provenance:

```html
<!--
  provenance:
    source-repo:   <org/repo>
    source-path:   <path inside source repo>
    source-commit: <full 40-char SHA>
    ingested:      YYYY-MM-DD
-->
```

This allows anyone to trace the artifact back to its origin in the source repo without parsing `manifest.json`.

---

## 4. Ingest Checklist

Follow these seven steps in order when adding a new demo:

1. **Create the demo directory:**
   ```bash
   mkdir -p demos/<slug>/
   ```

2. **Copy artifact files verbatim** — do not modify them:
   ```bash
   cp /path/to/source/slides.html  demos/<slug>/slides.html
   cp /path/to/source/explainer.html  demos/<slug>/explainer.html
   cp /path/to/source/*.pdf  demos/<slug>/
   ```

3. **Render Markdown to `doc.html`** using the pandoc command in §2.

4. **Capture the source commit SHA:**
   ```bash
   git -C /path/to/source-repo rev-parse HEAD
   ```
   Paste the full 40-character SHA into `manifest.json` and the cover provenance comment.

5. **Write `manifest.json`** following the schema in §1. Use the slug as the directory name exactly.

6. **Write the cover `index.html`:**
   - Copy an existing cover (e.g. `demos/llm-as-a-judge-bias-variance-tradeoff/index.html`) as a starting point.
   - Update the provenance comment, `<title>`, OG meta tags, eyebrow, `<h1>`, `.summary`, and the artifact grid links/icons.
   - Verify all four artifact links open correctly.

7. **Add a card to root `index.html`** — newest demo first (at the top of the `.gallery` div). Copy an existing card block, update slug, date, authors, title, summary, and chips.

---

## 5. When to Add Tooling

Hand-write everything through demo #2. By demo #3 or beyond, the repetition in steps 5–7 justifies a scaffolding script:

- A `scripts/ingest.sh` (or Python) that reads `manifest.json` and generates the cover `index.html` and the gallery card fragment.
- Keep the pandoc render as an explicit manual step (not automated) so the author always reviews the rendered output.
- Do not add a build system, bundler, or npm dependency. The site must stay static-first with no build step beyond pandoc.
