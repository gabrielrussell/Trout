# Trout Vector Asset Editor - Technical Specification

**Version:** 1.0
**Date:** 2025-11-14
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
  id: string;                   // Unique identifier
  name: string;                 // Human-readable name
  shape: Shape;                 // The geometry (possibly empty)
  palette: Palette;             // Palette for this unit
  children: Unit[];             // Child units (z-order: later = on top)
}

interface Shape {
  paths: Path[];                // Array of paths (z-order: later = on top)
}

interface Path {
  type: 'positive' | 'negative'; // Positive (fill) or negative (cutout)
  points: Point[];              // Control points (stored un-mirrored)
  closed: boolean;              // Whether path is closed
  stroke?: StrokeStyle;         // Stroke definition (optional)
  fill?: FillStyle;             // Fill definition (optional)
}

interface Point {
  x: number;                    // X coordinate (triangular grid units)
  y: number;                    // Y coordinate (triangular grid units)
}

interface StrokeStyle {
  paletteSlot: number;          // Index into palette
  width: number;                // Stroke width in pixels
  leftOverride?: number;        // Optional: different slot for left side
}

interface FillStyle {
  paletteSlot: number;          // Index into palette
  leftOverride?: number;        // Optional: different slot for left side
}
```

#### Palette Schema

```typescript
interface Palette {
  slots: PaletteSlot[];         // Array of color slots
}

interface PaletteSlot {
  right: Color | null;          // Right side color (null = inherit)
  left?: Color | null;          // Left side color (null = use right, undefined = same as right)
}

interface Color {
  r: number;                    // Red (0-255)
  g: number;                    // Green (0-255)
  b: number;                    // Blue (0-255)
  a: number;                    // Alpha (0-1)
}
```

### Coordinate System

**Triangular Grid:**
- Base unit: 1 grid unit
- Grid spacing in pixels: Configurable (powers of 2: 2px, 4px, 8px, 16px, etc.)
- Origin: Center of canvas
- Axis orientation:
  - X-axis: Horizontal (right is positive)
  - Y-axis: Vertical (down is positive)

**Hexagonal Canvas:**
- Canvas boundary is a hexagon aligned to the triangular grid
- Canvas size specified as the distance from center to vertex (in pixels)

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

### Symmetry Algorithm

**Storage:** Only right half of geometry stored
**Render-time application:**

```
For each point P(x, y):
  1. Render original point at (x, y)
  2. Render mirrored point at (-x, y)
```

**Path rendering with symmetry:**
- Close the path by connecting the mirrored points
- Mirror follows the original path order in reverse

### Palette Resolution Algorithm

```typescript
function resolveColor(
  unit: Unit,
  paletteSlot: number,
  side: 'left' | 'right'
): Color {
  // Look up slot in current unit's palette
  let slot = unit.palette.slots[paletteSlot];

  // If slot exists and has color for this side, return it
  if (slot) {
    if (side === 'left' && slot.left !== undefined) {
      if (slot.left !== null) return slot.left;
      // If left is explicitly null, fall through to check right
      if (slot.right !== null) return slot.right;
    } else if (side === 'right' && slot.right !== null) {
      return slot.right;
    }
  }

  // If we reach here, walk up the tree
  if (unit.parent) {
    return resolveColor(unit.parent, paletteSlot, side);
  }

  // If we reach root, use global palette (must have value)
  return globalPalette.slots[paletteSlot].right!;
}
```

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
- **Editor Canvas:** Shows vector paths, control points, grid, symmetry line
- **Preview Canvas:** Shows live rendered output using the rendering library

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

#### 1. Component Tree Panel
- Hierarchical view of units
- Drag-and-drop reordering
- Add/delete units
- Name editing

#### 2. Palette Panel
- Color slots (numbered)
- Color picker for each slot
- Empty slot indicator (inherits from parent)
- Left/right color variants
- Auto-generate button (nice-to-have)

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

**Deliverable:** Complete tool with all planned features

---

## Validation Rules

### Data Validation

**Document Level:**
- Version must match supported version string
- Global palette must have no empty slots (all slots must have right color defined)
- All asset IDs must be unique within document
- All unit IDs must be unique within asset

**Asset Level:**
- Canvas size must be positive integer
- Root unit must exist
- Unit tree must be acyclic (no circular references)

**Unit Level:**
- Palette slots can be sparse (missing indices inherit from parent)
- If left color is specified, right color must also be specified
- Empty slots (null values) are only valid in non-root units

**Path Level:**
- Paths must have at least 2 points
- Closed paths must have at least 3 points
- Either stroke or fill must be defined (or both)
- Palette slot references must be valid integers

**Palette Resolution:**
- All palette slot references must eventually resolve to global palette
- Global palette must have sufficient slots for all referenced indices

---

## Performance Considerations

### Rendering Performance
- Target: 60 FPS for real-time preview
- Use requestAnimationFrame for canvas updates
- Debounce palette changes
- Cache resolved palettes when possible

### Memory Management
- Limit maximum asset count per document
- Warn on large sprite sheets (>4096x4096)
- Implement undo/redo with history limit

### Browser Compatibility
- Target: Modern browsers with Canvas API support
- Minimum: Chrome 90+, Firefox 88+, Safari 14+, Edge 90+

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

