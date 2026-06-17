# Sticky Notes — Spec (Phase D, standalone)

The complete design + build spec for the Phase-D sticky-note overlay. This is a CANONICAL,
standalone spec — it lives on its own (sticky notes are a self-contained overlay, not a tile and
not in the grid/reflow/engine-dispatch path), and the four main specs POINT to it rather than
duplicate it. The one-line pointers are dropped in:
  - family-dashboard-spec.md → "Sticky-note overlay (Phase D)" section + the file manifest + the Build Status row
  - gui-spec.md (Home row), engine-spec.md (engine decomposition list), tiles-spec.md (Home → Sticky-notes)

Maintenance: same rules as the other specs — a live snapshot, not a log. Update at session
close-out; mark build steps done; delete anything no longer true. Sections below are grouped by
which of the four concerns they cover (project / design / engine / wiring).

================================================================================
BUILD STATUS (Progress)
================================================================================

Built in two passes. **Pass 1 (the overlay engine + plumbing) is DONE on glass.** Pass 2 (the
create-menu) is now BUILT on glass too.

**PASS 1 — overlay + plumbing — ✅ BUILT, on glass.**
The overlay renders the stored notes (photoreal paper, handwriting fonts), drag works
(long-press → arm/pulse → drag → persist), tap → Edit/Delete strip, Edit → on-glass keyboard,
Delete, auto-fade, and persistence across reloads. Files shipped:
  - **7 fonts** in `/config/www/fonts/` (`sn-lighthouse.ttf` · `sn-pastel.ttf` · `sn-lefty.otf` ·
    `sn-daniel.ttf` · `sn-gabriel.ttf` · `sn-jimmy.otf` · `sn-biro.otf`).
  - **`/config/www/notes-fonts.js`** — the global font loader (extra_module_url) + the FONTS table.
  - **`/config/notes_write.py`** — the atomic store writer (base64 full-array arg).
  - **`/config/www/notes/notes.json`** — the store (runtime data; seeded for testing).
  - **`/config/www/engine/notes.js`** — the `NotesMethods` overlay subsystem.
  - edits: `engine/tokens.js` (notes block) · `engine/styles.js` (the `.sticky-*` CSS) ·
    `engine/core.js` (mixin apply + build hook + store-read hook) · `configuration.yaml`
    (the font loader in `extra_module_url` + `shell_command.notes_write`).

**PASS 2 — the create-menu (`homeSticky`) — ✅ BUILT, on glass.**
`faces/home.js` `renderSticky` is now the real CREATE-ONLY menu: paper + ink 4+4 swatches (selected
outlined in cream) → a scrollable KIDS/ADULTS font list (each row previews in its own face, read
live from `window.WALL_NOTES_FONTS`) → a text field rendering LIVE in the picked font, edited on the
shared `FloatingKeyboard` (ABC/123, floated over a dismiss scrim — the chess precedent) → Add note.
The new note lands at a cascade offset and the field clears for the next.

  **ARCHITECTURE DEVIATION (load-bearing — supersedes the "menu fires `ctx.dispatch` notes_write"
  wording in the mirror sections below).** `notes_write.py` REPLACES the whole array, so a menu that
  fetched + appended + wrote on its own would be a SECOND writer racing the overlay's in-memory
  `this._notes` — a drag-end inside the ~4s read window would write the overlay's stale array and
  WIPE the new note. Instead the menu hands the user's CHOICES to the overlay through a new
  `ctx.addNote(rec)` seam; the overlay (already the array owner + sole writer) assigns
  id / cascade position / `created` + the baked look, renders the note immediately, and persists via
  the one `_writeNotes()` it already owns. Single source of truth, single writer, no race. This added
  `_notesAdd` + `_freshNoteLook` to `engine/notes.js` and the one-line `addNote` seam to the
  face-tap ctx in `engine/interactions.js` — two files the original Pass-2 plan didn't name.

**REFINEMENTS (logged, not blocking — tune-on-glass).**
  - Note size is `--note-size: 130px` (tuned down from 200 on glass). One-line change in styles.js
    `.sticky-pos`; the Pass-2 menu now drives the look variety (random tilt/bow/fold per note); size stays tune-on-glass.
  - The Edit/Delete-strip + keyboard CHROME colours are a hardcoded dark scrim + accent (functional
    overlay UI, not theme-tinted) — fine for v1; could be tokenised later.
  - Confirm the blend-mode/filter cost on the real panel under a full dozen notes (Open Item).

================================================================================
SOURCE FILES (where the raw build inputs live on the box)
================================================================================

- **/config/sticky_notes/** — the sticky-note source-input folder (NOT web-served; raw inputs only).
  - **/config/sticky_notes/sticky-note-master.html** — >>>[OWNER]<<<'s CSS prototype: the canonical LOOK
    source (the photorealistic paper — hue/sat, tilt + 3D bow, crease, multi-layer lighting, SVG
    fractal-noise grain, clip-path dog-ear folds, the probability-weighted-tilt manual). Adapted
    verbatim into engine/styles.js with the bake-once performance discipline.
  - **/config/sticky_notes/sn_fonts/** — the 7 dafont handwriting source files (.ttf/.otf).

These are dev INPUTS, not runtime artifacts: at build, the prototype's CSS was folded into
engine/styles.js and the 7 fonts were COPIED (box-side `cp`) to `/config/www/fonts/` and are served
directly as TTF/OTF (see the woff2 deviation under ENGINE → Fonts). The /config/sticky_notes/ folder
is never web-served.

================================================================================
FAMILY-DASHBOARD-SPEC.md  — the "Sticky-note overlay (Phase D)" section points here
================================================================================

## Sticky-note overlay (Phase D)

A separate transparent layer on top of the Wall — NOT a tile on the grid. By design it sits apart
from Sections, the grid, reflow, and cascade — none of that architecture knows it exists. It is the
LAST thing built (nothing depends on it).

- **Home:** a Wall-level overlay layer mounted on the card's shadow root, exactly like the
  banner-layer precedent (position:absolute, pointer-events:none on the layer, auto only on the
  note rectangles). It never reflows the grid, survives flip/cascade/board-flip, and the packer
  never knows it exists. **z-index ABOVE the banner layer (banner=6 → notes=7).** (Because the layer
  is pointer-events:none except on notes, a banner stays tappable everywhere a note isn't.)

- **Multiple notes, free placement.** Absolutely-positioned elements, each storing its own
  x/y + text + paper colour + ink colour + font + a few baked look params (tilt/bow/folds/crease).
  New notes spawn at a slight **cascade offset** from the last one. **Soft cap ~12 notes**
  (the look uses blend-mode/filter layers; a dozen is the ceiling — see Performance).

- **Look — >>>[OWNER]<<<'s CSS prototype (sticky-note-master.html).** Photorealistic CSS paper: warm
  paper colour, per-note tilt + 3D bow, a crease line, a multi-layer lighting pass, an SVG
  fractal-noise grain, and clip-path dog-ear folds (TL/TR/BL/BR). Handwriting is a real web font.
  Each note's look params are baked once at create.

- **PERFORMANCE — the override lens (load-bearing).** Bake the entire look ONCE at create and never
  recompute it. The ONLY things that ever animate are `transform` (drag + the long-press pulse) and
  `opacity` (fade-in / auto-fade-out). No blend/filter/layout property is ever animated.
  `will-change:transform` is added on pointerdown-drag and REMOVED on release. The pulse runs ONLY
  in the armed-but-not-moving window. The ~12-note soft cap bounds composited layers.

- **Interaction model (resolved).** The overlay is gesture-light; the CREATE flow lives in the
  Settings menu; deletion + edit live on the overlay.
  - **Quick tap** → the in-place **Edit/Delete** strip (a flat Metro strip — sharp, no floating
    modal). Delete is reversible-cheap → **no confirm**. "Edit" → on-glass keyboard.
  - **Long-press (~500ms hold)** → the note **arms for drag** (scales up + pulses). The NEXT press
    drags it; release writes the new x/y, disarms, stops the pulse. (Time-separated from tap → zero
    ambiguity.) An armed note auto-disarms after ~4s if no drag begins (no perpetual pulse).
  - A one-shot **click-swallow** after a drag stops the release from also firing the tap-edit.

- **Persistent-but-transient.** Notes survive reloads (stored) but **auto-fade after 12h** as a
  burn-in backstop (data persists; pixels don't). Overlay interaction counts as "touch," so
  screen-off + cascade freeze while it's in use.

- **Store — a single JSON file.** `/config/www/notes/notes.json` holds the array
  `[{id, x, y, text, paper, ink, font, look:{tilt,bowX,bowY,skew,creaseDir,creaseOp,folds}, created}]`.
  The overlay READS it fire-and-forget from `/local/notes/notes.json`. Writes go through a
  `shell_command` (`notes_write` → `notes_write.py`, stdlib, atomic .tmp→os.replace). Create / edit /
  delete / drag-end all funnel one full-array write, handed in as ONE base64'd arg (apostrophe /
  unicode safe — the notif_append.py precedent). (input_text is NOT usable — 255-char cap.)

- **Create UX — the Notes settings domino (Quick Settings → Sticky-notes).** The `homeSticky` face
  is **CREATE-ONLY** (no room in a 1×2 for create controls AND a delete-list). **Deletion + edit
  live on the dashboard** (tap a note → Edit/Delete strip). The menu makes notes; the overlay
  manages them. **(BUILT — Pass 2, on glass.)**

  **Create-menu layout (AS-BUILT, top → bottom):**
  1. **Paper + Ink** — one row, two side-by-side groups (4 paper + 4 ink swatches; selected outlined
     in cream). Paper hexes (LOCKED): yellow `#FFE99A` · green `#C8E6A0` · coral `#FFB3A0` · blue
     `#A8D8F0`. Ink (LOCKED): black `#222222` · blue `#1A3FB8` · red `#C0322B` · pencil-grey `#5A5A5A`.
  2. **Font** — a scrollable list split into **KIDS** (Lighthouse Treasure · Pastel Crayon · Lefty)
     and **ADULTS** (Daniel · Gabriel Weiss · Jimmy Script · Biro Script). Each row previews in its
     own typeface; selected row outlined.
  3. **Text field — renders LIVE in the selected font** ("Hello world!" placeholder; the field IS
     the preview).
  4. **Add note** → writes notes.json → the note appears at a cascade offset.

- **HACS stretch goal (separate later project).** Refactor the overlay into a standalone publishable
  custom Lovelace card after living with v1. Gate on the font licensing.

================================================================================
GUI-SPEC.md  — design (Home row points here)
================================================================================

## Sticky-note overlay (design)

A transparent display surface floating above the entire Wall (above banners). Not a tile. Owns no
grid cell, never reflows. Two halves: the OVERLAY (display + light gesture) and the MENU (create, in
Quick Settings).

- **The note (look).** Photorealistic CSS paper (>>>[OWNER]<<<'s prototype): warm paper colour, subtle
  grain, soft shadow, a crease, dog-ear folds, a slight per-note tilt/bow. Handwritten text in one
  of seven local fonts. The one deliberately tactile/skeuomorphic surface on an otherwise flat Wall
  — justified because it's literally imitating paper; the sharp-corner rule is waived for the paper
  only, everything else Metro holds. Note size ~130px (tuned on glass).

- **Placement.** Free drag-anywhere. New notes spawn at a small cascade offset. Soft cap ~12.

- **Gesture model (overlay is light; create lives in the menu).**
  - Quick tap → in-place Edit/Delete strip (flat, sharp, in-plane — NOT a floating modal).
  - Long-press → arms for drag (slight scale-up + gentle pulse); drag + release to place.
  - Editing text summons the on-glass keyboard.
  - Notes sit ABOVE banners; the overlay is pointer-transparent except on the notes.
  - **The Edit/Delete strip + keyboard CHROME is one layer-level element that positions itself by
    the active note and CLAMPS to the dashboard edges** — it flips above the note when there's no
    room below and slides sideways near a border, so the keyboard is never cut off. It renders flat
    (outside the note's 3D transform), crisp.

- **Motion discipline.** Only transform + opacity animate (drag, the pulse, fade-in/out). The costly
  look layers (blend/filter/noise/3D bow) are baked once and never animated. The pulse runs only
  while armed-not-moving.

- **The menu (Quick Settings → Sticky-notes, `homeSticky`) — BUILT.** Create-only. Layout
  top→bottom: paper + ink (4+4 swatches) → font (scrollable, split KIDS/ADULTS, each row in its own
  typeface) → text field rendering LIVE in the picked font → Add note. Metro-styled.

- **Burn-in.** Notes auto-fade after 12h. Overlay interaction freezes screen-off + cascade.

================================================================================
ENGINE-SPEC.md  — engine (the decomposition list points here)
================================================================================

## Sticky-note overlay (engine — BUILT, Phase D Passes 1 + 2)

A Wall-level overlay subsystem, the third overlay precedent after the banner layer and the per-tile
PTZ overlay-plane. Module **engine/notes.js** (`NotesMethods` mixin, stamped onto
TheWallCard.prototype in core.js — added to the `applyMixins(...)` call after InteractionMethods).
NO code in core beyond: the mixin apply, `this._buildStickyLayer()` in `_buildOnce` (after the
banner-layer build), and `this._notesStoreRead()` in `set hass` (after `_repaintBoardBack()`).

- **The overlay layer.** `_buildStickyLayer()` appends a `.sticky-layer` (inset:0, z-index 7,
  pointer-events:none) to the shadow root next to `.banner-layer`. Each note sets
  pointer-events:auto. Lives outside `.surface`, so it survives every reflow/flip/board-flip. The
  font loader is injected into this shadow root here (`window.WALL_NOTES_injectInto`).

- **Transform stack (the one that keeps drag + pulse + bow non-conflicting), outer → inner:**
  `.sticky-pos` (translate3d position = the dragged seam; perspective; opacity fade; pointer-events
  auto) > `.sticky-pulse` (scale ONLY — `.armed` = the pulse keyframe, `.lifted` = a static 1.06
  during drag) > `.sticky-container` (rotate(tilt) + the 3D bow, baked static, preserve-3d) >
  layers (shadow / paper+crease / ink / lighting / 4 folds). Keeping translate (drag), scale
  (pulse), and the baked tilt/bow on SEPARATE elements is what lets each animate without clobbering
  the others.

- **Store read.** `_notesStoreRead()` (from set hass, throttled ~4s; forced once ~450ms after our
  own write) fetches `/local/notes/notes.json` fire-and-forget. `_ingestNotes` applies the 12h
  render-time fade filter + a signature compare, then `_renderNotes` diffs by id (build + fade-in
  new, fade-out + drop gone, reposition moved unless that note is mid-drag). The overlay never
  writes except through the dispatch phone.

- **Note render + bake.** `_buildNoteEl` builds the prototype DOM; `_bakeNoteLook` writes the look
  params as inline CSS custom props (`--paper`, `--ink-color`, `--tilt`, `--bow-x/y`, `--crease-*`,
  `--f-*` folds) ONCE; `_bakeNoteInner` sets the ink text + the font-family (key → family via
  `window.WALL_NOTES_FONTS`). No re-render on hass pushes (notes aren't entity-backed).

- **Gestures (engine-owned own-pointer, NOT the tap contract).** A pointerdown handler per note:
  pointerdown → 500ms long-press timer + record origin (no capture yet); quick still tap → toggle
  the Edit/Delete strip; long-press → ARM (`.armed` pulse, bounded 4s auto-disarm); a SECOND
  pointerdown on the armed note → drag (setPointerCapture, translate3d follow, release writes x/y +
  one-shot click-swallow). A move past `dragThreshPx` before the timer cancels the arm.

- **Chrome (Edit/Delete strip + keyboard) — ONE layer-level element, edge-clamped.** Not nested in
  the note. `_renderChrome()` builds the strip (or the keyboard when editing) into a single
  `.sticky-chrome` child of the layer and positions it in LAYER coordinates from the active note's
  x/y + size: prefer below the note, flip ABOVE when there's no room, clamp horizontally to the
  layer — so the keyboard never spills off a border (the Pass-1 fix). Routes via `data-sn-action`
  through the layer's own delegated click listener (the notes live outside `.tile`, so the per-tile
  click listener never sees them — the board-back precedent).

- **Edit text → on-glass keyboard.** The FloatingKeyboard markup is inlined in notes.js (`_noteEditKbHtml`,
  ABC/123 two-mode) rather than imported (the overlay is outside the face module graph); keys route
  via `data-sn-action="kb"` to `_noteKbKey`; Done commits + writes.

- **Auto-fade.** Render-time filter (12h) + lazy prune-on-write — an expired note isn't rendered, and
  the next write grooms the file (a note with no `created` is treated as fresh). notes_write.py also
  soft-caps at 12 newest-by-created.

- **Fonts loader.** `notes-fonts.js` (the lato.js / mdi-font.js pattern) is the SINGLE SOURCE for the
  font set: it holds the FONTS table `[{key,label,family,file,fmt,group}]`, builds the @font-face
  block, injects it globally + into the card's shadow DOM (`window.WALL_NOTES_injectInto`), and
  publishes `window.WALL_NOTES_FONTS` for notes.js (renderer) + faces/home.js (the Pass-2 picker) to
  read at runtime. (DEVIATION from the original plan: the FONTS table lives HERE, not in tokens.js —
  better single-sourcing given the loader is a global script, not in the module graph.)
  **woff2 DEVIATION:** the fonts are served as their source **TTF/OTF**, not woff2. woff2 needs
  brotli (a C build) which isn't on the box, and the only paths to get sandbox-converted woff2 onto
  the box are the banned base64-file transport or a risky on-box pip build. On a LAN-served single
  panel the size delta (~300KB total) loads once with zero frame-budget impact. woff2 is logged for
  the HACS-publish stretch where bandwidth matters.

- **TOKENS.** `engine/tokens.js` → `notes:{ longPressMs:500, dragThreshPx:8, pulseScale:1.05,
  armWindowMs:4000, fadeHours:12, softCap:12, cascadeOffsetPx:24, readThrottleMs:4000 }`. notes.js
  reads these with inline fallbacks (works even before the block exists). The FONTS table is NOT in
  tokens.js — see Fonts loader above.

- **Files (Pass 1, BUILT):** NEW `engine/notes.js` · NEW `www/notes-fonts.js` (loader) · NEW
  `notes_write.py` (/config) · NEW `www/notes/notes.json` (runtime data) · 7 fonts in `www/fonts/` ·
  `engine/styles.js` (the `.sticky-*` CSS, adapted from the prototype, bake-once) · `engine/tokens.js`
  (the notes block) · `engine/core.js` (mixin apply + `_buildStickyLayer` + `_notesStoreRead`) ·
  `configuration.yaml` (`/local/notes-fonts.js` in extra_module_url + `shell_command.notes_write`).
  **Pass 2 (BUILT):** `faces/home.js` (the real Create menu + `stickyTap`/`snKey` handlers + the
    `homeSticky` onTap registration + the `_sn` draft state & locked 4+4 hex tables) · `engine/notes.js`
    (`_notesAdd` + `_freshNoteLook` — the overlay's create seam) · `engine/interactions.js` (the
    `ctx.addNote` face-tap seam) · `engine/styles.js` (the `.sn-*` menu CSS, replacing the dead
    `.hs-*` stub). The menu does NOT write the store — it hands choices to the overlay via
    `ctx.addNote`; the overlay stays the single writer (see the BUILD STATUS deviation note).

================================================================================
TILES-SPEC.md  — wiring (Home → Sticky-notes points here)
================================================================================

## Sticky Notes (overlay — wiring)

Not a tile (no integration, no entity). The overlay reads a local JSON; the menu writes it.

- **Store:** `/config/www/notes/notes.json` (served `/local/notes/notes.json`). Written by
  `shell_command.notes_write` → `notes_write.py` (stdlib, atomic, base64 full-array arg). The
  shell_command + the font loader live in **configuration.yaml** (the `shell_command` block +
  `extra_module_url`), NOT a packages/notes.yaml (avoids package-merge risk; configuration.yaml was
  being edited anyway). Runtime data → `www/notes/` should be in .gitignore (the overlay treats a
  missing file as an empty board).
- **Menu (Quick Settings → Sticky-notes, `homeSticky`) — BUILT:** CREATE-ONLY — paper + ink (4+4) →
  font (KIDS/ADULTS split, live-preview text field on the `FloatingKeyboard` ABC/123) → Add. Hands the
  choices to the overlay via `ctx.addNote(rec)`; the OVERLAY does the single `notes_write` (NOT the
  menu — see the BUILD STATUS deviation). Deletion + edit are on-note (dashboard tap), not in the menu.
- **Fonts:** 7 TTF/OTF in `/www/fonts/` (`sn-*.ttf/.otf`) + a `notes-fonts.js` loader (extra_module_url).
  Source files in `/config/sticky_notes/sn_fonts/`. License = confirm per-font embedding is allowed
  (several dafont fonts are personal-use-only; private family build is fine, log for the HACS stretch).
- **Manifest rows (family-dashboard-spec file manifest):** engine/notes.js, notes-fonts.js,
  notes_write.py, www/notes/notes.json (gitignored), the 7 `sn-*` font files; the shell_command +
  font loader are configuration.yaml lines (no packages/notes.yaml).

================================================================================
OPEN ITEMS
================================================================================
1. RESOLVED — Auto-fade = **12h, render-time filter + lazy prune-on-write** (a missing `created`
   treats the note as fresh).
2. OPEN — Font license check per font before any HACS publish (private family build is fine; HACS
   stretch gate).
3. RESOLVED — notes_write.py takes the full array as ONE **base64'd arg** (apostrophe/unicode/quote
   safe), well under any shell length limit for ~12 notes.
4. RESOLVED — Paper/ink are a curated **4 + 4** set, hexes LOCKED (paper yellow `#FFE99A` / green
   `#C8E6A0` / coral `#FFB3A0` / blue `#A8D8F0`; ink black `#222222` / blue `#1A3FB8` / red
   `#C0322B` / pencil-grey `#5A5A5A`).
5. OPEN — Confirm the prototype's blend-mode/filter cost on the real panel under a full dozen notes;
   if it ever janks a camera feed, drop `mix-blend-mode` on the lighting layer first (priciest).
6. OPEN (refinements, tune-on-glass) — note size (currently 130px), the chrome scrim/accent colours
   (currently hardcoded dark; could tokenise), and any look-param range tweaks once the Pass-2 menu
   drives variety.
