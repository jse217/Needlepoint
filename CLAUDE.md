# StitchCraft Studio — Project Context

## What This Is
A single-file needlepoint belt pattern generator. The user uploads a photo, it gets quantized into a DMC thread color palette and rendered as a stitch grid. Patterns can be edited, saved to a library, and assembled into a belt layout.

**File:** `/Users/jared_eisenberg/Developer/Needlepoint/index.html`
**Open in browser:** `file:///Users/jared_eisenberg/Developer/Needlepoint/index.html`

---

## Tech Stack
- **React 18** via UMD CDN (no build step)
- **Babel Standalone** for JSX transpilation in-browser
- **Tailwind CSS** via CDN (used sparingly — most UI now uses inline styles)
- **Inter font** from Google Fonts
- All logic, components, and styles live in one `index.html` file

---

## Component Map (line numbers approximate)

| Component | Line | Purpose |
|---|---|---|
| `ICONS` / `Icon` | ~48 | Inline Lucide SVG icon system |
| Color utilities | ~102 | `hexToRgb`, `rgbToHex`, `colorDistance` |
| `DMC_COLORS` | ~116 | Full DMC thread database (~200 colors) |
| `findClosestDMC` | ~200 | Nearest DMC color matcher |
| `kMeansQuantize` | ~211 | K-means color quantization |
| `applySmoothing` | ~237 | Confetti stitch removal |
| `mergePaletteToCount` | ~261 | Agglomerative palette merge |
| `imageToPattern` | ~301 | Master: image → stitch grid |
| `removeBackground` | ~373 | Flood-fill background removal |
| `CropModal` | ~412 | Image crop UI (drag handles) |
| `ZoomPreview` / `ZoomCanvas` | ~651 | Multi-scale viewing distance preview |
| `PatternCanvas` | ~713 | Main editing canvas (all mouse interaction) |
| `ColorSwatch` | ~994 | Palette row with fixed-position dropdown |
| `BeltPreview` | ~1110 | Cross-stitch belt canvas renderer |
| `NeedlepointApp` | ~1209 | Root app, all state, all tabs |

---

## Pattern Data Model
```js
pattern = {
  grid: number[],      // flat array, width*height. -1 = blank/transparent
  width: number,
  height: number,
  palette: [{ id: number, hex: string, rgb: [r,g,b] }]
}
```
Grid values are indices into `palette[]`. `-1` means transparent (shows `canvasBg`).

---

## App State (NeedlepointApp)

```js
// Pattern
pattern, setPattern           // current stitch grid
generating                    // spinner flag
undoStack / redoStack         // undo history (pushUndo called before mutations)

// Editing modes
selectedColor                 // palette id currently selected for painting (null = none)
stitchSelectMode              // bool: select/delete mode vs paint mode
selectedCells                 // Set<number> of selected cell indices
cropMode                      // bool: crop resize mode
cropBounds                    // { x1, y1, x2, y2 } in cell coords (null when inactive)
zoomOut                       // bool: preview mode vs edit mode
patternZoom                   // float 0.5–4.0

// Canvas
canvasBg                      // hex string, belt background color (default "#D4C5A9" linen)

// Library (persisted to localStorage 'sc_library')
library                       // [{ id, name, pattern, thumb, date }]
saveName

// Belt designer (persisted to localStorage)
beltSlots                     // [{ id, pattern, thumb, name }]
beltBg, beltSpacing, beltLength
savedBelts                    // [{ id, name, slots, bg, spacing, length, date }]

// Tabs
tab                           // "create" | "library" | "belt"
```

---

## PatternCanvas — Key Details

### Props
```js
{ grid, width, height, palette,
  selectedColor, onCellPaint, onPaintEnd, onColorPick,
  stitchSelectMode, selectedCells, onCellSelect,
  zoom, canvasBg,
  cropMode, cropBounds, onCropBoundsChange }
```

### Canvas Rendering
- `CELL = Math.max(3, Math.min(24, Math.floor(Math.min(900/width, 700/height)))) * zoom`
- Canvas fills with `canvasBg` first (entire background)
- Blank cells (`ci === -1`): draw nothing — `canvasBg` shows through
- Colored cells: `fillRect` full cell first (prevents corner bleed), then `arc` circle on top for stitch-painted look
- Dimmed colors (when a color is selected): `globalAlpha = 0.2` for non-selected colors

### Two canvas layers
- `canvasRef` — main stitch drawing
- `overlayRef` — absolute positioned, pointer-events none — used for:
  - Stitch select drag rect (indigo)
  - Crop overlay (green, dims outside area + white circle handles)

### Mouse interaction refs (avoid re-renders)
```js
isDrawing.current          // mousedown is held
isDraggingRect.current     // crossed CELL threshold → rect mode
dragStart.current          // { x, y, px, py } of mousedown
dragCurrent.current        // { x, y, px, py } of current mouse
dragStartIsBlank.current   // bool: did drag start on a blank cell?
dragStartColor.current     // color index where drag started (-1 for blank)
cropHandleDrag.current     // { handle, startPx, startPy, startBounds } for crop
```

### Three interaction modes in PatternCanvas
1. **Paint mode** (`!stitchSelectMode && !cropMode`): mousedown picks color, drag paints
2. **Select mode** (`stitchSelectMode`): click toggles cell, drag = brush or rect select
   - Selection constrained to same type as drag start: blank-only or same-color-only
   - `dragStartColor.current` used to prevent selecting red cells when clicking gray cells
3. **Crop mode** (`cropMode`): hover near handles shows resize cursor, drag handle moves crop bounds

---

## Selection Logic (Critical Bug History)

**The problem:** drag-selecting blank cells was also selecting red filled cells.

**The fix (partially in place, `dragStartColor` ref added but handlers not yet updated):**

In `handleMouseDown`:
```js
dragStartIsBlank.current = grid[y * width + x] === -1;
dragStartColor.current = grid[y * width + x]; // ← need to set this
```

In `handleMouseMove` (brush path) and `handleMouseUp` (rect path), filter logic should be:
```js
// For blank: match all blank cells
// For colored: match ONLY cells with the same color as drag start
const isBlank = grid[...] === -1;
const sameType = isBlank === dragStartIsBlank.current;
const sameColor = dragStartIsBlank.current || grid[...] === dragStartColor.current;
if (sameType && sameColor) // include this cell
```

⚠️ **`dragStartColor` ref was added but the filter logic in handleMouseDown, handleMouseMove, and handleMouseUp still needs to be updated to use it.**

---

## Crop Mode

- Enter: `toggleCropMode()` — initializes `cropBounds` to full pattern, disables other modes
- Exit: `toggleCropMode()` (cancel) or `applyCrop()` (confirm)
- Handles: 8 points (4 corners + 4 edge midpoints), detected by pixel proximity (`HANDLE_HIT` threshold)
- Dragging handles outward creates expansion (blank cells added)
- `applyCrop()` handles both trim AND expand — out-of-bounds cells become `-1`

---

## Belt Preview (BeltPreview component)

- Renders cross-stitches as two diagonal strokes (\ and /) per cell
- Back stitch (`/`): slightly darker shade of thread color
- Front stitch (`\`): full thread color on top
- Canvas mesh grid lines between cells
- `CELL = compact ? 4 : 9` pixels per stitch
- Belt is always `BELT_H = 21` rows tall
- Icons centered vertically, placed left-to-right with `spacing` gaps

---

## Design System

All UI uses **inline styles** with these tokens:
```
Background:    #f5f5f4
Surface:       #ffffff
Surface-2:     #f4f4f3
Border:        #e5e5e3
Border-strong: #d4d4d2
Text-primary:  #1a1a18
Text-secondary:#6b6b68
Text-muted:    #a0a09d
Accent:        #4f46e5 (indigo)
Accent-light:  #f5f4ff
```

CSS utility classes defined in `<style>`:
- `.btn`, `.btn-primary`, `.btn-secondary`, `.btn-ghost`, `.btn-danger-ghost`
- `.btn-sm`, `.btn-icon`
- `.panel` (white card with border + border-radius)
- `.section-label` (small-caps section header)
- `.usage-bar` / `.usage-bar-fill` (palette usage bar)

---

## ColorSwatch Dropdown

Uses `position: fixed` with JS-calculated coords (from button's `getBoundingClientRect`) to escape any parent `overflow: hidden`. This was necessary because the palette panel clips absolute-positioned children.

---

## localStorage Keys
| Key | Contents |
|---|---|
| `sc_library` | Saved pattern library `[{ id, name, pattern, thumb, date }]` |
| `sc_belts` | Saved belt designs |
| `sc_beltBg` | Last used belt background color |
| `sc_beltSpacing` | Last used icon spacing |
| `sc_beltLength` | Last used belt length |

---

## Known Pending Issues / Next Steps

1. **`dragStartColor` filter not fully wired** — ref is added in PatternCanvas, but `handleMouseDown`, `handleMouseMove` (brush), and `handleMouseUp` (rect) still need to use it to constrain selection to same-color cells. Without this, selecting a gray stitch also grabs red stitches in the same drag rect.

2. **Canvas background** — blank cells correctly show `canvasBg` now. The canvas element CSS `background` is also set to `canvasBg` as a fallback.

3. **Possible UX improvements the user may want:**
   - Better way to select individual stitches without accidentally grabbing neighbors
   - Auto-trim blank border rows/columns (smart crop)
   - Stitch count summary by color for purchasing thread
   - More export options (e.g. PNG with grid lines for printing)

---

## How to Edit

1. Open `index.html` in any text editor — everything is in one file
2. Refresh the browser to see changes
3. Library data persists in `localStorage` — safe across refreshes
4. Use browser DevTools console to debug; errors will appear there

To open: `open /Users/jared_eisenberg/Developer/Needlepoint/index.html`
