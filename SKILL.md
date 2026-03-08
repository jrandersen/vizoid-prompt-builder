---
name: Vizoid Interior Rendering Prompt Builder
description: Assembles structured Vizoid AI rendering prompts from uploaded room wireframes or line drawings. Activate when the user uploads a base image (wireframe, line drawing, or shaded view) and names the room or view angle, or mentions Vizoid, rendering, or interior prompt building. Reads the project's blocks/fixtures/materials library, parses the image to determine which elements are visible, assembles a complete paste-ready prompt with explicit exclusions, and saves the result to views/ with a calibration status flag.
license: Alder PLLC internal
---

# SKILL: Vizoid Rendering Prompt Builder
**Project:** Alder PLLC — Mid-Century Modern Residential Interior
**Last Updated:** 2026-03-06

---

## TRIGGER
Activate this skill when the user uploads a base image (wireframe, line drawing, or shaded view)
and identifies a room, or mentions Vizoid, rendering, or prompt building.

---

## LIBRARY FILES — read all three at the start of every session

| File | Contents |
|------|----------|
| `blocks.txt` | All T1 (home-wide) and T2 (room) prompt blocks with visual triggers |
| `fixtures.txt` | All fixture descriptions — pendants, faucets, appliances, seating |
| `materials.txt` | All surface material specs — countertops, flooring, etc. |
| `views/` | Saved assembled prompts, one file per view |

---

## WORKFLOW — 5 steps

### 1 · READ
Read blocks.txt, fixtures.txt, and materials.txt before doing anything else.
Also scan views/ to check if this view angle has been assembled before. The image filename will designate the view.

### 2 · PARSE THE IMAGE
Examine the uploaded image. For each element, determine: FULL · PARTIAL · ABSENT.

**Home-wide (T1):** ceiling · beams · walls · windows · doors · floor

**Kitchen (T2):** cabinets · countertops · backsplash · open shelving · hood ·
appliances · sink/faucet · island (full or edge-partial?) · pendant · dining/banquette

**Special conditions to catch:**
- Tall cabinet panels spanning floor-to-soffit? → changes cabinet scale language
- Island: full body visible, or near edge only? → selects FULL vs PARTIAL island prompt
- Any windows blown out or dark? → always include exterior tree language
- No beams in frame? → omit T1-D

### 3 · BUILD MANIFEST (internal — don't show user unless asked)
```
INCLUDE (FULL):    [block IDs]
INCLUDE (PARTIAL): [block IDs + note]
EXCLUDE:           [block IDs + reason]
FIXTURES USED:     [fixture IDs from fixtures.txt]
SWAP POINTS:       [any ⚑ flags]
SPECIAL:           [anything unusual]
```

### 4 · ASSEMBLE PROMPT
Order: T1-A → remaining T1 (visible only) → T2 blocks (visible only) → explicit exclusion line → atmosphere line.

**For blocks with SLOTS:**
1. Read the SLOT definition to identify which fixture IDs are active
2. Pull the full fixture description from fixtures.txt by that ID
3. Insert the fixture description into the prompt at that position
4. Use PLACEMENT language from the block, translated into view-relative terms using the view's SPATIAL_ORIENTATION header
5. To hot-swap a fixture, change the slot reference — no other edits needed

**Exclusion line format:**
"Do not add [list of elements not visible in this view] — not visible in this view."

**Atmosphere line:** Write one sentence default based on window placement and light direction visible
in the image. Flag it as something the user should refine.

**Swap points:** When T2-B (countertops) is included, append:
`⚑ SWAP POINT — default is Calacatta Miele. See materials.txt > countertops for alternates.`
Do not swap the default unless the user has explicitly requested it.

### 5 · SAVE VIEW + OUTPUT

Save to `views/[room]-[angle-descriptor].txt` using this format:
```
VIEW_ID: [id]
VIEW_LABEL: [Room — Angle]
ROOM: [room]
STATUS: PENDING CALIBRATION
WIREFRAME: [uploaded filename]
DATE: [today]
BLOCKS_ACTIVE: [list]
ISLAND_MODE: [full / edge-partial / absent]
NOT_IN_FRAME: [list]

--- ASSEMBLED PROMPT ---
[full prompt]
```

Output to user:
1. The full assembled prompt — ready to paste
2. A 3-line manifest: what's in, what's out, any flags
3. Confirmation of where the view was saved

---

## CALIBRATION LOOP

After user submits to Vizoid and gets a render back:
- **Good render:** Update view file STATUS → `CALIBRATED ✓`. Note the date.
- **Bad render:** Identify which block(s) caused the issue. Revise only those lines.
  Update the source block in blocks.txt so all future prompts benefit.
- **New fixture discovered:** Add to fixtures.txt. Drop reference image in same folder when available.

---

## ADDING NEW ROOMS

1. Append a new `TIER 2 — [ROOM NAME]` section to blocks.txt
2. Add relevant fixtures to fixtures.txt under a new category header
3. Add any new materials to materials.txt
4. No changes to SKILL.md needed — it reads the files dynamically

---

## TODO — PLANNED REFACTOR
> Discussed 2026-03-08. Do not implement until architecture is approved.

### ✓ TODO-1 · Separation of Concerns — Library Architecture (DONE 2026-03-08)
Currently `blocks.txt` contains inline fixture descriptions (e.g. the refrigerator sentence in T2-F)
that duplicate and can conflict with `fixtures.txt`. The fix:
- **`fixtures.txt`** = single source of truth for what every fixture *is* (description, exclusions, style)
- **`blocks.txt`** = references fixture IDs only — knows *which* fixtures belong and *where* in the room, but contains no inline descriptions
- Assembling a prompt means: read T2 block for fixture slot → pull full description from fixtures.txt by ID
- A fixture update then only ever happens in one place

### ✓ TODO-2 · Remove Absolute Spatial Language from blocks.txt (DONE 2026-03-08)
Current blocks use hardcoded directions: "west wall", "to the right of the cooktop", "left of center".
These break when the camera angle changes.
- **`blocks.txt`** = view-agnostic placement (e.g. "refrigerator pair lives in the back wall cabinet run")
- **View files** = resolve room positions into view-relative language at assembly time
- Add a `SPATIAL_ORIENTATION:` header to each view file mapping room directions to camera-relative directions

### ✓ TODO-3 · Formal Fixture Slots with Hot-Swap Support (DONE 2026-03-08)
Each T2 block should define named fixture slots (e.g. `SLOT: refrigerator`, `SLOT: pendant`).
Each slot has a default fixture ID and a list of alternates from fixtures.txt.
Swapping a fixture = changing one slot reference, not rewriting a block.
This formalizes the existing ⚑ SWAP POINT pattern into proper slot architecture.

### ✓ TODO-4 · Fix T2-F Immediately (pre-refactor patch) (DONE 2026-03-08 — resolved by TODO-1)
T2-F still contains "Refrigerator, if visible, is panel-ready with walnut slab front..."
This is wrong and conflicts with `appliance-refrigerator-pair-dark-stainless` in fixtures.txt.
Replace the inline refrigerator sentence in T2-F with a fixture ID reference as a stopgap
until the full TODO-1 refactor is complete.

### TODO-5 · Align with PyRevit App
The same library architecture (fixtures.txt as source of truth, blocks.txt as view-agnostic layout,
view files as spatial translation layer) should be mirrored in the PyRevit app.
The PyRevit app reads the same library files directly — fixture swaps happen in one place
and propagate to both the Cowork skill and the app.

---

## FILE MAP
```
/mnt/skills/user/vizoid/
├── SKILL.md         ← this file (instructions)
├── blocks.txt       ← all T1 + T2 prompt blocks
├── fixtures.txt     ← all fixture descriptions
├── materials.txt    ← all material specs
└── views/           ← saved assembled prompts (auto-generated)
    ├── kitchen-entry-hall.txt     ← CALIBRATED ✓
    └── kitchen-north-wall.txt     ← PENDING CALIBRATION
```
