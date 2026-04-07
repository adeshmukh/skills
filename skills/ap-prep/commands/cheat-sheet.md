---
description: Generate a dense, print-ready cheat sheet for exam memorization (AP Physics C, AP Calculus AB, AP Chemistry, GRE, SAT, etc.). Outputs a beautifully formatted HTML (default), LaTeX, or Markdown file with formulas, key facts, and color-coded sections.
argument-hint: "<topic> [--format html|latex|markdown] [--color color|grayscale] [--paper letter|a4|legal|a5]"
allowed-tools: Write, WebSearch, WebFetch
---

# Cheat Sheet Generator

Generate a comprehensive, visually polished, print-ready cheat sheet for rapid exam memorization.

## Step 1 — Parse Arguments

Extract from the user's invocation:

| Parameter | Default | Accepted values |
|-----------|---------|-----------------|
| `topic` | (required) | Any exam or subject: "AP Physics C: Mechanics", "AP Calculus AB", "AP Chemistry", "GRE Quant", "SAT Math", "Organic Chemistry" |
| `--format` | `html` | `html`, `latex`, `markdown` |
| `--color` | `color` | `color`, `grayscale` |
| `--paper` | `letter` | `letter` (8.5×11 in), `a4` (210×297 mm), `legal` (8.5×14 in), `a5` (148×210 mm) |

If arguments are missing, use defaults without asking. Infer `--format` if the user writes something like "in LaTeX" or "as a markdown file."

## Step 2 — Research the Topic

If the topic is well-known (AP exams, SAT, GRE, common university courses), rely on your training knowledge.

For niche or unfamiliar topics, use `WebSearch` to find the official syllabus, formula sheet, or exam specification before generating content.

## Step 3 — Plan the Content

Organize the cheat sheet into these sections (adapt based on subject — omit irrelevant ones, add subject-specific ones):

1. **Core Formulas** — Every testable formula, grouped by sub-topic. This is the most important section.
2. **Constants & Units** — Physical constants, conversion factors, standard values.
3. **Definitions** — Precise, exam-ready definitions for key terms (brief, not prose).
4. **Theorems & Rules** — Named theorems, laws, identities, with conditions of applicability.
5. **Quick Reference Tables** — Trig identities, derivative/integral rules, periodic data, reaction types, etc.
6. **Common Traps** — Mistakes students frequently make, with the correct approach.
7. **Mnemonics & Tips** — Memory aids, sign conventions, dimensional analysis hints.

Aim for completeness — a student should be able to walk into the exam having only studied this sheet.

## Step 4 — Generate the Output

### HTML Format (default)

Generate a **single self-contained HTML file** — all CSS and JavaScript inlined, no external resources except KaTeX via CDN for math rendering.

Design requirements:
- **3-column grid layout** using CSS Grid, collapsing to 2 or 1 column when printed on small paper
- **KaTeX** for math rendering: include `<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/katex@0.16.9/dist/katex.min.css">` and the KaTeX auto-render script
- **Formula boxes**: bordered boxes with subtle background, `break-inside: avoid`
- **Section headers**: large, bold, with a colored left border (or grayscale equivalent)
- **Color scheme**: assign a distinct accent color per major section (6–8 colors max); in grayscale mode use shades of gray
- **Typography**: system font stack, 9–10pt body, 1.2–1.3 line-height, tight but not cramped
- **Minimal margins**: 0.4 in margins in print mode, full-bleed sections
- **Print CSS** via `@media print`: hide nothing, force background colors to print (`-webkit-print-color-adjust: exact; print-color-adjust: exact`), correct page size via `@page { size: <paper> portrait; margin: 0.4in; }`
- **Title block**: exam name, date generated, paper/format info — compact, at top

Color palette for `--color color`:
```
Section 1 (Formulas):    #1a73e8 (blue)
Section 2 (Constants):   #0f9d58 (green)
Section 3 (Definitions): #f4b400 (amber)
Section 4 (Theorems):    #db4437 (red)
Section 5 (Tables):      #ab47bc (purple)
Section 6 (Traps):       #ff7043 (deep orange)
Section 7 (Mnemonics):   #00acc1 (teal)
```

Color palette for `--color grayscale`:
Use black section headers with gray left borders and light gray box backgrounds. No color anywhere.

HTML structure skeleton:
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>[TOPIC] Cheat Sheet</title>
  <!-- KaTeX -->
  <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/katex@0.16.9/dist/katex.min.css">
  <script defer src="https://cdn.jsdelivr.net/npm/katex@0.16.9/dist/katex.min.js"></script>
  <script defer src="https://cdn.jsdelivr.net/npm/katex@0.16.9/dist/contrib/auto-render.min.js"
    onload="renderMathInElement(document.body, {delimiters: [{left:'$$',right:'$$',display:true},{left:'$',right:'$',display:false}]})"></script>
  <style>
    /* === Reset & Base === */
    *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }
    body { font-family: system-ui, -apple-system, sans-serif; font-size: 9pt; line-height: 1.3; color: #111; background: #fff; }

    /* === Layout === */
    .page { padding: 0.4in; }
    .title-block { margin-bottom: 0.2in; border-bottom: 2px solid #111; padding-bottom: 6px; }
    .title-block h1 { font-size: 14pt; font-weight: 800; letter-spacing: -0.5px; }
    .title-block .meta { font-size: 7.5pt; color: #555; margin-top: 2px; }
    .grid { display: grid; grid-template-columns: repeat(3, 1fr); gap: 0.15in; }

    /* === Sections === */
    .section { break-inside: avoid-column; margin-bottom: 0.1in; }
    .section-header { font-size: 9pt; font-weight: 700; text-transform: uppercase; letter-spacing: 0.5px;
      padding: 3px 6px; margin-bottom: 6px; border-left: 4px solid; }
    .section-content { padding-left: 4px; }

    /* === Formula Boxes === */
    .formula-box { border: 1px solid; border-radius: 3px; padding: 5px 7px; margin: 4px 0;
      break-inside: avoid; font-size: 9pt; background: #fafafa; }
    .formula-box .label { font-size: 7.5pt; font-weight: 600; display: block; margin-bottom: 2px; opacity: 0.75; }
    .formula-box .expr { display: block; }

    /* === Lists & Tables === */
    ul, ol { padding-left: 14px; margin: 3px 0; }
    li { margin-bottom: 1px; }
    table { width: 100%; border-collapse: collapse; font-size: 8pt; margin: 4px 0; }
    th { background: #f0f0f0; font-weight: 700; padding: 2px 4px; border: 1px solid #ccc; text-align: left; }
    td { padding: 2px 4px; border: 1px solid #ddd; }
    .trap { background: #fff8f8; border-left: 3px solid #db4437; padding: 3px 6px; margin: 3px 0;
      font-size: 8.5pt; break-inside: avoid; }
    .tip { background: #f0fff4; border-left: 3px solid #0f9d58; padding: 3px 6px; margin: 3px 0;
      font-size: 8.5pt; break-inside: avoid; }
    .definition { margin: 3px 0; }
    .definition .term { font-weight: 700; }
    .definition .def { color: #333; }
    strong { font-weight: 700; }
    code { font-family: 'Courier New', monospace; font-size: 8.5pt; background: #f5f5f5; padding: 0 2px; border-radius: 2px; }

    /* === Color-coded Section Accents (replace with gray variants for grayscale) === */
    /* Applied via .s1 through .s7 classes on .section-header and .formula-box */

    /* === Print === */
    @media print {
      -webkit-print-color-adjust: exact;
      print-color-adjust: exact;
      body { font-size: 8.5pt; }
      .page { padding: 0; }
      @page { size: letter portrait; margin: 0.4in; }
    }
  </style>
</head>
<body>
  <div class="page">
    <div class="title-block">
      <h1>[TOPIC] Cheat Sheet</h1>
      <div class="meta">Generated [DATE] &bull; Print: [PAPER] &bull; [COLOR MODE]</div>
    </div>
    <div class="grid">
      <!-- sections go here -->
    </div>
  </div>
</body>
</html>
```

Apply section accent colors by setting `border-left-color` and `background-color` inline or via a `<style>` block with `.s1 .section-header { border-left-color: #1a73e8; }` etc. For formula boxes in that section, add `border-color: #1a73e8; background: #e8f0fe;`.

In grayscale mode: use `border-left-color: #555` for all sections, `background: #f5f5f5` for boxes.

**Span a section across all 3 columns** when it contains a wide table: add `grid-column: 1 / -1` to that `.section`.

#### Using the `frontend-design` skill for HTML

When generating an HTML cheat sheet, invoke the `frontend-design` skill's visual sensibility:
- Clean whitespace rhythm (4px/8px spacing increments)
- Subtle shadows on formula boxes (`box-shadow: 0 1px 3px rgba(0,0,0,0.1)`)
- Rounded corners (3–4px) on boxes and section headers
- Consistent accent colors with 10–15% opacity backgrounds
- Production-grade feel: no gradients, no decorative fluff — just clarity

---

### LaTeX Format

Generate a complete `.tex` file that compiles with `pdflatex`.

Packages to use:
```latex
\usepackage[margin=0.5in, landscape]{geometry}  % or portrait for letter
\usepackage{multicol}
\usepackage{amsmath, amssymb, amsthm}
\usepackage{tcolorbox}
\usepackage{xcolor}
\usepackage{booktabs}
\usepackage{microtype}
\usepackage{enumitem}
\setlength{\parindent}{0pt}
\setlength{\columnsep}{1em}
```

Layout: `\begin{multicols}{3}` … `\end{multicols}` for 3 columns on landscape letter.

Formula boxes: use `tcolorbox` with `colback` for section-specific color.

For grayscale: use `\colorlet` to map all colors to shades of gray.

For paper size, set `\usepackage[margin=0.5in, <paper>, <orientation>]{geometry}`:
- letter = `letterpaper`
- a4 = `a4paper`
- legal = `legalpaper`
- a5 = `a5paper`

Include compile instructions at the top as a comment:
```latex
% Compile with: pdflatex cheatsheet.tex
```

---

### Markdown Format

Generate a structured Markdown file, organized with:
- H1 for the title
- H2 for each section
- H3 for sub-topics
- Fenced code blocks for formulas (use LaTeX syntax: `$formula$` inline, `$$formula$$` block)
- Tables for reference data
- Bold for key terms

This format is the simplest — useful for importing into Obsidian, Notion, or other note-taking tools.

---

## Step 5 — Write the File

Write the output to a file named `<topic-slug>-cheatsheet.<ext>` in the current directory, where:
- `<topic-slug>` = topic name lowercased, spaces replaced with hyphens (e.g., `ap-physics-c-cheatsheet.html`)
- `<ext>` = `html`, `tex`, or `md`

After writing the file, tell the user:
- The filename
- How to open/view it (for HTML: open in browser, then File > Print or Ctrl+P; for LaTeX: compile with `pdflatex`)
- Any topic areas that may be incomplete or where they should verify content against their specific exam's official formula sheet

## Quality Bar

Before writing the file, mentally verify:
- [ ] Every major testable formula for this exam/topic is included
- [ ] Math is written correctly in the chosen format (KaTeX for HTML, LaTeX syntax for `.tex`, `$...$` for Markdown)
- [ ] No section is empty
- [ ] The layout fits on the paper size when printed (density is correct — not too sparse, not impossible to read)
- [ ] Color/grayscale mode is consistently applied throughout
- [ ] The file is self-contained (HTML has no broken external links beyond CDN)
