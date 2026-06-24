---
name: figure
description: Use this skill WHENEVER you generate, modify, or save a scientific-data-analysis figure or plot — any time the work involves a figure, plot, panel, subplot, legend, axis, savefig, matplotlib, or code that emits a figure file. It enforces the lab's publication-figure standards: a bold-titled, self-contained legend at 10 pt that describes every panel, 14 pt panel letters, all on-figure text ≥ 8 pt, output as PDF + PNG plus a clickable .code.html source listing, and a programmatic no-text-overlap check that MUST pass before a figure is considered done.
---

# Figure standards

Every figure this project emits is a **publication-grade artifact**. Before a figure is "done"
it must satisfy all six rules below. These are non-negotiable; do not skip the overlap check
because a figure "looks fine."

## The six rules

1. **Legend with a bold title, body at 10 pt.** Every figure carries a self-contained legend:
   a **bold title** lead-in followed by the body, rendered at **10 pt**.
2. **Panel letters at 14 pt.** Each subpanel is labelled A, B, C … in reading order, **bold, 14 pt**.
3. **No text overlaps anything — checked programmatically.** No legend, title, axis label, panel
   letter, annotation, colorbar label, or source link may overlap another text or a panel. This is
   verified by a bounding-box check at save time, **not by eye**. A non-empty report is a blocker.
4. **PDF + PNG + `.code.html`.** Save a vector **PDF**, a same-named **PNG** companion (for GitHub
   previews), and an auto-generated **`<fig>.code.html`** listing the functions used to build the
   figure, linked from a clickable annotation inside the PDF.
5. **The legend says what each panel shows.** Break the body into per-panel references —
   `(A) … (B) …` — naming exactly what each panel shows (axes, conditions, groups, pooling, time
   windows) plus a brief note on the analytical approach, so the figure stands alone.
6. **All on-figure text ≥ 8 pt.** Nothing smaller than 8 pt anywhere.

These rules are about figure *formatting and provenance*. They say nothing about the science —
keep the legend's wording accurate to the data and defer all domain/biology wording to the
project's own `CLAUDE.md`.

## Step 0 — make the helper available (do this first)

All six rules are implemented in one bundled, dependency-light module (pure matplotlib, no
seaborn). It ships with this plugin at:

```
${CLAUDE_PLUGIN_ROOT}/helpers/figure_helpers.py
```

Before writing figure code, ensure the project can import it:

- **If the project already has an equivalent helper** (a `plotting.py` / `savefig` that does
  PDF+PNG, the overlap check, and the `.code.html` listing), use it. Verify its defaults match the
  rules — legend **10 pt** body, panel letters **14 pt**, overlap check **on** by default,
  PNG companion **on**. If they don't, fix the defaults rather than passing overrides at every call.
- **Otherwise, copy the bundled helper into the project** so figures don't depend on the plugin
  path at runtime. Put it somewhere importable — the project's package (e.g. `<pkg>/figure_helpers.py`)
  or a `fig_util/` directory — by copying `${CLAUDE_PLUGIN_ROOT}/helpers/figure_helpers.py`. Then
  `from figure_helpers import savefig, panel_label, justified_legend` (adjust the import path).

Do not re-implement the overlap check or the code-listing from scratch — reuse the helper.

## How each rule is satisfied (with the helper)

Call `apply_style()` once at startup, then build the figure and finish with these:

- **Panel letters (rule 2):** `panel_label(ax, "A")` for each subpanel in reading order. It is
  bold 14 pt by default — don't shrink it.
- **Legend (rules 1 & 5):** `justified_legend(fig, title, body)`.
  - `title` is the bold lead-in (e.g. `"Graded E:I modulation of V1 responses."`).
  - `body` is the per-panel description: `"(A) ...  (B) ...  (C) ..."`. **Read the panel-building
    code before writing this** — the text must accurately state each panel's axes, conditions,
    groups, pooling, and time windows, plus a one-line note on the analytical approach.
  - It renders at 10 pt, auto-places below the lowest panel text, and only shrinks toward an 8 pt
    floor to avoid an overlap. Reserve a bottom band so it stays at the full 10 pt: make the figure
    taller and call `fig.subplots_adjust(bottom=…)` (≈ 0.22–0.32 depending on legend length).
- **Save with provenance (rules 4):** `savefig(fig, "<name>", outdir="figures/<subdir>", functions=[…])`.
  - PDF + PNG are written by default (`png_companion=True`).
  - `functions=[…]` is **mandatory**: pass every callable used to build the figure — the script's
    own `main` / `collect` / panel helpers **plus** the helper functions used
    (`justified_legend`, `panel_label`, and any project analysis functions). This produces
    `<name>.code.html` and embeds the clickable "▸ source code" link.
- **All text ≥ 8 pt (rule 6):** guaranteed by `apply_style()` (tick/legend floor 8 pt) and the
  legend's 8 pt shrink floor. If you set any font size by hand, keep it ≥ 8 pt.

Multi-page figures (`PdfPages`): call `write_code_listing(pdf_path, [...])` once, then
`attach_source_link(fig, sidecar.name)` on each page, and save pages via `pdf_savefig(pdf, fig, name=…)`
so each page still runs the overlap check.

## Rule 3 is a hard gate

`savefig` (and `pdf_savefig`) run `check_text_overlaps` automatically. On any overlap they print a
loud `⚠ TEXT OVERLAP …` report and write a `<fig>.overlap.txt` sidecar next to the figure.

**Treat a non-empty report / any `*.overlap.txt` as an error to fix — not a warning.** Fix it and
**regenerate** until the report is clean and no sidecar remains:
- make the figure taller and enlarge `fig.subplots_adjust(bottom=…)` so the legend band has room;
- reposition or shrink the offending text (keep it ≥ 8 pt);
- move a colorbar/inset out of the way (give a colorbar its own `fig.add_axes([...])`);
- keep the legend at `y_top=None` (auto-placement) rather than pinning it.

Pass `strict=True` to `savefig` in batch/CI runs so an overlap raises instead of silently leaving a
sidecar. Eyeballing the PNG is only a final backstop — the programmatic report is the gate.

## Definition of done — verify every box before declaring the figure finished

- [ ] Legend present, **bold title**, body at **10 pt**, broken into `(A) … (B) …` that accurately
      describes each panel (axes/conditions/pooling/windows + analytical note).
- [ ] Panel letters A, B, C … in reading order, **14 pt** bold.
- [ ] `savefig(..., functions=[…])` ran → a `.pdf`, a `.png`, and a `.code.html` exist side by side,
      and the in-PDF "▸ source code" link is present.
- [ ] Overlap check is **clean**: no `⚠ TEXT OVERLAP` printed, **no `*.overlap.txt` sidecar** left.
- [ ] No on-figure text smaller than **8 pt** anywhere.
- [ ] Legend wording is accurate to what the code actually plots (you read the panel code first).
