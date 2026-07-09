# Plan: Setlists + MusicXML Import Pipeline

*Two features, one direction: Interleaves as a practice app that grows a score-reader arm — never the reverse. The reader exists to feed the practice engine.*

---

## Part 1 — Setlists

### Goal
Let the user scope the app to a named collection of Scores — "open different setlists of scores to make passages around them." Doubles as practice scope and (later) performance set.

### Schema

```js
Setlist {
  id, name, createdAt, order,
  scoreIds: []        // references only — a Score can live in many setlists
}
```

- New store `setlists` in `interleaves_v2` (pre-release: clean-slate, no migration).
- **Passages are never listed in a setlist** — they follow their source Score implicitly via `sourceId`. One membership list, no sync problem.
- Active setlist id → `localStorage['active_setlist']`. `null` = **Everything** (virtual, not a DB row — cannot be renamed or deleted).

### UI flow

1. **Picker** at the top of the sidebar, above the Scores section: current setlist name + ▾. Tapping opens a simple list: Everything, each setlist, "＋ New setlist…", "Edit setlists…".
2. **Scoping**: with a setlist active, the Scores section shows only member Scores; the Passages section shows only passages whose source is a member. Everything else in the app is untouched — the sidebar *is* the scope.
3. **Membership editing** — two paths:
   - "Edit setlists…" modal: pick a setlist, check/uncheck Scores (checkbox list, 44pt rows).
   - Per-score shortcut: a "⊕ setlist" item next to rename/delete on the score row → checklist popover of setlists.
4. **Practice**: `buildPracticePool()` gains one filter — passages and in-rotation scores whose Score is in the active setlist. `go-btn` disabled state computed per scope.
5. **Import**: new Scores land in Everything always; if a setlist is active at import time, also auto-add to it (importing "into" the open setlist — least surprising).

### Rules

| Case | Behavior |
|---|---|
| Delete a Score | Remove its id from every setlist |
| Delete a setlist | Scores untouched; if it was active, fall back to Everything |
| Score in zero setlists | Fine — always visible under Everything |
| Export/import | `setlists` store included; export `version` 3 → 4; importing a v3 file just has no setlists |

### Explicitly out of scope (this pass)
- Performance/gig mode (ordered play-through of a setlist, no shuffle, no timer) — the schema supports it later via existing `order` + manual-order machinery; don't build now.
- Setlist-level settings (timer length, rest cadence per setlist) — revisit only if real use demands it.

---

## Part 2 — MusicXML import ("XML in, Score out")

### Goal
Open a lead sheet (`.xml` / `.musicxml` / `.mxl`), choose **clef, key/transposition, and size**, then **Save** — which bakes the rendered result to fixed raster pages as a completely normal Score. Downstream (passages, annotations, Compás, practice, export) is untouched and unaware.

### Core principle
**Rasterize at import — MusicXML is an import pipeline, not a new document type.** The whole object model (pageId-anchored passages, pixel-coordinate annotation layers, two-canvas rendering) assumes fixed pages. Reflowable notation would break all of it; baked pages break nothing. This is the same posture as PDF.js: a converter in front of the same Score object.

### Flow

1. Import window accepts the new extensions. `.mxl` is a zip → unzip in-browser (fflate, ~8 KB).
2. Instead of committing straight to a Score, open a **Render Options screen** — full-screen staged editor in the Score Edit mold (deliberate place you enter and leave; nothing persists until Save; Cancel discards):
   - **Clef**: treble · treble-8vb (guitar) · bass
   - **Key / transpose**: semitone stepper ±11 with live key name ("C → A, sounding down a minor 3rd")
   - **Part picker** — only shown if the file has multiple parts; lead sheets are usually one
   - **Size**: render scale (roughly small / medium / large staff)
   - Live preview pane re-renders on each change
3. **Save** → render each page via OSMD to canvas at import resolution (reuse `MAX_PX`/`IMG_Q`) → pages become ordinary `{pageId, src, annotation:null}` → normal Score, auto-named from the XML `<work-title>`.

### Keep the source XML

```js
Score {
  ...existing fields,
  sourceXml   // string | null — present only for XML-born Scores
}
```

- Enables **"♪ Re-render"** in Score Edit (button visible only when `sourceXml` exists): reopen Render Options, change key/clef, Save replaces the pages.
- **Safety rule, same philosophy as split-PDF**: re-render **replaces pages only if the Score has no passages**. If passages exist, re-rendering changes page geometry under their pixel annotations — so Save creates a **new Score** instead ("Milonga de mis amores (in A)"), original untouched. No remapping, no corruption path.
- Cost: XML text is small (tens of KB); negligible next to page images.

### Rendering engine
- **OpenSheetMusicDisplay** (BSD, VexFlow-based) via CDN, loaded lazily on first XML import — same pattern as pdf.js. Verovio (WASM) is the fallback candidate if OSMD's lead-sheet output (chord symbols!) disappoints — evaluate chord-symbol rendering early, it's the make-or-break for lead sheets.
- Offline/PWA note: once the Capacitor wrap happens, bundle the library locally rather than CDN.

### What this is *not*
- Not live reflow, not playback, not a notation editor, not forScore parity. The moment the Save button is hit, it's just sheet music in the practice engine — which is the product.

---

## Sequencing

1. **Setlists** — small, immediately useful (students, tango/choro sets), schema is free pre-release.
2. **MusicXML** — build when real XML files show up; the rasterize-at-import decision above means deferring costs nothing.

## Open questions
- Setlist picker: dropdown vs. horizontal chips (Boox: chips = fewer nested taps, more topbar space used).
- Import-while-setlist-active auto-add: confirm this feels right in use, or make it a toggle in the import window.
- OSMD chord-symbol + slash-notation quality on real choro/tango lead sheets — test with actual files before committing.
- Does Re-render need transposition presets per instrument (7-string, Bb, Eb) or is the stepper enough?

## Dependencies
- fflate (mxl unzip), OSMD (render) — both lazy-loaded, no framework impact
- Export format version bump 3 → 4
- No changes to Passage schema, annotation model, or practice machinery in either part
