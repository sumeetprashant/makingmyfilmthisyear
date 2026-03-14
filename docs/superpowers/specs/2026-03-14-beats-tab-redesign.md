# PageZero Beats Tab Redesign

**Date:** 2026-03-14
**File:** `film/screenplay-editor.html` (single-file HTML app, ~3180 lines)
**Scope:** Beats tab primarily. Dark mode applies globally (all tabs share CSS variables). No functional changes to Script or Music & Mood tabs.

## Summary

Redesign the Beats tab from a single-column scrolling list into a full-width hybrid grid with scene images, drag-to-reorder, dark mode, and bigger cards. All changes are CSS/HTML/JS within the existing single-file architecture.

## Decisions

| Decision | Choice |
|----------|--------|
| Layout | Hybrid — grid within movements, full browser width |
| Images | Toggle between Storyboard (full bg) and Index Card (thumbnail) views |
| Image upload | Both clipboard paste (Ctrl+V) and file picker button |
| Drag behavior | Auto-reassign movement on cross-movement drop |
| Dark mode | CSS variables, gold accent, localStorage toggle |
| Card sizing | ~30% larger fonts, generous padding |

## 1. Layout: Hybrid Grid

### Current
- `max-width: 860px` centered container
- Cards stacked vertically, one per row
- Movement headers as flex rows

### New
- Remove max-width — full browser width with `padding: 24px 32px`
- Movement headers span full width: circle number + title + scene count right-aligned (format: "N scenes", shows filtered count when filter active)
- Cards in CSS grid per movement: `grid-template-columns: repeat(auto-fill, minmax(320px, 1fr))`
- Responsive: ~3 cols on wide screens, 2 on tablet, 1 on phone
- Gap: 14px between cards

### Why `minmax(320px, 1fr)` instead of fixed columns
Cards auto-size to fill available width. On a 1920px screen you get 4-5 cards per row. On 1280px you get 3. No media queries needed for the grid itself.

## 2. Scene Images

### Data model addition
Add `image` field to beat entry: `string | null` — base64 data URL or null.

### Two view modes (toolbar toggle)
- **Storyboard View**: image fills card background via `background-image`. Text overlaid with bottom gradient (covers bottom 60% of card, `rgba(0,0,0,0.92)` → `transparent`). Text also gets `text-shadow: 0 1px 3px rgba(0,0,0,0.8)` for legibility over any image. Cards without images show solid background.
- **Index Card View**: 120px thumbnail on left (flex child), text on right. Cards without images show text-only (no empty thumbnail space).

### Adding images
- **Clipboard paste**: listen for `paste` event on the card's outer div (not on textarea children — textarea paste remains text-only). When paste contains image data, intercept and process. When paste contains text, let it through to textarea.
- **File picker**: "Add Image" button visible on card hover (in beat-actions row). Opens hidden `<input type="file" accept="image/*">`. Reads via FileReader as data URL.
- **Remove image**: "Remove Image" button in the Edit form (slide-in panel), shown only when image exists.
- **Edit form image section**: At the top of the form, show a small thumbnail preview (max 100px tall) of the current image if one exists, with "Remove Image" and "Replace Image" buttons below it. If no image, show an "Add Image" file picker button.

### Storage
- Base64 data URL stored in `beatEntries[].image`
- Persisted in autosave (localStorage) and .project.json exports
- Compress/resize on paste to max 800px wide, JPEG quality 0.7 using canvas
- Max image size after compression: 150KB. If larger, warn user and reject.
- On `QuotaExceededError` from localStorage: show toast warning "Storage full — images may not save. Export your project to keep data safe." Autosave continues without images (strips `image` fields from the save payload, keeps them in memory).

## 3. Drag to Reorder

### Implementation
- HTML5 Drag and Drop API (no library — keeps single-file architecture clean)
- Drag handle: 6-dot grip icon on left edge of card, `cursor: grab`
- `draggable="true"` on card element
- Events: `dragstart` (set data, add ghost class), `dragover` (show insertion line), `drop` (reorder array), `dragend` (cleanup)

### Cross-movement drops
- Each movement section (header + grid) is a drop zone
- Dropping a card under a different movement header updates `entry.movement` and `entry.movementTitle`
- Scene IDs remain unchanged (user controls numbering manually)
- After drop: splice entry from old position, insert at new position in `beatEntries` array, re-render list, trigger autosave
- Empty movements (last card dragged out): movement header disappears from render. Movement still exists if other cards reference it.

### Insertion indicator
- Horizontal line (2px, accent color) between cards at drop position
- Calculated via `dragover` event: compare mouse Y to each card's bounding rect midpoint to find insertion index
- Use browser default drag ghost (semi-transparent snapshot of card). No custom ghost needed.

### Known limitation
- No keyboard-based reordering. Drag is mouse-only. Acceptable for a solo-user tool.

## 4. Dark Mode

### CSS variable approach
```
:root (light) — existing colors
:root.dark — dark overrides
```

Key dark mode variables:
- `--bg`: `#0d0d0d`
- `--paper`: `#1c1916` (dark warm card)
- `--ink`: `#e0d8cc`
- `--accent`: `#d4a853` (unchanged)
- `--card-border`: `#2a2520`
- `--card-hover-glow`: `rgba(212,168,83,0.08)`

### Toggle
- Sun/moon icon button in toolbar (next to existing buttons)
- Toggles `dark` class on `<html>` element
- Saves preference to `localStorage.setItem('pagezero-theme', 'dark'|'light')`
- Loads preference on init, defaults to light

### Neon glow on hover (dark mode only)
```css
:root.dark .beat-card:hover {
  box-shadow: 0 0 1px rgba(212,168,83,0.3),
              0 0 8px rgba(212,168,83,0.08),
              0 4px 20px rgba(0,0,0,0.4);
}
```

## 5. Card Size Fix

### Current sizes (too small)
- Title: 14px
- Description: 12px
- Scene ID: 11px
- Tags: 10px
- Card padding: 16px 20px

### New sizes
- Title: **16px** bold
- Description: **13px**
- Scene ID: **12px** monospace bold
- Tags: **10px** (unchanged — these are labels, small is fine)
- Status pill: **10px**
- Notes textarea: **13px**
- Card padding: **18px 20px**
- Card min-height: **none** (let content determine height — with bigger fonts cards will naturally be taller)

## 6. View Toggle in Toolbar

Add to `#toolbar-beats`:
- View toggle: two buttons `Storyboard | Index Card` (styled like tab pills)
- Dark mode toggle: sun/moon icon button (in main toolbar, not beats-specific — applies globally)
- Both persist to localStorage

## 7. Autosave Compatibility

- Bump `AUTOSAVE_KEY` to `'pagezero-autosave-v5'` (new image field + view preferences)
- **Migration**: on init, check for `pagezero-autosave-v4`. If found, load it, add `image: null` to each beat entry, save as v5, delete v4 key. One-time migration preserves existing data.
- Project format stays v3 (image field is additive, backward compatible for export/import)
- New localStorage keys: `pagezero-theme`, `pagezero-beat-view`

## Out of Scope

- Changes to Script or Music & Mood tabs
- Theming beyond light/dark (future work — CSS vars make it easy)
- Image generation (user generates externally, pastes in)
- Multi-select drag (single card drag only)
- Undo/redo for drag operations
