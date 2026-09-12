---
name: figure
description: Use this skill WHENEVER you generate, modify, or save a scientific-data-analysis figure or plot — any time the work involves a figure, plot, panel, subplot, legend, axis, savefig, matplotlib, or code that emits a figure file. It enforces the lab's publication-figure standards: two bold-titled legends at 10 pt — a brief journal-style caption on the figure page and a detailed legend on a second page of the same PDF — 14 pt panel letters, all on-figure text ≥ 8 pt, identical axis limits across panels meant for comparison, panel edges that line up across rows and columns, output as PDF + PNG plus a clickable .code.html source listing (sorted into pdf/ png/ html/ subfolders), and a programmatic no-text-overlap check that MUST pass before a figure is considered done.
---

# Figure standards

Every figure this project emits is a **publication-grade artifact**. Before a figure is "done"
it must satisfy all eight rules below. These are non-negotiable; do not skip the overlap check
because a figure "looks fine."

## The eight rules

1. **TWO legends: a brief one on the figure page, a detailed one on its own page.** Both
   carry a **bold title** lead-in and a body at **10 pt**. The **brief** legend goes on the
   figure page and reads like a real journal caption — what each panel shows and the one or
   two facts a reader needs in order to read it, and nothing else. The **detailed** legend
   goes on a **second page of the same PDF** and carries everything a figure needs to stand
   alone: windows, thresholds, exclusions, n, provenance, and why each non-obvious choice was
   made. Two-page figure PDFs are expected and fine. One long combined legend holds the same
   content, but nobody can find the crucial sentence in it — that is the failure this splits.
2. **Panel letters at 14 pt.** Each subpanel is labelled A, B, C … in reading order, **bold, 14 pt**.
3. **No text overlaps anything — checked programmatically.** No legend, title, axis label, panel
   letter, annotation, colorbar label, or source link may overlap another text or a panel. This is
   verified by a bounding-box check at save time, **not by eye**. A non-empty report is a blocker.
4. **PDF + PNG + `.code.html`, text embedded as real text, sorted into type subfolders.** Save a
   vector **PDF**, a same-named **PNG** companion (for GitHub previews), and an auto-generated
   **`<fig>.code.html`** listing the functions used to build the figure, linked from a clickable
   annotation inside the PDF. **Within each figure folder these three artifact types are kept in
   sibling subfolders — the PDF in `pdf/`, the PNG in `png/`, the `.code.html` in `html/`** — so a
   folder holding many figures stays navigable (`savefig` routes each type there automatically; see
   below). PDF/PS text must be embedded as **real, selectable/editable text — never vectorized into
   outlines**; matplotlib defaults to Type-3 fonts (`pdf.fonttype = 3`), so force
   **`pdf.fonttype = 42`** (and `ps.fonttype = 42`) to embed the actual TrueType font. `apply_style()`
   does this; if a project ever emits SVG, also set `svg.fonttype = 'none'`.
5. **Both legends are broken out per panel.** Write each body as `(A) … (B) …` references.
   In the **brief** legend that is one clause per panel — the variable on each axis and the
   condition, at most a sentence. In the **detailed** legend it is the full account: axes,
   conditions, groups, pooling, n, time windows, exclusions, and a note on the analytical
   approach, so the pair together lets the figure stand alone.
6. **All on-figure text ≥ 8 pt.** Nothing smaller than 8 pt anywhere.
7. **Shared axis limits when panels are meant to be compared.** When several panels plot the
   **same measured variable** and the figure's purpose is to compare across them, give those panels
   **identical axis limits** — independent autoscaling silently rescales each panel, so different
   values look alike and the comparison is defeated (or worse, misleading). Set it explicitly:
   matplotlib `sharey=True` / `sharex=True` on the shared quantity's axis, or a common
   `set_ylim` / `set_xlim` computed from the pooled data. Axes that legitimately differ per panel —
   a *different* variable, or a deliberately different (e.g. log) scale — stay independent; this
   rule is only about the axis the panels have in common.
8. **Panel edges line up.** Panels stacked above one another share a **left edge** (and a right
   edge where their widths allow); panels side by side in a row share a **bottom edge** (and a
   top edge). A reader uses panel edges as the figure's implicit grid, so one panel inset by a few
   points reads as a mistake even when the data is perfect. **The key shared edges are exact, not
   approximate** — see the section below for the four ways a layout engine breaks this silently and
   how to pin each one down. Compromises are unavoidable (a half-width panel cannot align its right
   edge with the full-width panel beneath it); spend the slack on edges nobody compares and keep the
   shared ones exact.

These rules are about figure *formatting, provenance, and legibility*. They say nothing about the
science — keep the legend's wording accurate to the data and defer all domain/biology wording to
the project's own `CLAUDE.md`.

## Rule 8 in practice — the four ways alignment breaks silently

None of these raise, none of them are caught by the overlap check, and all of them look like
sloppiness in the printed figure. Handle each deliberately.

**1. Place every panel from one `GridSpec`.** A single `fig.add_gridspec(..., left=, right=)`
gives every panel in a column the same edge by construction. Panels positioned by hand with
`add_axes([...])`, or built from several gridspecs with different margins, have to be kept in
agreement by arithmetic that silently rots the next time a margin changes. Nest with
`gs[row, :].subgridspec(...)` so a montage row inherits the parent's `left`/`right` instead of
declaring its own.

**2. `aspect="equal"` shrinks the axes and then CENTRES it.** A fixed-aspect axes whose allotted
cell is the wrong shape keeps one dimension, shrinks the other, and centres what remains inside the
cell — so its left edge moves inward with no warning. `imshow` sets `aspect="equal"` by default, so
**every image panel is affected**. Two fixes, use both:

- `ax.set_anchor("SW")` pins the surviving box to the bottom-left of its cell, so the left and
  bottom edges hold even when the cell is the wrong shape.
- Size the row so the dimension you are aligning is the one that survives. If the cell is *taller*
  than the aspect needs, the width is preserved and left and right edges are both exact; if it is
  wider, only the anchored edges are.

**3. Align the drawn content, not just the axes.** An axes at the right position is still
misaligned if what the reader reads as the panel's edge sits inside it — a frame drawn as a
`Rectangle` patch inset from the data limits, or an image whose `extent` does not start at
`xlim[0]`. Either set the limits so the content's own edge *is* the axes edge, or draw that frame
as the axes spines and let the axes box be the frame.

**4. Colourbars and insets overhang.** `ax.inset_axes([1.07, ...])` puts a colourbar outside the
panel's right edge, and its tick labels go further still. If that right edge is one you are
aligning, budget the figure's margin for the overhang instead of pulling the panel inward to make
room.

**Then verify it, the same way rule 3 is verified — by measuring, not by eye.** Rendered axes
positions are available after a draw, so assert the shared edges agree to a fraction of a point:

```python
fig.canvas.draw()
edges = [ax.get_window_extent().x0 for ax in (ax_a, ax_c, first_tile)]
assert max(edges) - min(edges) < 0.5, f"left edges disagree by {max(edges) - min(edges):.2f} px"
```

A sub-pixel tolerance is right: these edges are either equal by construction or they are wrong.

## Units and symbols

Use the correct Unicode symbol for units and quantities, as a real embedded glyph (rule 4) — not an
ASCII stand-in. In particular, **micrometers are `µm` with the micro sign µ (U+00B5), never `um`
with a plain "u"**. Likewise prefer literal `±`, `°`, `Δ`, and Greek letters (µ, α, β, …) over
`+/-`, `deg`, `delta`, or `u`/`a`/`b`. matplotlib's default font (DejaVu Sans) contains these
glyphs, so write the literal character (not a mathtext `$\mu$`) so the text stays
selectable/editable; if a font lacks a glyph the overlap check still passes but the PDF shows a
missing-glyph box, so confirm it in the rendered PNG.

## Step 0 — make the helper available (do this first)

Rules 1–6 are implemented in one bundled, dependency-light module (pure matplotlib, no
seaborn) — rule 7 is a plotting-time choice you make when building the axes (see below). The
module ships with this plugin at:

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
- **Brief legend (rules 1 & 5):** `justified_legend(fig, brief_title, brief_body)` on the
  figure itself.
  - `title` is the bold lead-in (e.g. `"Graded E:I modulation of V1 responses."`).
  - `body` is one clause per panel: `"(A) ...  (B) ...  (C) ..."`. **Read the panel-building
    code before writing it** — brief does not mean vague, and a caption that misstates an axis
    is worse than a long one.
  - It renders at 10 pt, auto-places below the lowest panel text, and only shrinks toward an 8 pt
    floor to avoid an overlap. Reserve a bottom band so it stays at the full 10 pt: make the figure
    taller and call `fig.subplots_adjust(bottom=…)`. A brief legend needs far less band than a
    combined one did — that room goes back to the panels.
- **Detailed legend (rule 1):** pass `detail=(detail_title, detail_body)` to `savefig`, with
  `detail_x=(left, right)` set to the figure's own panel margins so both pages share a measure.
  `savefig` builds the second page with `detail_legend_page`, sizes it to the text so the legend
  cannot shrink and the page is not mostly blank, writes both pages into the one PDF, and runs the
  overlap gate on each. Do not build the second page by hand or bolt the detail onto the brief
  legend's body.
- **Save with provenance (rule 4):** `savefig(fig, "<name>", outdir="figures/<subdir>", functions=[…])`.
  - PDF + PNG are written by default (`png_companion=True`).
  - **The three artifacts are routed into type subfolders automatically:** the PDF into
    `figures/<subdir>/pdf/`, the PNG into `figures/<subdir>/png/`, and the `.code.html` into
    `figures/<subdir>/html/`. You pass the plain figure folder; `savefig` does the sorting — don't
    add `pdf`/`png`/`html` to the name or `outdir` yourself.
  - `functions=[…]` is **mandatory**: pass every callable used to build the figure — the script's
    own `main` / `collect` / panel helpers **plus** the helper functions used
    (`justified_legend`, `panel_label`, and any project analysis functions). This produces
    `<name>.code.html` and embeds the clickable "▸ source code" link.
- **Shared axes (rule 7):** if the panels share a measured variable for comparison, tie their
  common axis at construction — `plt.subplots(…, sharey=True)` (or `sharex=True`), or compute a
  pooled `set_ylim`/`set_xlim` and apply it to every panel. `savefig` cannot infer this — it is
  your choice when you build the axes.
- **All text ≥ 8 pt (rule 6):** guaranteed by `apply_style()` (tick/legend floor 8 pt) and the
  legend's 8 pt shrink floor. If you set any font size by hand, keep it ≥ 8 pt.
- **Aligned panel edges (rule 8):** one `GridSpec` with a single `left`/`right`, `set_anchor("SW")`
  on every fixed-aspect axes (which includes every `imshow` panel), data limits set so the drawn
  frame is the axes edge, and a measured assertion on the shared edges before saving. `savefig`
  cannot infer any of this — like rule 7 it is a choice you make when you build the axes. See
  "Rule 8 in practice" above.

Multi-page figures (`PdfPages`) build the output path by hand, so route it into the type subfolders
yourself with `fig_dir`: write the PDF to `fig_dir("figures/<subdir>", "pdf") / "<name>.pdf"` and put
the listing in `html/` by passing `sidecar_dir=fig_dir("figures/<subdir>", "html")` to
`write_code_listing(pdf_path, [...], sidecar_dir=…)`. Then `attach_source_link(fig, sidecar)` on each
page and save pages via `pdf_savefig(pdf, fig, name=…)` so each page still runs the overlap check.

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

- [ ] **Brief** legend on the figure page: **bold title**, body at **10 pt**, one clause per
      panel, journal-caption length — no windows/thresholds/provenance.
- [ ] **Detailed** legend on page 2 of the same PDF (`savefig(..., detail=(title, body))`), at
      **10 pt**, carrying the windows, thresholds, exclusions, n, provenance and the reasons
      behind each non-obvious choice.
- [ ] Panel letters A, B, C … in reading order, **14 pt** bold.
- [ ] `savefig(..., functions=[…])` ran → the `.pdf`, `.png`, and `.code.html` are in the figure
      folder's `pdf/`, `png/`, and `html/` subfolders, and the in-PDF "▸ source code" link is present.
- [ ] Overlap check is **clean**: no `⚠ TEXT OVERLAP` printed, **no `*.overlap.txt` sidecar** left.
- [ ] No on-figure text smaller than **8 pt** anywhere.
- [ ] Panels that plot the same variable for comparison share **identical axis limits** on the common
      axis (`sharey`/`sharex` or a pooled `set_ylim`/`set_xlim`).
- [ ] Shared panel edges **measured and asserted** after a draw — left edges equal down a column,
      bottom edges equal across a row — not judged by eye. Fixed-aspect and `imshow` panels carry
      `set_anchor("SW")`.
- [ ] Legend wording is accurate to what the code actually plots (you read the panel code first).
