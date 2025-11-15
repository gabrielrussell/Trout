# Trout Vector Asset Editor - Technical Specification

**Version:** 2.0
**Date:** 2025-11-14
**Last Updated:** 2025-11-14 (Post-Review)
**Status:** Draft

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Data Structures](#data-structures)
4. [File Formats](#file-formats)
5. [Rendering Specification](#rendering-specification)
6. [API Specification](#api-specification)
7. [Editor UI Specification](#editor-ui-specification)
8. [Export Specification](#export-specification)
9. [Implementation Priorities](#implementation-priorities)

---

## Overview

### Purpose
This document provides the technical specification for Trout, a web-based vector asset editor designed for creating 2D top-down game assets. The editor generates sprite sheets and provides a standalone TypeScript rendering library for game engines.

### Target Platform
- **Client-side web application** (HTML/CSS/JavaScript/TypeScript)
- **No server-side dependencies**
- **GitHub Pages compatible**
- **Primary game engine target:** Excalibur.js
- **Secondary target:** Any TypeScript game engine

### Core Principles
- **Separation of concerns:** Editor and renderer are distinct systems
- **Library-first architecture:** TypeScript rendering library is foundation
- **Minimal complexity:** Keep codebase small and maintainable
- **Workflow focus:** Create many simple assets quickly

---

## Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Browser Environment                   │
│                                                          │
│  ┌────────────────────┐      ┌─────────────────────┐   │
│  │   Editor UI        │      │   Renderer Output   │   │
│  │                    │      │                     │   │
│  │  - Canvas          │      │  - Preview Canvas   │   │
│  │  - Tools           │      │  - Live Updates     │   │
│  │  - Palette Editor  │      │                     │   │
│  │  - Component Tree  │      │                     │   │
│  └─────────┬──────────┘      └──────────▲──────────┘   │
│            │                            │              │
│            │ Working File (JSON)        │              │
│            ▼                            │              │
│  ┌──────────────────────────────────────┴──────────┐   │
│  │   TypeScript Rendering Library                  │   │
│  │                                                  │   │
│  │  - Vector Path Rendering                        │   │
│  │  - Symmetry Application                         │   │
│  │  - Palette Resolution                           │   │
│  │  - Component Assembly                           │   │
│  │  - PNG/WebP Generation                          │   │
│  └──────────────────────────────────────────────────┘   │
│                            │                            │
│                            ▼                            │
│                   ┌────────────────┐                    │
│                   │ Export Output  │                    │
│                   │ - Sprite Sheet │                    │
│                   │ - JSON Metadata│                    │
│                   └────────────────┘                    │
└─────────────────────────────────────────────────────────┘
```

### Module Structure

```
trout/
├── src/
│   ├── renderer/              # TypeScript rendering library (standalone)
│   │   ├── core/
│   │   │   ├── vector.ts      # Path rendering
│   │   │   ├── symmetry.ts    # Symmetry application
│   │   │   ├── palette.ts     # Color resolution
│   │   │   └── compositor.ts  # Component assembly
│   │   ├── export/
│   │   │   ├── sprite-sheet.ts
│   │   │   └── metadata.ts
│   │   └── index.ts           # Public API
│   │
│   ├── editor/                # Editor UI
│   │   ├── canvas/
│   │   ├── tools/
│   │   ├── panels/
│   │   └── state/
│   │
│   └── types/                 # Shared type definitions
│       └── schema.ts
│
└── package.json
```

---

## Data Structures

### Working File Format (JSON)

The working file format is a JSON document that stores all vector asset data. This is the format that the editor saves and loads.

#### Root Schema

```typescript
interface TroutDocument {
  version: string;              // Format version (e.g., "1.0")
  metadata: DocumentMetadata;   // Document-level metadata
  globalPalette: Palette;       // Root palette (no empty slots)
  assets: Asset[];              // Array of assets in this document
}

interface DocumentMetadata {
  name: string;
  created: string;              // ISO 8601 timestamp
  modified: string;             // ISO 8601 timestamp
  author?: string;
}
```

#### Asset Schema

```typescript
interface Asset {
  id: string;                   // Unique identifier (UUID)
  name: string;                 // Human-readable name
  rootUnit: Unit;               // Root of the component tree
  canvasSize: number;           // Hexagonal canvas size in pixels
  exportSettings?: ExportSettings;
}

interface ExportSettings {
  generateRotations: boolean;   // Generate rotation frames
  rotationCount: number;        // Number of rotation frames (if enabled)
  includeMetadata: boolean;     // Export JSON metadata
}
```

#### Unit Schema (Component Tree)

```typescript
interface Unit {
  id: string;                   // Unique identifier (UUID)
  name: string;                 // Human-readable name
  shape: Shape;                 // The geometry (possibly empty)
  palettes: UnitPalettes;       // Left and right palettes for this unit
  children: Unit[];             // Child units (z-order: later = on top)
}

interface Shape {
  paths: Path[];                // Array of paths (z-order: later = on top)
}

interface Path {
  type: 'positive' | 'negative'; // Positive (fill) or negative (cutout)
  points: Point[];              // Control points (stored un-mirrored, right side only)
  closed: boolean;              // Whether path is closed
  stroke?: StrokeStyle;         // Stroke definition (optional)
  fill?: FillStyle;             // Fill definition (optional)
}

interface Point {
  x: number;                    // X coordinate (triangular grid units, integer)
  y: number;                    // Y coordinate (triangular grid units, integer)
}

interface StrokeStyle {
  paletteSlot: number;          // Index into palette (0-11)
  width: number;                // Stroke width in grid units
}

interface FillStyle {
  paletteSlot: number;          // Index into palette (0-11)
}
```

#### Palette Schema

**Important:** Each unit has TWO separate palettes (left and right), treated equally.

```typescript
interface UnitPalettes {
  left: Palette;                // Left side palette (12 slots)
  right: Palette;               // Right side palette (12 slots)
}

interface Palette {
  slots: (Color | null)[];      // Array of 12 color slots (null = inherit from parent)
}

interface Color {
  r: number;                    // Red (0-255)
  g: number;                    // Green (0-255)
  b: number;                    // Blue (0-255)
  a: number;                    // Alpha (0-1)
}
```

**Palette Resolution Rules:**
- Each palette has exactly 12 slots (indexed 0-11)
- Slot can be a Color or null (null = inherit from parent)
- **Global palette requirement:** For each slot, at least ONE side (left OR right OR both) must be filled
- **Lookup for left side:** Walk up tree checking left palette, then walk up tree checking right palette
- **Lookup for right side:** Walk up tree checking right palette, then walk up tree checking left palette

### Coordinate System

**Triangular Grid:**
- Regular triangular tessellation (alternating up/down triangles)
- Base unit: 1 grid unit (abstract integer coordinates)
- Origin: Center of canvas at (0, 0)
- Axis orientation:
  - X-axis: Horizontal (right is positive)
  - Y-axis: Vertical (down is positive)
- Grid spacing in editor: Selected power-of-2 level (2^0 through 2^6)
  - This is SNAPPING precision, not canvas size

**Hexagonal Canvas:**
- **Fixed size:** 2^6 (64) grid units wide
- Canvas boundary is a hexagon aligned to the triangular grid
- Flat-topped orientation
- **Never changes** - size is fixed per asset

**Pixels vs Grid Units:**
- Working file stores only grid coordinates (size-agnostic)
- At render/export time: User specifies output size in pixels
- System calculates: pixels-per-grid-unit = output_size / 64
- All geometry scales proportionally

**Grid Mathematics:**

*1D Scaling:*
- One 2^n trixel base = 2^n × 2^0 trixel bases

*2D Area Scaling:*
- One 2^n trixel = 4^n × 2^0 trixels (area quadruples per level)

*Hexagon Composition:*
- Regular hexagon = 6 equilateral triangles meeting at center
- Hexagon width 2^n → each triangle side length = 2^(n-1)
- Total 2^0 trixels in hexagon = 6 × (2^(n-1))² = 6 × 2^(2n-2)
- Example: 64-unit hexagon contains 6 × 32² = 6,144 smallest trixels

*Pixel Alignment Challenge:*
- Triangular trixels rendered to square pixel grid
- Only one edge aligns perfectly with pixels
- Other two angled edges require anti-aliasing/coverage calculation

---

## File Formats

### 1. Working File (.trout)

**Format:** JSON
**Encoding:** UTF-8
**Schema:** See Data Structures section above

**Example:**

```json
{
  "version": "1.0",
  "metadata": {
    "name": "My Game Assets",
    "created": "2025-11-14T12:00:00Z",
    "modified": "2025-11-14T14:30:00Z"
  },
  "globalPalette": {
    "slots": [
      { "right": { "r": 255, "g": 0, "b": 0, "a": 1.0 } },
      { "right": { "r": 0, "g": 255, "b": 0, "a": 1.0 } },
      { "right": { "r": 0, "g": 0, "b": 255, "a": 1.0 } }
    ]
  },
  "assets": [
    {
      "id": "asset-001",
      "name": "Player Character",
      "canvasSize": 64,
      "rootUnit": {
        "id": "unit-001",
        "name": "Root",
        "shape": { "paths": [] },
        "palette": { "slots": [] },
        "children": []
      }
    }
  ]
}
```

### 2. Sprite Sheet Export

**Format:** PNG or WebP
**Encoding:** RGBA (8-bit per channel)
**Layout:** Grid layout with configurable spacing

### 3. Metadata Export (.json)

**Format:** JSON
**Purpose:** Companion file for sprite sheets containing frame locations, rotation data, and composition information

```typescript
interface SpriteSheetMetadata {
  version: string;
  spriteSheetFile: string;      // Filename of associated sprite sheet
  spriteSheetSize: {
    width: number;
    height: number;
  };
  frames: Frame[];
}

interface Frame {
  id: string;                   // Unique frame identifier
  sourceAssetId: string;        // ID of source asset in .trout file
  location: {
    x: number;                  // Top-left X in sprite sheet
    y: number;                  // Top-left Y in sprite sheet
    width: number;              // Frame width
    height: number;             // Frame height
  };
  rotation?: number;            // Rotation angle (degrees, if part of rotation set)
  composition: {
    components: string[];       // IDs of units used in this frame
    paletteSnapshot: Palette;   // Resolved palette for this frame
  };
}
```

---

## Rendering Specification

### Rendering Pipeline

The rendering process follows this sequence:

```
1. Parse unit tree
2. For each unit (depth-first):
   a. Resolve palette (inherit from parent as needed)
   b. Render shape:
      i. For each path in shape:
         - Apply symmetry (mirror across vertical axis)
         - Render stroke (if defined)
         - Render fill (if defined)
         - Apply negative paths (boolean subtraction)
   c. Render children (recursive)
3. Composite all units (z-order)
4. Export to PNG/WebP
```

### Component Positioning Model

**Fixed Center-Aligned Composition:**
- ALL units are rendered at the same center point: (0, 0)
- **No transforms:** No translation, rotation, scale, or positioning
- **No bounding boxes:** No center calculation needed
- **Implicit alignment:** All geometry shares the same origin
- Components compose by overlaying at shared center

**Rendering Order:**
1. Each unit renders its own `shape.paths[]` at (0, 0)
2. Then renders all `children[]` units (each also at (0, 0))
3. Z-order determined by array position (later = on top)

**Implications:**
- To create complex shapes, draw paths in multiple units
- To layer components, add them as children
- All paths from all units overlay at same center
- No need for "attachment points" or positioning systems

### Symmetry Algorithm

**Half-Geometry Storage:**
- Store ONLY the right half of geometry (x ≥ 0)
- Points exactly on the symmetry axis (x = 0) are stored once
- Left half (x < 0) is NOT stored - purely computed at render time
- Left half is NOT editable in the editor (read-only visual)

**Render-time application:**

```
For each point P(x, y) where x ≥ 0:
  1. Render original point at (x, y)  [right side]
  2. If x > 0: Render mirrored point at (-x, y)  [left side]
  3. If x = 0: Point is on axis, render once only
```

**Path rendering with symmetry:**
- Original points form right half of path
- Mirrored points form left half of path
- For closed paths: Connect mirrored points back to original points
- Mirror follows the original path order in reverse
- Result: Symmetric closed shape

### Palette Resolution Algorithm

**Dual-Palette Lookup:** Each unit has separate left and right palettes (12 slots each). Color resolution walks up the tree, checking the matching-side palette first, then the opposite-side palette as fallback.

```typescript
function resolveColor(
  unit: Unit,
  paletteSlot: number,
  side: 'left' | 'right',
  globalPalettes: UnitPalettes
): Color {
  // Determine which palette to check first based on side
  const primaryPalette = side === 'left' ? 'left' : 'right';
  const fallbackPalette = side === 'left' ? 'right' : 'left';

  // Walk up tree checking primary side
  let currentUnit: Unit | null = unit;
  while (currentUnit) {
    const color = currentUnit.palettes[primaryPalette].slots[paletteSlot];
    if (color !== null) {
      return color;
    }
    currentUnit = currentUnit.parent;
  }

  // If not found, walk up tree checking fallback side
  currentUnit = unit;
  while (currentUnit) {
    const color = currentUnit.palettes[fallbackPalette].slots[paletteSlot];
    if (color !== null) {
      return color;
    }
    currentUnit = currentUnit.parent;
  }

  // If still not found, check global palette
  // Global palette must have at least one side filled for each slot
  const globalColor = globalPalettes[primaryPalette].slots[paletteSlot];
  if (globalColor !== null) {
    return globalColor;
  }

  return globalPalettes[fallbackPalette].slots[paletteSlot]!;
}
```

**Special Cases:**
- **Geometry on symmetry axis (x=0):** Blend left and right colors (50/50 mix)
- **Global palette validation:** Each slot must have at least one side (left OR right OR both) filled

### Negative Space Algorithm

**Boolean Subtraction:**

```
For each negative path in a shape:
  1. Render the positive paths to a temporary buffer
  2. Apply symmetry to the negative path
  3. Use path as a mask to cut out from the buffer
  4. Composite result
```

**Implementation Notes:**
- Use HTML5 Canvas composite operations: `globalCompositeOperation = 'destination-out'`
- Apply negative paths in order after all positive paths are rendered
- Negative paths also respect symmetry

### Stroke and Fill Rendering

**Order of operations:**
1. Render all fills first
2. Then render all strokes on top

**For each path:**
1. Resolve fill color (if fill is defined)
2. Apply symmetry and render fill
3. Resolve stroke color (if stroke is defined)
4. Apply symmetry and render stroke with specified width

---

## API Specification

### TypeScript Rendering Library API

The rendering library is a standalone TypeScript library that can be used both by the editor and by game projects.

#### Core API

```typescript
// Main rendering function
export function render(asset: Asset, options?: RenderOptions): Canvas2DContext;

export interface RenderOptions {
  gridSpacing?: number;         // Grid spacing in pixels (default: 4)
  backgroundColor?: Color;      // Background color (default: transparent)
  width?: number;               // Output width (default: calculated from canvasSize)
  height?: number;              // Output height (default: calculated from canvasSize)
}

// Generate sprite sheet
export function generateSpriteSheet(
  assets: Asset[],
  options?: SpriteSheetOptions
): SpriteSheetResult;

export interface SpriteSheetOptions {
  spacing?: number;             // Spacing between sprites (default: 2)
  padding?: number;             // Padding around edges (default: 2)
  format?: 'png' | 'webp';      // Output format (default: 'png')
  backgroundColor?: Color;
}

export interface SpriteSheetResult {
  canvas: HTMLCanvasElement;    // The generated sprite sheet
  metadata: SpriteSheetMetadata; // Metadata for frames
}

// Generate rotation frames
export function generateRotations(
  asset: Asset,
  count: number,
  options?: RenderOptions
): Canvas2DContext[];

// Utility functions
export function validateDocument(doc: TroutDocument): ValidationResult;
export function createEmptyDocument(): TroutDocument;
export function createEmptyAsset(name: string): Asset;
```

#### Palette Utilities

```typescript
export function resolvePalette(unit: Unit, globalPalette: Palette): ResolvedPalette;

export interface ResolvedPalette {
  slots: ResolvedPaletteSlot[];
}

export interface ResolvedPaletteSlot {
  right: Color;                 // Always resolved (never null)
  left: Color;                  // Always resolved (never null)
}

// Auto-generate palette from single color
export function generatePalette(baseColor: Color, scheme: ColorScheme): Palette;

export type ColorScheme = 'hexadic' | 'triadic' | 'complementary' | 'analogous';
```

---

## Editor UI Specification

### Layout

**Main Window Structure:**

```
┌─────────────────────────────────────────────────────────────┐
│ Menu Bar                                                    │
├──────────────┬────────────────────────────┬─────────────────┤
│              │                            │                 │
│  Left Panel  │     Main Canvas            │   Right Panel   │
│              │                            │                 │
│  - Tools     │     (Editor View)          │  - Properties   │
│  - Layers    │                            │  - Palette      │
│              │                            │  - Component    │
│              │                            │    Tree         │
│              │                            │                 │
├──────────────┼────────────────────────────┼─────────────────┤
│              │  Preview Canvas            │                 │
│              │  (Rendered Output)         │                 │
└──────────────┴────────────────────────────┴─────────────────┘
```

**Two Canvas System:**
- **Editor Canvas (Wireframe Only):**
  - Shows ONLY black line outlines of paths
  - Possibly gray fill to distinguish shapes
  - NO colors displayed
  - Grid always visible
  - Symmetry line visible (x=0)
  - Control points editable
  - Pure geometric editing interface

- **Preview Canvas (Fully Rendered):**
  - Shows complete rendered output using rendering library
  - ALL colors, lighting, effects applied
  - Real-time updates as you edit
  - Identical to export output
  - Uses same rendering code as final export
  - Can show child components in parent context (context-aware preview)

### Tools

**Phase 0 (MVP):**
- Path drawing tool (click to add points)
- Point selection/move tool
- Delete point/path tool

**Phase 1:**
- Negative path toggle
- Stroke/fill color assignment
- Stroke width adjustment

**Phase 2:**
- Grid spacing control
- Snap to grid (always on)
- Component duplication

### Panels

#### 1. Component/Unit Navigation
**Project Level:**
- Flat list of root-level units (components)
- Each unit has a name
- Actions: Edit (opens component editor), Delete
- "Create New Component" button

**Component Editor Level:**
- Breadcrumb navigation (Root > Body > Arm)
- Back button to return to parent
- Shows current unit's:
  - List of paths (selectable)
  - List of child units (can navigate into)
- "Add Path" and "Add Child Component" buttons

#### 2. Palette Panel
- **Dual palette UI:** Left palette (12 slots) and Right palette (12 slots)
- Each slot numbered 0-11
- For each slot:
  - Color picker (RGB + alpha)
  - "Empty/Inherit" checkbox (marks slot as null)
- Global palette: At least one side must be filled per slot
- Auto-generate button (Feature #12, nice-to-have)

#### 3. Properties Panel
- Selected path properties:
  - Type (positive/negative)
  - Stroke (on/off, palette slot, width)
  - Fill (on/off, palette slot)
- Canvas properties:
  - Size
  - Grid spacing

### Interaction Patterns

**Path Drawing:**
1. Select path tool
2. Click to place points (snaps to grid)
3. Double-click or press Enter to close path
4. ESC to cancel

**Point Editing:**
1. Select move tool
2. Click point to select
3. Drag to move (snaps to grid)
4. Delete key to remove point

**Symmetry:**
- Always visible as vertical line at x=0
- Automatically mirrors all geometry
- Cannot be moved or disabled

---

## Export Specification

### Sprite Sheet Generation

**Layout Algorithm:**
1. Calculate bounding box for each frame
2. Sort frames by height (tallest first)
3. Pack into rows using shelf packing algorithm
4. Add padding and spacing as specified

**Rotation Frame Generation:**
- Generate N evenly spaced rotations (360° / N)
- For each rotation:
  1. Apply lighting gradient (if lighting feature enabled)
  2. Render frame
  3. Store in sprite sheet

**Metadata Generation:**
- For each frame, record:
  - Position in sprite sheet (x, y, width, height)
  - Source asset ID
  - Rotation angle (if applicable)
  - Component IDs used
  - Resolved palette

### File Naming

**Convention:**
```
[asset-name]-sheet.[png|webp]        # Sprite sheet
[asset-name]-sheet.json              # Metadata
```

---

## Implementation Priorities

Based on the feature priorities defined in the brainstorming document, implementation should proceed in this order:

### Phase 0: Ultra-Minimal MVP
**Goal:** Prove end-to-end pipeline works

**Components:**
1. Basic data structures (Asset, Unit, Shape, Path, Point)
2. Simple vector path rendering (no symmetry, no palettes)
3. Minimal editor (draw one path, preview it)
4. Export to PNG

**Deliverable:** Can create a simple shape and export it

### Phase 1: Critical Features
**Goal:** Make it usable

**Components:**
1. Symmetry system (#1)
2. Component/Layer Assembly (#5)
3. Basic palettes (#6 - no inheritance)
4. Triangular grid snapping (#2)
5. Save/Load working files

**Deliverable:** Can create symmetric, multi-layer assets with colors

### Phase 2: Medium Priority Features
**Goal:** Add polish and workflow enhancements

**Components:**
1. Triangular pixel rendering (#3)
2. Negative space paths (#10)
3. Full palette inheritance (#6 advanced)
4. Left/Right palette variation (#7)
5. Sprite sheet metadata export (#11)

**Deliverable:** Full-featured workflow with advanced composition

### Phase 3: Nice-to-Have Features
**Goal:** Polish and quality of life

**Components:**
1. Faux 3D lighting with rotations (#4)
2. Automatic palette generation (#12)
3. Undo/Redo system (#13)

**Deliverable:** Complete tool with all planned features

### Infrastructure Features (Will Get Done)
These are foundational and will be implemented as needed:
- **TypeScript Rendering Library (#8):** Core foundation used by editor and game projects
- **Client-Side Web Application (#9):** Browser-based architecture with no server dependencies

---

## Validation Rules

### Data Validation

**Document Level:**
- Version must match supported version string
- **Global palettes:** For each slot (0-11), at least ONE side (left OR right OR both) must have a color
- All asset IDs must be unique within document (UUIDs)
- All unit IDs must be unique within asset (UUIDs)

**Asset Level:**
- Canvas size fixed at 64 grid units (2^6)
- Root unit must exist
- **Unit tree must be acyclic:** NO circular references (Unit A cannot contain Unit B if Unit B contains Unit A, directly or through chain)

**Unit Level:**
- Each unit must have both left and right palettes (12 slots each)
- Palette slots can be null (inherit from parent)
- Empty slots (null values) are only valid in non-root units

**Path Level:**
- Paths must have at least 2 points
- Closed paths must have at least 3 points
- Either stroke or fill must be defined (or both)
- Palette slot references must be valid integers (0-11)
- Control points must be integers (triangular grid units)

**Palette Resolution:**
- All palette slot references (0-11) must eventually resolve via inheritance
- Global palettes ensure all slots can resolve (at least one side filled)

---

## Performance Considerations

**Philosophy:** Simplicity over optimization. Implement straightforward solutions first, optimize only if actually slow.

### Rendering Performance
- Target: Real-time preview updates
- Use requestAnimationFrame for canvas updates
- **No premature optimization:**
  - Full redraws are acceptable
  - No dirty-region tracking initially
  - No component caching initially
- Optimize later if performance issues arise

### Memory Management
- No hard limits on asset count or path count
- If user creates 1000 paths and it's slow, that's acceptable
- Warn on very large sprite sheets (>4096x4096) if needed
- Undo/redo (Feature #13) would have history limit if implemented

### Browser Compatibility
- Target: Modern browsers with Canvas API support
- Minimum: Chrome 90+, Firefox 88+, Safari 14+, Edge 90+

---

## Implementation Notes

**Resolved Design Decisions** (from specification reviews):

### Path Data Structure
- Paths are arrays of integer points: `Point[]` where `Point = {x: number, y: number}`
- Points are in triangular grid coordinate space
- Paths use straight line segments only (no Bezier curves)
- Closed paths connect last point back to first

### Grid Coordinate System
- **Coordinate space:** Simple Cartesian (x, y) on triangular grid lattice
- **No special hex coordinates needed:** Standard integer grid works
- **Triangular tessellation:** Regular pattern of alternating up/down triangles
- **Pixel conversion:** At render time, multiply grid coordinates by (output_size / 64)

### Z-Ordering Clarification
- **Within a unit:** Paths in `shape.paths[]` are ordered (later = on top)
- **Between units:** Children in `children[]` are ordered (later = on top)
- **Render order:** All paths of current unit, THEN all children units
- **Not interleaved:** Paths and child units are in separate arrays

### Rotation Generation (Feature #4)
- **6-way minimum:** 0°, 60°, 120°, 180°, 240°, 300°
- **12-way option:** Every 30° for smoother rotation
- **What rotates:** The geometry (asset facing different directions)
- **Light stays fixed:** In world space (not rotating around asset)
- **L/R colors:** Rotate WITH the asset (character-relative, not world-relative)

### Negative Space Rendering (Feature #10)
- **Algorithm:** Canvas2D `globalCompositeOperation = 'destination-out'`
- **Order:** Render all positive paths first, then subtract negative paths
- **Symmetry:** Negative paths also respect bilateral symmetry

### File Format
- **Format:** JSON (human-readable, version-control friendly)
- **Extension:** .trout
- **Encoding:** UTF-8
- **IDs:** UUIDs for asset and unit identifiers

### Import Support
- **Not in v1.0:** No import of external formats (SVG, PNG, etc.)
- **May add later:** Potential future enhancement
- **Focus:** Asset creation, not editing imported assets

### Grid Scale Selection (Issue #9)
- **Behavior:** Changing grid snapping level does NOT affect existing geometry
- Points already placed remain at their exact positions
- Grid level change only affects snapping for new points and point movement operations
- Example: Switch from 2^2 to 2^4, existing points stay put

### Tri-Pixel Output (Issue #10)
- **No separate trixel size setting:** Whatever you drew is what renders
- Drawing at grid level 2^3 means those triangles render as trixels
- No "output trixel size" separate from construction grid
- Geometry and rendering are unified

### Sprite Sheet Layout (Issue #15)
- **Grid layout:** Rows and columns
- **Vertical (rows):** Different units/assets
- **Horizontal (columns):** Rotation angles (0°, 60°, 120°, etc.)
- Example: 3 units × 6 rotations = 18 frames in 3×6 grid

### Component Organization (Issue #16)
- **No special component library:** Just units in the tree
- Any unit can be reused as a child of another unit
- Components are organizational units, not a separate concept
- Project file contains flat list of root units

### Undo/Redo Granularity (Issue #17, if implementing Feature #13)
- **Undoable operations:** Layer list changes
  - Add/remove path from unit
  - Add/remove child unit
  - Move/reorder items in layer list
- **NOT undoable:** Individual point operations
  - Adding 3rd point to a path
  - Moving individual points (these complete immediately)
- **Granularity:** Unit editor "layer list" operations

### Auto Palette Generation Parameters (Issue #18, if implementing Feature #12)
- **Single input:** User selects one base color
- **System generates:** Complete palette using hexadic harmony + shades/tints
- **No other parameters required** for basic functionality
- **Optional (nice-to-have):** Saturation/brightness adjustment sliders

---

## Future Considerations

### Extensibility Points

**Custom Export Formats:**
- Plugin system for additional export formats
- Custom metadata schemas

**Advanced Features (Out of Scope for v1.0):**
- Animation timeline editor
- Texture fills (patterns)
- Gradient fills
- Radial symmetry
- Custom grid types

---

## Appendix

### Coordinate System Examples

**Triangular Grid Coordinates:**

```
      (-2,0)    (-1,0)    (0,0)    (1,0)    (2,0)
         *--------*--------*--------*--------*
        / \      / \      / \      / \      / \
       /   \    /   \    /   \    /   \    /   \
      /     \  /     \  /     \  /     \  /     \
     *-------*--------*--------*--------*--------*
(-2,1)   (-1,1)    (0,1)    (1,1)    (2,1)
```

### Color Space

All colors use sRGB color space with alpha channel (RGBA).

### Version History

- **1.0** (2025-11-14): Initial specification

