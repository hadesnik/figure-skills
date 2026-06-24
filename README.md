# figure-skills

A Claude Code **plugin marketplace** holding the lab's figure-generation standards. Add it once and
every team member gets the same `/figure` behavior — and any new skills we add later arrive as
updates, with no per-repo copying.

## What's in here

```
figure-skills/                              # this repo = the marketplace
├── .claude-plugin/marketplace.json         # lists the plugin(s)
└── plugins/
    └── figure-standards/                   # the plugin (holds a growing set of skills)
        ├── .claude-plugin/plugin.json
        ├── helpers/figure_helpers.py        # shared drop-in helper (pure matplotlib, no seaborn)
        └── skills/
            └── figure/SKILL.md              # the /figure-standards:figure skill
```

The `figure` skill enforces six rules on every figure: a bold-titled, self-contained legend at
**10 pt** that describes each panel; **14 pt** panel letters; **all text ≥ 8 pt**; output as
**PDF + PNG**; an auto-generated **`.code.html`** listing the functions used (clickable from the
PDF); and a **programmatic no-text-overlap check** that must pass before a figure is done. The
heavy machinery lives in `helpers/figure_helpers.py`, which the skill copies into (or matches
against) whatever project you're working in.

## Install (each team member, once)

```
# 1. Add this marketplace. Use the git remote once it's pushed:
/plugin marketplace add hadesnik/figure-skills
#    (or a full URL, e.g. https://github.com/hadesnik/figure-skills.git
#     or a local clone path for testing: /plugin marketplace add ~/path/to/figure-skills)

# 2. Install the plugin:
/plugin install figure-standards@figure-skills
```

Then, in any repo, the skill is available as `/figure-standards:figure` and Claude will also invoke
it automatically whenever it builds or edits a figure.

## Updating

`plugin.json` intentionally has **no `version` field**, so every commit pushed to this repo is
treated as a new version. Team members pull updates with:

```
/plugin marketplace update figure-skills
```

## Adding another figure skill later

1. Create `plugins/figure-standards/skills/<new-skill>/SKILL.md` (frontmatter: `name`, `description`).
2. Reuse the shared helper — reference `${CLAUDE_PLUGIN_ROOT}/helpers/figure_helpers.py` from the
   skill body, exactly as `skills/figure/SKILL.md` does.
3. Commit and push. Team members get it on the next `/plugin marketplace update`.

To add an unrelated plugin (not just a skill), create another folder under `plugins/` and list it
in `.claude-plugin/marketplace.json`.

## Using the helper directly (no Claude)

`helpers/figure_helpers.py` is a standalone module — copy it into a project and:

```python
from figure_helpers import apply_style, panel_label, justified_legend, savefig

apply_style()
# ... build fig with subplots, call panel_label(ax, "A") per panel ...
justified_legend(fig, "Bold title.", "(A) ...  (B) ...")
savefig(fig, "myfigure", outdir="figures", functions=[main, justified_legend, panel_label])
```

Run its self-check with `python figure_helpers.py --selftest` (verifies the overlap detector flags
a deliberately overlapping figure and passes a clean one).
