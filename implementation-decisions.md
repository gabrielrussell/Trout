# Implementation Decisions & Answers to Review Questions

**Date:** 2025-11-14
**Status:** Consolidated answers from design review

This document captures all architectural decisions and answers to implementation questions raised during the specification review.

---

## Core Architecture Decisions

### Grid & Canvas System

**Hexagonal Canvas:**
- Fixed size per asset: 2^6 (64) grid units wide
- Never changes once set
- Hexagonal boundary aligned to triangular grid

**Triangular Grid:**
- Regular triangular tessellation
- Origin at center (0, 0)
- Coordinates: Abstract integer grid units
- Always visible, always-on snapping

**Grid Scaling:**
- User selects snapping precision: 2^0, 2^1, 2^2, ... 2^6
- This is snapping level, not canvas size
- Finer levels (2^0) = precise detail, coarser levels (2^6) = large features

**Grid Mathematics:**
- 1D: One 2^n trixel base = 2^n × 2^0 trixel bases
- 2D: One 2^n trixel = 4^n × 2^0 trixels (area relationship)
- Hexagon composition: Regular hexagon = 6 equilateral triangles meeting at center
- If hexagon width = 2^n, each triangle side length = 2^(n-1)
- Total 2^0 trixels in hexagon: 6 × (2^(n-1))² = 6 × 2^(2n-2)

**Rendering to Pixels:**
- User specifies output size in pixels (e.g., "128 pixels wide")
- System calculates: pixels-per-grid-unit = output_size / 64
- All trixels scale proportionally
- **Important:** Only one edge of triangular trixels aligns with square pixel grid
- Other two angled edges require anti-aliasing/coverage calculation

---

## Feature #1: Core Symmetry System

**Data Model:**
- Store ONLY right half of geometry (un-mirrored)
- Left side computed on-the-fly by mirroring across x=0

**Symmetry Enforcement:**
- Fixed vertical center line at x=0
- Cannot be moved or rotated
- Bilateral symmetry only (no radial, no multi-axis)

**Asymmetry:**
- Geometry ALWAYS symmetric
- Asymmetry achieved through COLOR only (L/R palette variation)

**Editing:**
- Left side NOT editable
- Left side NOT selectable
- Only edit right side (stored geometry)

**Copy/Paste:**
- No copy/paste functionality planned

**Selection Model:**
- Only select points on the right side (actual stored geometry)
- Mirrored points are visual only

---

## Feature #2: Triangular Grid Snapping System

**Grid Topology:**
- Regular triangular tessellation
- Alternating triangle orientations (standard pattern)

**Grid Origin:**
- Center of canvas at (0, 0)

**Coordinate System:**
- Abstract integer grid units
- Size-agnostic (no pixel dimensions in working file)

**Grid Visibility:**
- Always visible (no toggle)

**Snapping:**
- Always on (no disable)
- Power-of-2 grid level selector (2^0 through 2^6)

**Zoom:**
- No zoom planned currently

**Grid Scaling UI:**
- Selector/dropdown for power-of-2 levels
- Maybe: 1× (2^0), 2× (2^1), 4× (2^2), 8× (2^3), 16× (2^4), 32× (2^5), 64× (2^6)

---

## Feature #3: Triangular Pixel Rendering

**Tessellation:**
- Same as grid tessellation (alternating triangles)

**Alignment:**
- Tri-pixels ARE the grid
- No separate "trixel display size"

**Rendering:**
- User draws on grid at various levels
- At export, entire canvas scales to fit output pixel size
- Example: 64 grid units → 128 pixels = 2 pixels per grid unit

**Rasterization:**
- NOT option A (fill all touched trixels)
- Between option B (center point) or C (anti-aliasing)
- **DECISION DEFERRED** - will decide based on aesthetic testing

**Stroke Rendering:**
- Stroke = all 2^0 trixels along path border
- May be configurable

**Color per Trixel:**
- Each trixel gets solid color (no gradients within trixel)
- May be configurable

**Preview:**
- Editor shows wireframe only (no rendering)
- Preview pane shows fully rendered output using rendering library

**Anti-aliasing Challenge:**
- Triangle edges don't align with square pixels (except base)
- Requires coverage calculation for angled edges

---

## Feature #4: Faux 3D Lighting

**Lighting Model:**
- Simulated 3D (not just simple gradient)
- Considers shape orientation/normals

**Light Source:**
- Infinitely far (parallel rays, like sunlight)
- Fixed direction in world space

**Rotation Generation:**
- 6-way minimum (every 60°), 12-way option (every 30°)
- Rotate the GEOMETRY (character facing directions)
- Light source stays FIXED in world
- NOT rotating light around static asset

**Gradient Application:**
- Uniform gradient direction for all shapes (option A)
- Top-to-bottom or similar, based on light direction

**Multi-shape Lighting:**
- Z-order affects lighting
- Front shapes can cast shadows on back shapes

**Color Interaction:**
- Blend mode (lighten and darken)
- Not just multiply

**Export:**
- Rotation variants exported as sprite sheet frames

---

## Feature #5: Component/Layer Assembly System

**Component Structure:**
- Unit tree (recursive hierarchy)
- Each unit contains:
  - One shape (set of paths, possibly empty)
  - Zero or more child units
  - Palette (left and right, 12 slots each)

**Component Storage:**
- All units stored in .trout file
- Unit tree is the native data structure

**Component Library:**
- Components are just units in the tree
- Any unit can be reused
- No special "library component" designation

**Positioning:**
- Fixed: All centers aligned
- No translation, rotation, scale, or transforms

**Z-ordering:**
- List position determines draw order
- Later items draw on top (obscure earlier items)

**Component Updates:**
- Automatic (units are nodes in tree, edit propagates)

**Component Variants:**
- Create separate units (e.g., shirt-short, shirt-long)
- No variant system within components

**Export:**
- Can export component metadata (which units used)
- Metadata export feature #11

**Circular Reference Prevention:**
- **CRITICAL:** Validate that Unit A cannot contain Unit B if Unit B contains Unit A (directly or through chain)
- Must prevent circular references in unit tree

---

## Feature #6: Dynamic Palette Inheritance

**Palette Structure:**
- Each unit has TWO 12-slot palettes: LEFT and RIGHT
- Both palettes treated equally (neither is "primary")

**Color Lookup Algorithm:**
```
To get color for LEFT side, slot N:
  1. Walk up tree checking LEFT palette slot N
  2. If not found, walk up tree checking RIGHT palette slot N

To get color for RIGHT side, slot N:
  1. Walk up tree checking RIGHT palette slot N
  2. If not found, walk up tree checking LEFT palette slot N
```

**Global Palette Requirement:**
- For each slot (1-12): At least ONE side must be filled (left OR right OR both)
- Ensures all palette references eventually resolve

**Slot Count:**
- 12 slots per palette (24 total colors possible: 12 left + 12 right)
- Symmetric shapes: Use 12 colors (left = right)
- Asymmetric color: Up to 24 colors (different on each side)

**Slot Semantics:**
- Numbered (1-12), not named/semantic

**Color Assignment:**
- Before drawing: Select palette slot (1-12)
- Path references that slot number (not RGB value)
- Can edit palette slot color → all paths using that slot update

**Editor Workflow:**
- Editor pane: Shows black wireframe (no colors)
- Preview pane: Shows fully rendered with resolved colors
- No "preview color" needed - inheritance always resolves to global palette

**Palette Editing:**
- Panel showing 12 color swatches
- For each slot: Can set color OR mark as "empty/inherit"
- Global palette: All slots must have at least one side filled

**Center/Axis Geometry:**
- Points exactly on x=0 (symmetry axis): **BLEND left and right colors**
- **TRACK THIS DECISION** - may want to change later

---

## Feature #7: Left/Right Palette Variation

**Region Determination:**
- Automatic based on geometry side
- Stored geometry (right side, x > 0) → uses RIGHT palette
- Mirrored geometry (left side, x < 0) → uses LEFT palette

**Rotation Interaction:**
- L/R colors rotate WITH the asset (character-relative)
- NOT fixed to world coordinates
- Character's left hand stays their left hand across all rotation frames

**Center Handling:**
- Geometry on x=0: Blend left and right colors
- **TRACKED IMPLEMENTATION DETAIL**

---

## Feature #8: TypeScript Rendering Library

**Architecture:**
- Standalone TypeScript library
- Used by both editor (preview) and game projects

**Target API:**
- Canvas2D (browser CanvasRenderingContext2D)

**Dependencies:**
- Standard npm/TypeScript package management
- Standard browser APIs only

**Library Implements:**
- Vector path rendering
- Symmetry application (mirroring)
- Palette resolution (tree-walking inheritance)
- Component assembly (unit tree composition)
- Lighting gradients
- Tri-pixel rasterization
- PNG/WebP generation
- Sprite sheet layout
- Metadata generation

**Main API:**
```typescript
function render(asset: Asset, options?: RenderOptions): Canvas2DContext
function generateSpriteSheet(assets: Asset[], options?): SpriteSheetResult
function generateRotations(asset: Asset, count: number): Canvas2DContext[]
```

---

## Features 9-12 (Added Features)

### Feature #9: Client-Side Web Application
- Browser-based (HTML/CSS/JavaScript/TypeScript)
- No server-side dependencies
- GitHub Pages compatible
- File I/O via File System Access API or downloads

### Feature #10: Negative Space Paths
- Boolean subtraction for creating cutouts
- Path type flag: 'positive' or 'negative'
- Symmetry-compatible
- Render-time processing

### Feature #11: Sprite Sheet Metadata Export
- JSON companion file for sprite sheets
- Frame locations (x, y, width, height)
- Rotation angles
- Component composition data
- Palette snapshots
- Source asset references

### Feature #12: Automatic Palette Generation
- Generate palette from single base color
- Hexadic color scheme (6-color harmony)
- Multiple shades/tints
- Nice-to-have feature

### Feature #13: Undo/Redo (NEW)
- Nice-to-have feature
- Not in MVP
- Details TBD

---

## Editor UI Architecture

### Two-Pane Layout

**Editor Pane (Left):**
- Shows ONLY wireframe (black lines, maybe gray fill)
- NO colors displayed
- Geometric editing only
- Grid always visible

**Preview Pane (Right):**
- Shows fully rendered sprite using rendering library
- ALL colors, lighting, effects applied
- Real-time updates
- Same output as export

### Editor Hierarchy

**Project Level:**
- List of components/units (root level)
- Each has name
- Actions: Edit, Delete
- "Create New Component" button

**Component Editor:**
- Edit one unit at a time
- Shows flat list of:
  - Paths in this unit
  - Child components in this unit
- Breadcrumb navigation (Root > Body > Arm)
- Back button to return to parent

**Path List:**
- Selectable list of paths
- Selected path actions:
  - Move individual points
  - Change stroke palette slot (1-12 or none)
  - Change fill palette slot (1-12 or none)
  - Delete path

**Child Component List:**
- List of child units
- Actions: Delete
- Can navigate into child (drill down)
- "Add Component" button (adds child unit)

### Context-Aware Preview
- When editing child component, preview pane can show it in parent assembly context
- Useful for seeing palette inheritance in action
- Toggle or automatic when navigating into child

### Drawing Workflow

1. Select palette slot (1-12) for stroke
2. Select palette slot (1-12) for fill
3. Click to place points on grid
4. Double-click or Enter to close path
5. Path appears in path list
6. Preview pane shows rendered result

### No Planned Features
- No copy/paste
- No zoom/pan (currently)
- No moving whole paths (only individual points)
- No curved paths (straight edges only)
- No transform tools (move, rotate, scale paths)
- No undo/redo (unless added as nice-to-have #13)

### Nice-to-Have Editor Features
- Add points to existing paths
- Delete individual points from paths
- Undo/redo (#13)

---

## File Formats

### Working File (.trout)
- Format: JSON
- Encoding: UTF-8
- Single file per project
- Version control friendly
- Contains all assets and unit tree

### Sprite Sheet Export
- Format: PNG or WebP
- RGBA (8-bit per channel)
- Grid layout with configurable spacing

### Metadata Export (.json)
- JSON companion file for sprite sheets
- Frame locations and metadata
- Component composition data
- Rotation information

---

## Data Validation Rules

1. **Version:** Must match supported version string
2. **Global Palette:** Each slot (1-12) must have at least one side (left OR right OR both) filled
3. **Asset IDs:** Must be unique within document (UUIDs)
4. **Unit IDs:** Must be unique within asset (UUIDs)
5. **Unit Tree:** Must be acyclic - NO circular references
6. **Paths:** Must have at least 2 points
7. **Closed Paths:** Must have at least 3 points
8. **Stroke/Fill:** Each path must have at least stroke OR fill (or both)
9. **Palette Slot References:** Must be valid integers 1-12
10. **Palette Resolution:** All slot references must eventually resolve via inheritance chain

---

## Open Design Questions (Answered)

### Scope & Philosophy
1. ✓ "Super simple" = low line count primarily
2. ✓ Workflow: Create many simple assets quickly
3. ✓ Target: Excalibur.js primary, any TypeScript engine secondary

### Symmetry Architecture
4. ✓ Fixed vertical axis at x=0
5. ✓ Bilateral only (no radial, no multi-axis)
6. ✓ Symmetry always applies (color asymmetry only)

### Grid System
7. ✓ Grid defines hexagonal canvas boundaries
8. ✓ Triangular grid only (no rectangular option)

### Export & File Format
9. ✓ Native format: JSON (.trout)
10. ✓ Raster formats: PNG, WebP
11. ✗ No import support (out of scope)
12. ✓ Sprite sheet export supported

### Rendering & Performance
13. ✓ Real-time preview required (separate pane)
14. ✓ No complexity limits (if slow with 1000 paths, so be it)
15. ✓ Batch export for rotation variants

### File Format & Persistence
16. ✓ Single JSON file per project
17. ✓ JSON (text, human-readable, VCS-friendly)
18. ✓ Yes, designed for git diffing
19. ✓ UUIDs for references
20. ✓ No undo/redo (unless Feature #13 added)

### Canvas & Workspace
21. ✓ Fixed hexagonal canvas (64 grid units)
22. ✓ Abstract integer grid units
23. ✓ Origin at center (0, 0)
24. ✗ No zoom planned currently
25. ✓ Single asset at a time (no multi-asset tabs)

### Drawing & Editing Tools
26. ✓ Paths only (no circles, rectangles as primitives)
27. ✗ No Bezier curves (straight edges only)
28. ✓ Both fill and stroke supported
29. ✓ Click-to-place points
30. ✓ Click to select paths from list
31. ✗ No moving whole paths (only individual points)
32. ✓ Palette-based (12 slots × 2 sides)

### Component System Workflow
33. ✓ "Create New Component" button at project level
34. ✓ Component editor is separate context with breadcrumb/back navigation
35. ✓ Preview pane shows context (can show child in parent assembly)
36. ✓ Flat list with names (not tree view)

### Export Workflows
37. ✓ Export profiles not important now
38. ✓ Batch export for rotation variants
39. ✓ Sprite sheets primary; rotation variants get number suffix
40. ✓ Preview pane provides live preview (no separate export preview dialog)

### Performance & Scalability
41. ✓ No performance targets (simple implementation, optimize later if needed)
42. ✓ Full redraws fine (no dirty-region optimization)
43. ✓ No caching optimization needed initially
44. N/A (12 palette slots is reasonable)

### Platform & Distribution
45. ✓ Web app (browser-based)
46. ✓ Offline support possible (PWA)
47. ✓ Browser-based (no installation)
48. ✓ Local filesystem (File System Access API or downloads)
49. ✓ Single-user only

---

## Philosophy & Principles

1. **Simplicity over optimization** - make it work, optimize later if needed
2. **No premature performance engineering** - full redraws are fine
3. **Library-first architecture** - rendering library is foundation
4. **Separation of concerns** - editor (wireframe) vs renderer (full visual)
5. **Minimal line count** - primary simplicity metric
6. **Workflow focus** - create many simple assets quickly
7. **No feature creep** - defer nice-to-haves to later phases

---

## Implementation Notes to Track

These are specific implementation details we may want to revisit:

1. **Axis blending (x=0):** Colors on symmetry axis blend left and right - may want to change
2. **Trixel rendering:** Choice between center-point sampling vs anti-aliasing - decide based on aesthetics
3. **Stroke rendering:** All 2^0 trixels along border - may make configurable
4. **Performance:** If slow, we'll optimize then (not prematurely)
5. **Circular reference validation:** Must implement to prevent infinite loops in unit tree

---

## Next Steps

1. ✓ Answer all implementation questions
2. Update specification with these decisions
3. Merge insights back into brainstorming document
4. Create TODO document for implementation
5. Begin Phase 0 (Ultra-Minimal MVP)

