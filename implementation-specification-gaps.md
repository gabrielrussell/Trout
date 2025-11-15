# Trout Implementation Specification - Complete Gap Analysis

**Purpose**: Identify every ambiguity, under-specification, and missing detail that would block implementation.

**Perspective**: Writing code TODAY - what questions would arise?

---

## CRITICAL: Complete Data Structure Definitions Needed

### 1. Point Type - UNDEFINED

**Current State**: Mentioned throughout but never defined.

**Need to Specify**:
```typescript
// Option A: Cartesian coordinates (simplest)
type Point = {
  x: number;  // In what units? Pixels? Grid units?
  y: number;
};

// Option B: Hexagonal axial coordinates (if using hex grid)
type Point = {
  q: number;  // Axial coordinate q
  r: number;  // Axial coordinate r
};

// Option C: Triangular grid coordinates
type Point = {
  // What coordinate system for triangular grid?
  // Need: Explicit definition
};
```

**Questions**:
- Which coordinate system? Cartesian, axial, cube, offset?
- Are coordinates integers or floats?
- What are the units? Pixels, grid cells, abstract units?
- Is there a separate "screen coordinate" vs "grid coordinate" distinction?

**Blocker Level**: CRITICAL - Cannot write ANY code without this

---

### 2. Path Type - UNDEFINED

**Current State**: "a set of closed vector paths" mentioned, no structure given.

**Need to Specify**:
```typescript
// Minimum definition needed:
type Path = {
  points: Point[];           // Ordered vertices
  closed: boolean;           // Connect last to first?
  // More properties?
};

// Or more complete:
type Path = {
  points: Point[];
  closed: boolean;
  isNegative: boolean;       // Feature #10
  fillSlot?: PaletteSlotRef; // Which palette slot for fill?
  strokeSlot?: PaletteSlotRef; // Which palette slot for stroke?
  strokeWidth?: number;      // In what units?
};
```

**Questions**:
- Are paths always simple (no self-intersection)?
- Can paths have Bezier curves, or only straight segments?
- Is winding order (CW/CCW) significant?
- How are open vs closed paths handled?
- Can a path have zero points? One point? Two points? Minimum count?
- Are points stored in half-geometry form (x ≥ 0 only) or full geometry?

**Blocker Level**: CRITICAL - Cannot implement rendering without this

---

### 3. Shape Type - UNDEFINED

**Current State**: "exactly one shape (a set of closed vector paths, possibly empty)"

**Need to Specify**:
```typescript
// Current understanding - verify this is correct:
type Shape = {
  paths: Path[];  // Empty array = empty shape?
  // Other properties?
};

// Or does shape have more metadata?
type Shape = {
  paths: Path[];
  bounds?: BoundingBox;  // Cached for performance?
  center?: Point;        // Cached center point?
};
```

**Questions**:
- Can a shape have zero paths? (is "empty shape" literally `paths: []`?)
- Are paths in a shape ordered for rendering?
- How do multiple paths compose? Union? (Before negative space subtraction)
- Is there shape-level metadata separate from path-level?

**Blocker Level**: CRITICAL

---

### 4. Unit Type - PARTIALLY DEFINED

**Current State**: "each unit contains: exactly one shape + zero or more child units + a palette"

**Need to Specify**:
```typescript
type Unit = {
  id: string;               // Required for references?
  name?: string;            // Display name?
  shape: Shape;             // Exactly one (may be empty)
  children: Unit[];         // Ordered for Z-order?
  palette: Palette;

  // Missing specifications:
  center?: Point;           // Explicit center, or always computed?
  visible?: boolean;        // Can units be hidden?
  locked?: boolean;         // Can units be locked from editing?

  // How are z-order relationships expressed?
  // Are shapes and children in separate lists or interleaved?
};

// OR is it:
type Unit = {
  id: string;
  name?: string;
  palette: Palette;
  items: RenderItem[];      // Shapes AND children interleaved?
};

type RenderItem =
  | { type: 'shape', shape: Shape }
  | { type: 'unit', unit: Unit };
```

**Questions**:
- How is Z-ordering represented? Separate shape and children arrays, or interleaved?
- Is unit ID required? Auto-generated? User-specified?
- Can units be nested arbitrarily deep, or is there a limit?
- How are circular references prevented?
- What happens if a unit references itself (directly or indirectly)?

**Blocker Level**: CRITICAL - Affects core data model

---

### 5. Palette Type - PARTIALLY DEFINED

**Current State**: "palette with slots that can be filled or empty", "slots can have left/right colors"

**Need to Specify**:
```typescript
// How many slots? This is still TBD!
const PALETTE_SLOT_COUNT = 8;  // NEEDS DECISION

type Palette = {
  // Option A: Simple array with nulls for empty
  slots: (Color | null)[];  // Length = PALETTE_SLOT_COUNT

  // Option B: Separate left/right arrays
  rightSlots: (Color | null)[];
  leftSlots: (Color | null)[];   // null = inherit from right

  // Option C: Slots with optional left override
  slots: PaletteSlot[];
};

type PaletteSlot = {
  right: Color | null;  // null = inherit from parent
  left?: Color | null;  // undefined = use right; null = inherit from parent
};

type Color = {
  r: number;  // 0-255 or 0-1?
  g: number;
  b: number;
  a: number;  // 0-255 or 0-1?
};
```

**Questions**:
- **CRITICAL**: How many palette slots? 4? 8? 16?
- How is "empty" represented? `null`? `undefined`? Special sentinel value?
- For L/R palettes, how is "no left override" represented?
- Are color components 0-255 integers or 0-1 floats?
- Is alpha always present, or optional for opaque colors?
- Can palette slots be named, or only numbered?

**Blocker Level**: CRITICAL - Affects core data model and UI

---

### 6. PaletteSlotRef Type - UNDEFINED

**Current State**: "stroke and fill reference palette slots"

**Need to Specify**:
```typescript
type PaletteSlotRef = {
  slot: number;              // 0 to PALETTE_SLOT_COUNT-1
  side?: 'left' | 'right';   // For L/R variation; undefined = right?
};

// Or can you reference a direct color instead of a slot?
type ColorRef =
  | { type: 'palette', slot: number, side?: 'left' | 'right' }
  | { type: 'direct', color: Color };
```

**Questions**:
- Can you use direct colors instead of palette refs?
- If so, how do direct colors interact with palette inheritance?
- Is `side` optional with default to 'right'?
- Can you reference an out-of-bounds slot number? Error or clamp?

**Blocker Level**: HIGH

---

### 7. AssetFile Type - UNDEFINED (marked "TBD exact schema")

**Current State**: "JSON (TBD exact schema)"

**Need to Specify**:
```typescript
type AssetFile = {
  version: string;              // Format version, e.g., "1.0.0"
  name: string;                 // Asset name

  // Canvas/Grid settings
  canvasRadius?: number;        // Hexagon radius in pixels?
  gridScale: number;            // Triangle side length in pixels (power of 2)

  // Asset data
  rootUnit: Unit;               // Root of unit tree

  // Component library?
  library?: {
    [componentId: string]: Unit;  // Reusable components?
  };

  // Metadata?
  created?: string;             // ISO 8601 timestamp?
  modified?: string;
  author?: string;
};
```

**Questions**:
- What's in the root level?
- How is canvas size specified? Radius? Width/height? Grid cells?
- Is component library embedded in asset file or separate?
- How are component references resolved? By ID? By name?
- What metadata is included?
- Is there a schema version for format evolution?
- How are unknown fields handled (for forward compatibility)?

**Blocker Level**: CRITICAL - Cannot save/load files without this

---

## CRITICAL: Coordinate System & Grid Mathematics

### 8. Triangular Grid Coordinate System - UNDEFINED

**Current State**: "triangular/hexagonal lattice" mentioned, no coordinate system defined.

**Need to Specify**:

**A. Grid Topology - Which one?**
- Regular triangular tessellation (all triangles)
- Hexagonal grid with triangle vertices
- Other?

**B. Coordinate System - Pick one:**

**Option 1: Hexagonal Axial Coordinates (q, r)**
```
Point at (q=0, r=0) is at pixel (0, 0)
Point at (q=1, r=0) is at pixel (gridScale * sqrt(3), 0)
Point at (q=0, r=1) is at pixel (gridScale * sqrt(3)/2, gridScale * 3/2)

General formula:
pixelX = gridScale * (sqrt(3) * q + sqrt(3)/2 * r)
pixelY = gridScale * (3/2 * r)
```

**Option 2: Cartesian with snap-to-triangle logic**
```
Points stored as (x, y) in pixels
Snapping algorithm rounds to nearest triangle vertex
Need: Explicit snap-to-grid algorithm
```

**Option 3: Custom triangular coordinate system**
```
Need: Complete specification of coordinate math
```

**Questions**:
- **CRITICAL**: Which coordinate system is used?
- What is the grid origin point (0, 0) in pixel space?
- Are coordinates integers only, or can they be fractional?
- How do you convert grid coordinates → pixel coordinates? Need exact formula.
- How do you convert pixel coordinates → grid coordinates? Need exact formula.
- What is grid orientation? Pointy-top or flat-top hexagons?
- Do triangles have preferred orientation (all point up, alternating, etc.)?

**Blocker Level**: CRITICAL - Cannot implement grid without this

---

### 9. Half-Geometry Storage Details - UNDER-SPECIFIED

**Current State**: "Vector paths are stored un-mirrored (only half the shape)"

**Need to Specify**:

**A. Which half is stored?**
- Right half (x ≥ 0)?
- Left half (x < 0)?
- Right half not including center (x > 0)?

**B. Center line handling:**
```
For point exactly at x = 0:
- Stored once and never mirrored?
- Stored once and mirrored (appears twice)?
- Special handling?

For path that crosses x = 0:
Example: path from x=-5 to x=+5
- Split into two paths?
- Store only right portion?
- Store full path?
```

**C. Multi-path shapes:**
```
If shape has paths A, B, C:
- Does each path follow half-geometry storage independently?
- Or is half-geometry storage at shape level?

Example:
Shape with two paths:
  Path 1: points at x = -10, -5, 0, 5, 10
  Path 2: points at x = 2, 4, 6

Half-geometry storage of:
  Path 1: points at x = 0, 5, 10 (or 5, 10?)
  Path 2: points at x = 2, 4, 6
```

**D. Rendering algorithm:**
```typescript
function renderWithSymmetry(path: Path): RenderedPath {
  // Pseudo-code - need exact algorithm:

  // 1. Render right half as-is?
  // 2. Mirror across x=0 to create left half?
  // 3. How are center-line points handled?

  // Need: Exact step-by-step algorithm
}
```

**Questions**:
- Exactly which points are stored?
- How are center-line points handled?
- How are paths that cross the center line handled?
- Is the symmetry axis always at x=0, or can it be offset?
- During editing, are left-side points shown but locked?
- When user tries to select left-side point, what happens?

**Blocker Level**: CRITICAL - Affects data model, rendering, and editor

---

### 10. Grid Scale Selection & Existing Geometry - UNSPECIFIED

**Current State**: "Power-of-2 scaling system (2px, 4px, 8px, 16px, etc.)"

**Need to Specify**:

**A. Scale change behavior:**
```
User has drawn points at grid scale = 4px
User switches to grid scale = 8px

Q: What happens to existing points?

Option A: Points re-snap to new grid (may move)
Option B: Points keep position, may be off-grid
Option C: Points stored in "grid units" not pixels (position unchanged)
```

**B. Point storage format:**
```
// Option A: Store in grid coordinates (scale-independent)
type Point = {
  gridQ: number;    // Grid coordinate
  gridR: number;
};
// When grid scale changes, points stay at same grid positions

// Option B: Store in pixel coordinates
type Point = {
  x: number;        // Pixel coordinate
  y: number;
};
// When grid scale changes, points may be off-grid

// Which one?
```

**C. Off-grid points:**
```
If points can be off-grid:
- How are they displayed?
- Can they be moved?
- Do they snap to grid on next edit?
- Or are off-grid points forbidden?
```

**Questions**:
- Are points stored in grid coordinates or pixel coordinates?
- What happens to existing geometry when grid scale changes?
- Can points exist off-grid?
- What is the minimum grid scale? Maximum?
- Can user enter arbitrary grid scale, or only power-of-2 values?

**Blocker Level**: HIGH - Affects data model and editor behavior

---

## CRITICAL: Rendering Pipeline Step-by-Step

### 11. Rendering Pipeline - HIGH-LEVEL ONLY

**Current State**: 8 steps listed, but each step is under-specified.

**Need to Specify Each Step in Detail**:

**Step 1: Vector Construction**
```
Input: ???
Output: ???
Process: ???

Q: What is actually happening here?
Q: Is this an editing step, not a rendering step?
```

**Step 2: Component Assembly**
```
Input: Unit tree
Output: Flattened list of shapes with positions?

Algorithm needed:
function assembleComponents(root: Unit): AssembledShape[] {
  // Pseudo-code:
  // 1. Traverse unit tree in what order? Depth-first? Breadth-first?
  // 2. For each unit, where is its center?
  // 3. How are shapes positioned relative to parent?
  // 4. Return what data structure?
}

Q: How is center calculated?
Q: How are child positions determined?
Q: What is the output format?
```

**Step 3: Symmetry Application**
```
Input: Assembled shapes (half-geometry)
Output: Full geometry with mirroring

Algorithm needed:
function applySymmetry(halfGeometry: Path[]): Path[] {
  // For each path:
  //   - Keep right half as-is
  //   - Generate left half by mirroring
  //   - Handle center-line points how?
  //   - Connect mirrored portions how?

  // Need: Exact algorithm
}
```

**Step 4: Palette Resolution**
```
Input: Shapes with palette slot references + unit tree
Output: Shapes with resolved colors

Algorithm needed:
function resolvePalette(
  shape: Shape,
  unit: Unit,
  side: 'left' | 'right'
): Color {
  // Tree-walking algorithm:
  // 1. Start at current unit
  // 2. Check if slot is filled
  // 3. If not, walk to parent
  // 4. Repeat until filled slot found
  // 5. What if root palette has empty slot? (Should never happen?)

  // For L/R variation:
  // 1. First walk tree looking for left color
  // 2. If not found, walk tree looking for right color

  // Need: Exact step-by-step algorithm
}
```

**Step 5: Lighting Application**
```
Input: Shapes with resolved colors + light direction + z-height
Output: Shapes with gradient colors applied

Algorithm needed:
function applyLighting(
  shape: Shape,
  baseColor: Color,
  lightDirection: Vector2D,
  zHeight: number
): GradientShape {
  // How is gradient calculated?
  // What is the math for light → gradient?
  // How does z-height affect lighting?
  // Is it per-shape or per-vertex?

  // Need: Complete lighting model specification
}

Q: What is the lighting model? Diffuse? Specular? Simple gradient?
Q: How is light direction specified? Angle? Vector?
Q: What is z-height in units? Pixels? Grid units? Abstract?
```

**Step 6: Tri-Pixel Rasterization**
```
Input: Vector shapes
Output: Pixel buffer with triangular pixel aesthetic

Algorithm needed:
function rasterizeToTriPixels(
  shapes: Shape[],
  triPixelSize: number,
  canvasSize: Size
): PixelBuffer {
  // 1. Determine tri-pixel grid overlay on canvas
  // 2. For each tri-pixel:
  //    - Determine if shape covers it
  //    - Sample color (how?)
  //    - Apply anti-aliasing (how?)
  // 3. Render tri-pixel as multiple regular pixels

  // Need: Complete rasterization algorithm
}

Q: How is coverage determined? Point sampling? Area sampling?
Q: How is anti-aliasing applied?
Q: Are tri-pixels rendered as filled triangles or outlined?
```

**Step 7: Rotation Generation**
```
Input: Single rendered image
Output: 6 or 12 rotated variants

Algorithm needed:
function generateRotations(
  baseImage: PixelBuffer,
  angleCount: 6 | 12
): PixelBuffer[] {
  // For each angle:
  //   - Rotate geometry by angle
  //   - Re-render with rotated geometry
  //   - Or rotate rasterized image?
  //   - Or rotate light direction?

  // Need: Clarification on what rotates
}

Q: Does geometry rotate, or light direction rotates, or both?
Q: Does symmetry axis rotate with geometry?
Q: Are rotations applied to vector data or rasterized image?
```

**Step 8: Sprite Sheet Export**
```
Input: Array of rendered images (base + rotations + variants)
Output: Single PNG with all sprites + metadata JSON

Algorithm needed:
function layoutSpriteSheet(
  sprites: PixelBuffer[],
  metadata: SpriteMetadata[]
): { image: PNG, metadata: JSON } {
  // 1. Determine sheet size (how?)
  // 2. Layout sprites (algorithm?)
  // 3. Generate metadata (format?)

  // Need: Layout algorithm and metadata schema
}

Q: What layout algorithm? Grid? Bin packing? Custom?
Q: What metadata format? (Feature #11 describes this)
```

**Blocker Level**: HIGH - Need detailed algorithms for each step

---

## HIGH PRIORITY: Missing Algorithms

### 12. Center Alignment Algorithm - UNDEFINED

**Current State**: "All components overlaid unscaled with centers exactly aligned"

**Need to Specify**:

```typescript
function calculateUnitCenter(unit: Unit): Point {
  // Option A: Bounding box center
  const bounds = calculateBoundingBox(unit.shape);
  return {
    x: (bounds.minX + bounds.maxX) / 2,
    y: (bounds.minY + bounds.maxY) / 2
  };

  // Option B: Centroid of all points
  const points = getAllPoints(unit.shape);
  return calculateCentroid(points);

  // Option C: Explicit center property
  return unit.center ?? calculateBoundingBox(unit.shape).center;

  // Which one?
}

// How are child units positioned?
function positionChildUnits(parent: Unit, children: Unit[]): PositionedUnit[] {
  const parentCenter = calculateUnitCenter(parent);

  return children.map(child => {
    const childCenter = calculateUnitCenter(child);

    // All centers align to parent's center:
    const offset = {
      x: parentCenter.x - childCenter.x,
      y: parentCenter.y - childCenter.y
    };

    return {
      unit: child,
      position: offset
    };
  });
}

// Edge cases:
// 1. Empty shape (no paths) - what is center?
// 2. Unit with child units - does child affect parent center calculation?
// 3. Paths exactly on center line - do they affect center?
```

**Questions**:
- How is center calculated? Bounding box? Centroid? Explicit property?
- If unit has empty shape, what is its center?
- Do child units affect parent's center calculation?
- Are centers calculated recursively or only for leaf nodes?

**Blocker Level**: HIGH - Needed for Phase 2

---

### 13. Palette Tree-Walking Algorithm - PARTIALLY SPECIFIED

**Current State**: "Color resolution walks up the unit tree"

**Need to Specify**:

```typescript
function resolveColor(
  unit: Unit,
  slotIndex: number,
  side: 'left' | 'right'
): Color {
  // Specification says:
  // 1. Walk up tree to find first non-empty slot
  // 2. For left: first try left, then try right

  // Pseudo-code:
  let currentUnit: Unit | null = unit;

  // If looking for left color, try left slots first
  if (side === 'left') {
    while (currentUnit !== null) {
      const leftColor = currentUnit.palette.leftSlots?.[slotIndex];
      if (leftColor !== null && leftColor !== undefined) {
        return leftColor;
      }
      currentUnit = currentUnit.parent;  // Q: How to get parent?
    }

    // Not found in left, fall back to right
    currentUnit = unit;
  }

  // Try right slots
  while (currentUnit !== null) {
    const rightColor = currentUnit.palette.rightSlots[slotIndex];
    if (rightColor !== null && rightColor !== undefined) {
      return rightColor;
    }
    currentUnit = currentUnit.parent;
  }

  // Should never reach here if root palette has all slots filled
  throw new Error('Palette resolution failed - root palette has empty slot');
}
```

**Questions**:
- How does unit know its parent? Parent reference? Walk from root?
- What if root palette has empty slot? (Should be forbidden, but what if data is corrupted?)
- Is `null` different from `undefined` for empty slots?
- Does the algorithm cache results for performance?

**Blocker Level**: MEDIUM - Well-specified but needs implementation details

---

### 14. Grid Snapping Algorithm - UNDEFINED

**Current State**: "Control points snap to triangular lattice"

**Need to Specify**:

```typescript
function snapToGrid(pixelPoint: Point, gridScale: number): Point {
  // Given a pixel coordinate, find nearest grid point

  // If using hexagonal axial coordinates:
  // 1. Convert pixel to axial (q, r)
  // 2. Round to nearest integer axial coordinates
  // 3. Convert back to pixel

  // Need: Complete algorithm with math

  // Questions:
  // - What is "nearest"? Euclidean distance?
  // - How to handle points equidistant from multiple grid points?
  // - Is there a snapping threshold, or always snap?
}

// Also need:
function pixelToGridCoordinate(pixel: Point, gridScale: number): Point {
  // Convert pixel (x, y) to grid (q, r) or whatever coordinate system
  // Need: Exact formula
}

function gridCoordinateToPixel(grid: Point, gridScale: number): Point {
  // Convert grid coordinates to pixel coordinates
  // Need: Exact formula
}
```

**Questions**:
- What is the exact snapping algorithm?
- How are pixel coordinates converted to/from grid coordinates?
- What coordinate system is used internally?

**Blocker Level**: HIGH - Needed for Phase 1

---

### 15. Negative Space Rendering - UNDER-SPECIFIED

**Current State**: "Negative paths create transparent cutouts"

**Need to Specify**:

```typescript
function renderShapeWithNegativeSpace(shape: Shape): PixelBuffer {
  // Specification says:
  // - Negative paths subtract from positive paths
  // - Multiple negative paths can combine

  // Questions:
  // 1. Rendering order?

  // Option A: Positive first, then subtract all negatives
  const positivePaths = shape.paths.filter(p => !p.isNegative);
  const negativePaths = shape.paths.filter(p => p.isNegative);

  let result = unionAll(positivePaths);  // Union of all positive
  result = subtractAll(result, negativePaths);  // Subtract all negative

  // Option B: Process in path order
  let result = emptyPath();
  for (const path of shape.paths) {
    if (path.isNegative) {
      result = subtract(result, path);
    } else {
      result = union(result, path);
    }
  }

  // Which one?

  // 2. Do negative paths only affect same shape, or can they cut through child units?
  // 3. What happens with overlapping negative paths?
  //    - Union (OR operation)?
  //    - Keep both cuts?
  // 4. What if negative path is outside all positive paths? (no effect?)
}
```

**Questions**:
- What is the exact boolean operation order?
- Do all positives union first, or are paths processed in order?
- Do negative paths only affect their parent shape?
- How do overlapping negative paths interact?

**Blocker Level**: MEDIUM - Needed for Phase 2

---

## HIGH PRIORITY: File Format & Persistence

### 16. Complete JSON Schema - NEEDED

**Current State**: "JSON (TBD exact schema)"

**Need**: Complete, unambiguous schema definition.

**Proposed Schema** (needs validation):

```typescript
// ===== COMPLETE TYPE DEFINITIONS =====

type AssetFile = {
  version: '1.0.0';           // Semantic version
  metadata: {
    name: string;             // Asset name
    created?: string;         // ISO 8601 timestamp
    modified?: string;
    author?: string;
  };
  config: {
    gridScale: number;        // Triangle side length in pixels (power of 2)
    canvasRadius?: number;    // Hexagon radius in pixels
  };
  rootUnit: Unit;             // Root of unit tree
  library?: ComponentLibrary; // Optional component library
};

type Unit = {
  id: string;                 // UUID or unique identifier
  name?: string;              // Display name
  shape: Shape;               // Exactly one shape (may be empty)
  children: Unit[];           // Child units, ordered for Z-order
  palette: Palette;           // Color palette
  metadata?: {
    locked?: boolean;         // Locked from editing?
    visible?: boolean;        // Hidden in editor/export?
  };
};

type Shape = {
  paths: Path[];              // Array of paths (empty array = empty shape)
};

type Path = {
  points: Point[];            // Ordered vertices (in grid coordinates)
  closed: boolean;            // If true, connect last to first
  isNegative: boolean;        // If true, subtracts from positive paths
  stroke?: PaletteSlotRef;    // Optional stroke color reference
  fill?: PaletteSlotRef;      // Optional fill color reference
  strokeWidth?: number;       // Stroke width in pixels (default: 1)
};

type Point = {
  q: number;                  // Hex axial q coordinate (integer)
  r: number;                  // Hex axial r coordinate (integer)
  // OR if using different coordinate system:
  // x: number;
  // y: number;
  // ^^^ NEED TO DECIDE
};

type Palette = {
  slots: PaletteSlot[];       // Length = PALETTE_SLOT_COUNT (TBD: 8?)
};

type PaletteSlot = {
  right: Color | null;        // Right-side color (null = inherit from parent)
  left?: Color | null;        // Left-side color (undefined = use right; null = inherit)
};

type Color = {
  r: number;                  // Red: 0-255 (integer)
  g: number;                  // Green: 0-255 (integer)
  b: number;                  // Blue: 0-255 (integer)
  a: number;                  // Alpha: 0-255 (integer, 255 = opaque)
};

type PaletteSlotRef = {
  slot: number;               // 0 to (PALETTE_SLOT_COUNT - 1)
  side?: 'left' | 'right';    // Default: 'right'
};

type ComponentLibrary = {
  [componentId: string]: Unit;  // Reusable components by ID
};

// ===== VALIDATION RULES =====

// 1. Root unit's palette must have all slots filled (no nulls)
// 2. All unit IDs must be unique within file
// 3. No circular references in unit children
// 4. gridScale must be power of 2
// 5. Palette slot references must be in range [0, PALETTE_SLOT_COUNT)
// 6. Component library IDs must not conflict with unit tree IDs
// 7. Points should be in half-geometry form (q >= 0? or x >= 0?)

```

**Questions**:
- Is this schema correct and complete?
- What validation rules are enforced on load?
- What happens if file has unknown fields? (Ignore for forward compatibility?)
- How are schema versions handled for evolution?
- **CRITICAL**: What is PALETTE_SLOT_COUNT? Still TBD!

**Blocker Level**: CRITICAL - Cannot implement save/load without this

---

### 17. Component Library Storage - UNDEFINED

**Current State**: "Library-based workflow" mentioned, storage not specified

**Options**:

**Option A: Embedded in Asset File**
```json
{
  "version": "1.0.0",
  "rootUnit": { ... },
  "library": {
    "component-123": { "id": "component-123", ... },
    "component-456": { "id": "component-456", ... }
  }
}
```
Pros: Self-contained file
Cons: Can't share components across assets

**Option B: Separate Library File**
```json
// asset.json
{
  "version": "1.0.0",
  "rootUnit": { ... },
  "imports": [
    "components/character-base.json",
    "components/clothing.json"
  ]
}

// components/character-base.json
{
  "version": "1.0.0",
  "components": {
    "body": { ... },
    "head": { ... }
  }
}
```
Pros: Reusable components
Cons: File dependencies, harder to manage

**Option C: Inline References (Units by Value)**
```json
// No separate library
// Child units are always fully inlined
// No component reuse
```

**Questions**:
- Which option is used?
- If Option B, how are import paths resolved?
- Can components reference components from different files?
- How are component updates propagated to uses?

**Blocker Level**: MEDIUM - Needed for Phase 2

---

## HIGH PRIORITY: UI Behavior Specifications

### 18. Editor Tool Behavior - UNDEFINED

**Current State**: No specification of editing tools

**Need to Specify**:

**A. Point Manipulation Tool**
```
Behavior:
- Click on empty space: Add new point? Do nothing?
- Click on existing point: Select it
- Drag selected point: Move it (with grid snapping)
- Click off-point: Deselect

Questions:
- Can you multi-select points?
- How do you delete a point? (Backspace? Del? Right-click menu?)
- Can you select mirrored (left-side) points, or are they always locked?
- What visual feedback for selected points?
```

**B. Path Creation Tool**
```
Behavior:
- Click: Add point to current path
- Double-click or Enter: Finish path
- Esc: Cancel path creation

Questions:
- How do you close a path? (Click first point again? Checkbox?)
- How do you create a new path vs. editing existing?
- Can you add points to existing path?
```

**C. Palette Editor**
```
Behavior:
- For each slot: Color picker or "inherit" checkbox
- Left/right tabs or split view?

Questions:
- How do you set slot to "empty"? (Checkbox? Clear button?)
- How do you assign path to palette slot? (Dropdown? Direct color picker?)
- How is inheritance visualized? (Gray out inherited slots? Show source unit?)
```

**D. Unit Tree Editor**
```
Behavior:
- Tree view showing unit hierarchy
- Drag-and-drop to reorder?
- Click to select unit?

Questions:
- How do you create new child unit?
- How do you move units in tree?
- How do you delete units?
- How is Z-order shown and edited?
```

**Blocker Level**: MEDIUM - Needed for Phase 1+, but can be designed during implementation

---

### 19. Keyboard Shortcuts - UNSPECIFIED

**Need to Define**:

Common expectations:
- Ctrl+Z: Undo
- Ctrl+Y or Ctrl+Shift+Z: Redo
- Ctrl+S: Save
- Ctrl+O: Open
- Delete or Backspace: Delete selected
- Ctrl+C / Ctrl+V: Copy / Paste
- Ctrl+D: Duplicate
- G: Toggle grid visibility?
- H: Toggle symmetry preview?

**Questions**:
- What are the standard keyboard shortcuts?
- Are they customizable?
- What about modifier key behaviors? (Shift = constrain, Alt = duplicate, etc.)

**Blocker Level**: LOW - Can be defined during implementation

---

## MEDIUM PRIORITY: Algorithm Details

### 20. Sprite Sheet Layout Algorithm - UNDEFINED

**Current State**: Feature #11 mentions sprite sheet metadata but not layout

**Need to Specify**:

```typescript
type SpriteSheetLayout = {
  sheetWidth: number;         // Power of 2? (512, 1024, 2048?)
  sheetHeight: number;
  padding: number;            // Pixels between sprites
  sprites: SpriteFrame[];
};

type SpriteFrame = {
  id: string;                 // Sprite identifier
  x: number;                  // Position in sheet
  y: number;
  width: number;              // Sprite dimensions
  height: number;
  rotation?: number;          // For rotation variants
  sourceUnit?: string;        // Reference to source unit ID
};

function layoutSpriteSheet(sprites: RenderedSprite[]): SpriteSheetLayout {
  // Algorithm options:

  // Option A: Simple grid layout
  // - Arrange sprites in rows/columns
  // - Sheet size = smallest power-of-2 that fits all sprites

  // Option B: Bin packing
  // - Use rectangle packing algorithm for efficiency
  // - More complex but better space utilization

  // Option C: Fixed-size grid with explicit positioning
  // - User specifies grid dimensions
  // - Sprites placed at fixed positions

  // Which one?
}
```

**Questions**:
- What layout algorithm? Grid, bin packing, user-defined?
- How is sheet size determined? Fixed? Dynamic?
- What padding between sprites?
- Are rotation variants grouped together?
- How are sprite IDs generated?

**Blocker Level**: MEDIUM - Needed for Phase 3

---

### 21. Automatic Palette Generation Algorithm - UNDEFINED

**Current State**: "Hexadic color scheme" mentioned, no algorithm

**Need to Specify**:

```typescript
function generatePalette(baseColor: Color): Palette {
  // Hexadic = 6 colors evenly spaced on color wheel (60° apart)

  // Algorithm:
  // 1. Convert baseColor to HSV
  const hsv = rgbToHsv(baseColor);

  // 2. Generate 6 hues at 60° intervals
  const hues = [
    hsv.h,
    (hsv.h + 60) % 360,
    (hsv.h + 120) % 360,
    (hsv.h + 180) % 360,
    (hsv.h + 240) % 360,
    (hsv.h + 300) % 360
  ];

  // 3. For each hue, generate shades?
  // How many shades? What saturation/value adjustments?

  // 4. Fill palette slots
  // If palette has 8 slots and hexadic gives 6 colors, what about slots 7-8?

  // Need: Complete algorithm
}
```

**Questions**:
- Exact color theory algorithm?
- How many shades per color?
- How to fill palette if PALETTE_SLOT_COUNT ≠ 6?
- What if base color is grayscale (no hue)?

**Blocker Level**: LOW - Nice-to-have feature

---

### 22. Lighting Gradient Calculation - UNDER-SPECIFIED

**Current State**: "Light-to-dark gradient on shapes"

**Need to Specify**:

```typescript
type LightConfig = {
  direction: Vector2D;        // Light direction (normalized)
  intensity: number;          // 0-1, how strong is lighting effect
  ambient: number;            // 0-1, minimum brightness
  zHeight: number;            // Height of asset center
};

function applyLighting(
  shape: Shape,
  baseColor: Color,
  light: LightConfig
): GradientShape {
  // What is the lighting model?

  // Option A: Simple directional gradient
  // - Perpendicular to light direction = dark
  // - Aligned with light direction = light
  // - Linear interpolation between ambient and full brightness

  // Option B: Simulated 3D diffuse lighting
  // - Calculate surface normals from shape geometry
  // - Apply Lambert's cosine law
  // - More complex but more realistic

  // How does z-height factor in?
  // - Higher z = more pronounced lighting?
  // - Or fixed lighting regardless of height?

  // Need: Exact mathematical model
}
```

**Questions**:
- What lighting model? Simple gradient or simulated 3D?
- How is light direction specified? Angle? Vector?
- How does z-height affect lighting?
- Is lighting per-shape or per-vertex?
- How does lighting interact with palette colors?

**Blocker Level**: LOW - Nice-to-have feature, Phase 3

---

## MEDIUM PRIORITY: Edge Cases & Error Handling

### 23. Invalid Data Handling - UNSPECIFIED

**Questions**:

**A. File Loading**
- What if JSON is malformed?
- What if required fields are missing?
- What if version is newer than editor supports?
- What if palette slot references are out of range?
- What if unit IDs are not unique?
- What if circular references exist in unit tree?

**B. Palette Resolution**
- What if root palette has empty slots? (Should be forbidden)
- What if palette slot reference is out of bounds?
- What if tree-walking reaches root without finding color?

**C. Geometry**
- What if path has < 2 points? (Invalid path)
- What if path has self-intersections?
- What if shape bounds exceed canvas?
- What if no shapes are visible (all empty or negative-only)?

**D. Grid**
- What if gridScale is not power of 2?
- What if gridScale is 0 or negative?
- What if point coordinates are NaN or Infinity?

**Need**: Error handling strategy for each case.
- Throw error? Show warning? Auto-correct? Ignore?

**Blocker Level**: MEDIUM - Needed for robustness

---

### 24. Performance Limits - UNSPECIFIED

**Current State**: "Expected complexity: Dozens of units, each with dozens of vertices"

**Need to Specify**:

**Hard Limits:**
- Maximum points per path: ?
- Maximum paths per shape: ?
- Maximum units in tree: ?
- Maximum tree depth: ?
- Maximum palette slots: ? (Partially defined as TBD)
- Maximum canvas size: ?
- Maximum grid scale: ?

**Performance Targets:**
- Preview update latency: < 16ms (60fps)? < 33ms (30fps)?
- Maximum file size: 1MB? 10MB?
- Maximum undo history depth: 100 operations?

**Validation:**
- Are limits enforced at runtime?
- What happens when limit is exceeded? Error? Warning? Silent clip?

**Blocker Level**: LOW - Can be determined during testing

---

## LOW PRIORITY: Polish & UX Details

### 25. Visual Feedback & UI Polish - UNSPECIFIED

**Need to Define**:

**A. Selection Visual Feedback**
- Selected points: Color? Border? Size change?
- Selected unit in tree: Highlight? Border?
- Mirrored (locked) half: Dimmed? Different color? Dashed lines?

**B. Palette Inheritance Visualization**
- Inherited slots: Gray text? Special icon?
- Slot source: Tooltip showing source unit?
- Empty slots: Checkered pattern? Special indicator?

**C. Grid Visualization**
- Grid lines: Color? Thickness? Opacity?
- Grid points: Dots? Small circles? Crosses?
- Adaptive zoom: Hide grid when zoomed out?

**D. Center Alignment Visualization**
- Show unit centers: Crosshair? Dot?
- Show alignment guides: Lines connecting centers?

**E. Negative Space Visualization**
- Negative paths: Different color? Dashed lines? Special icon?
- Result preview: Show cutouts in real-time?

**Blocker Level**: LOW - Can be designed during implementation

---

### 26. Undo/Redo Granularity - UNSPECIFIED

**Current State**: "Track editing operations"

**Need to Define**:

**What constitutes an undoable action?**

**Option A: Per-operation granularity**
- Move point: One action
- Add point: One action
- Change color: One action
- Very fine-grained, many undo steps

**Option B: Per-gesture granularity**
- Move point drag: One action (entire drag, not each pixel)
- Draw path: One action (entire path, not each point)
- Fewer undo steps, more intuitive

**Option C: Coarse granularity**
- Edit session on a unit: One action
- Palette edit: One action
- Fewer steps, less precise

**Questions**:
- What is the undo granularity?
- Are there undo checkpoints (e.g., after save)?
- Can you undo across file load?
- How many undo levels?

**Blocker Level**: LOW - Nice-to-have feature

---

## DECISIONS STILL NEEDED

### Summary of Critical Decisions

**MUST DECIDE BEFORE PHASE 0:**

1. **Point coordinate system** - Cartesian? Axial? Custom?
2. **Grid coordinate math** - Need explicit formulas
3. **Path data structure** - Bezier curves or polygons only?
4. **Half-geometry storage rules** - Which half? Center line handling?
5. **Palette slot count** - 4? 8? 16?
6. **Color component range** - 0-255 or 0-1?
7. **JSON file format schema** - Complete type definitions

**MUST DECIDE BEFORE PHASE 1:**

8. **Grid scale storage** - Grid units or pixel units?
9. **Z-ordering model** - Separate lists or interleaved?
10. **Center calculation method** - Bounding box, centroid, or explicit?
11. **Stroke width units** - Pixels? Grid units?

**MUST DECIDE BEFORE PHASE 2:**

12. **Component library storage** - Embedded or separate files?
13. **Negative space render order** - All positives first, or sequential?
14. **Palette resolution caching** - Cache results or compute on-demand?

**MUST DECIDE BEFORE PHASE 3:**

15. **Sprite sheet layout algorithm** - Grid, bin packing, or user-defined?
16. **Lighting model** - Simple gradient or simulated 3D?
17. **Rotation application** - To geometry, light, or both?

---

## RECOMMENDATIONS

### Immediate Actions

**For Phase 0 to Begin:**

1. **Define Point type with exact coordinate system**
   - Recommend: Hexagonal axial (q, r) for triangular grid
   - Provide explicit conversion formulas

2. **Define Path and Shape types**
   - Recommend: Polygons only (no Bezier), simplifies rasterization
   - Define minimum point count (recommend: 3 for closed, 2 for open)

3. **Specify half-geometry storage**
   - Recommend: Store right half (q ≥ 0), points at q=0 stored once
   - Document edge cases explicitly

4. **Decide palette slot count**
   - Recommend: 8 slots (good balance)
   - Can expand to 16 later if needed

5. **Complete JSON schema**
   - Use recommended schema from section #16
   - Add validation rules

6. **Document coordinate conversion formulas**
   - Pixel ↔ Grid coordinate conversion
   - Grid ↔ Hexagonal axial conversion
   - Include code examples

### Medium-Term Actions

**Before Phase 1:**

7. Define grid scale storage format (recommend: grid units)
8. Clarify Z-ordering model (recommend: separate shape/children lists)
9. Specify center calculation (recommend: bounding box center)
10. Define stroke width units (recommend: output pixels)

**Before Phase 2:**

11. Choose component library model (recommend: embedded for simplicity)
12. Specify negative space algorithm (recommend: all positives union first)
13. Design palette editor UI
14. Define unit tree editor UX

### Low-Priority Actions

**Before Phase 3:**

15. Design sprite sheet layout (recommend: simple grid for Phase 3)
16. Specify lighting model (recommend: simple directional gradient)
17. Define automatic palette generation algorithm

---

## CONCLUSION

**Current Specification Completeness**: ~40%

**Blockers to Implementation**:
- 7 CRITICAL gaps (data structures, coordinate system, file format)
- 15 HIGH priority gaps (algorithms, behaviors)
- 10 MEDIUM priority gaps (advanced features)
- 5 LOW priority gaps (polish)

**To Reach 100% Specification**:
- Resolve all CRITICAL gaps (required for any coding)
- Resolve HIGH priority gaps (required for each phase)
- Document all algorithms step-by-step
- Define all data structures with TypeScript types
- Specify all edge cases and error handling
- Provide mathematical formulas for coordinate systems

**Recommendation**: Focus on the 7 CRITICAL decisions listed above. Once those are resolved, Phase 0 can begin with a partial but unambiguous specification.
