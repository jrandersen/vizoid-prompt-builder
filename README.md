# vizoid-prompt-builder

A Claude skill that assembles AI rendering prompts for [Vizoid](https://vizoid.ai) from uploaded architectural wireframes or line drawings.

---

## What It Does

Vizoid is an AI tool that generates photorealistic interior renders from text prompts. Writing those prompts well requires precise, consistent language about materials, fixtures, and spatial relationships — language that needs to stay coherent across every camera angle of a project.

This skill automates that prompt assembly. When you upload a wireframe or line drawing of a room and identify the view, it:

1. **Reads the project library** — `blocks.txt`, `fixtures.txt`, and `materials.txt`
2. **Parses your image** — determines which elements are fully visible, partially visible, or out of frame
3. **Assembles a prompt** — pulls the right modular blocks, resolves fixture slots to full descriptions, and adds explicit exclusions for what's not in frame
4. **Saves the view** — writes the assembled prompt to `views/` with a calibration status flag
5. **Outputs a paste-ready prompt** — plus a 3-line manifest of what's included, excluded, and flagged

---

## Library Architecture

The skill is built around three library files that serve as a single source of truth for the project's design language:

### `blocks.txt` — Prompt Blocks

Modular prompt fragments organized by tier:

- **T1 (Tier 1 — Home-wide):** Always-include blocks for style anchor, walls, ceiling, beams, windows, doors, and flooring. Applied to every view regardless of room.
- **T2 (Tier 2 — Room-specific):** Kitchen (or other room) blocks for cabinets, countertops, backsplash, open shelving, hood, appliances, island, pendants, and dining. Included only when the element is visible in the image.

Each block has a visual trigger (e.g. "trigger: any cabinet face or drawer visible") and named fixture slots that reference `fixtures.txt` by ID rather than embedding inline descriptions.

### `fixtures.txt` — Fixture Descriptions

Full prose descriptions for every fixture in the project: pendant styles, faucets, appliances, seating. Each entry has:
- A unique ID used in block slots
- Active/alternate/retired status
- Calibration status from Vizoid render testing
- Exclusion language telling the renderer what *not* to produce

To swap a fixture in a view, change the slot reference — no block edits needed.

### `materials.txt` — Material Specs

Surface material definitions for countertops, flooring, and other finishes. Same structure as fixtures — default, alternates, calibration status. Countertop swap points are flagged in assembled prompts.

### `views/` — Saved Assembled Prompts

One file per camera angle. Each view file stores the full assembled prompt, the list of active blocks, spatial orientation (room-direction → camera-relative mapping), island mode, and calibration status (`PENDING CALIBRATION` → `CALIBRATED ✓`).

---

## Calibration Loop

After submitting a prompt to Vizoid:
- **Good render** → update the view file status to `CALIBRATED ✓`
- **Bad render** → identify which block caused the issue, revise it in `blocks.txt` so all future views benefit
- **New fixture found** → add it to `fixtures.txt` and drop a reference image in the folder

---

## Skill Trigger

The skill activates when you:
- Upload a wireframe, line drawing, or shaded view of a room and identify the room or view angle
- Mention **Vizoid**, **rendering**, or **interior prompt building**

---

## Adding New Rooms

1. Append a `TIER 2 — [ROOM NAME]` section to `blocks.txt`
2. Add relevant fixtures to `fixtures.txt` under a new category header
3. Add any new materials to `materials.txt`
4. No changes to `SKILL.md` needed — it reads the library files dynamically

---

## File Structure

```
vizoid-prompt-builder/
├── SKILL.md          # Claude skill instructions
├── blocks.txt        # T1 (home-wide) and T2 (room) prompt blocks
├── fixtures.txt      # Fixture descriptions — pendants, faucets, appliances, seating
├── materials.txt     # Surface material specs — countertops, flooring, etc.
└── views/            # Assembled prompts, one file per camera angle
    ├── kitchen-entry-hall.txt      # CALIBRATED ✓
    └── kitchen-north-wall.txt      # PENDING CALIBRATION
```

---

## Current Project

Mid-Century Modern Residential Interior — Alder PLLC
Rooms covered: **Kitchen**
Style: Warm cream walls, dark walnut cabinets, Calacatta Miele stone, white oak floors, brass accents
