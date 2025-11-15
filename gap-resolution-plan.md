# Implementation Gaps - Resolution Plan

## Gaps We Can Resolve from Existing Decisions

### From implementation-decisions.md and previous conversations:

#### 1. Palette Type (Gap #5) - **RESOLVED**
**Decision**: 12 slots per palette, dual palette system (LEFT and RIGHT)
```typescript
type Palette = {
  left: PaletteSlot[];   // 12 slots
  right: PaletteSlot[];  // 12 slots
};

type PaletteSlot = {
  color: Color | null;  // null = inherit from parent
};
```

**Color resolution algorithm**:
- For LEFT side: Walk up tree checking LEFT slots, then fall back to RIGHT slots
- For RIGHT side: Walk up tree checking RIGHT slots only

#### 2. Canvas Size (Gap #7 partial) - **RESOLVED**
**Decision**: Fixed hexagon size of 64 grid units (2^6)
- Never changes
- Composed of 6 equilateral triangles meeting at center (0,0)
- Each triangle has side length 32 (2^5) when canvas is 64 units

#### 3. Grid Scale Behavior (Gap #10) - **RESOLVED**
**Decision**: Changing grid scale does NOT affect existing geometry
- Points remain at their exact positions
- Grid scale only affects snapping for NEW points and point movement

#### 4. Half-Geometry Storage (Gap #9) - **RESOLVED**
**Decision**: Store right half only (x ≥ 0)
- Points exactly on symmetry axis (x = 0) are stored once
- Left half (x < 0) is NOT stored - purely computed at render time
- Left half is NOT editable in editor (read-only visual)

#### 5. Component Positioning (Gap #12 partial) - **RESOLVED**
**Decision**: Fixed center-aligned composition
- ALL units rendered at same center point: (0, 0)
- NO transforms: No translation, rotation, scale
- NO bounding boxes needed
- Implicit alignment: all geometry shares same origin

#### 6. Z-Ordering (Gap #4 partial) - **RESOLVED**
**Decision**: Separate shape and children arrays
```typescript
type Unit = {
  shape: Shape;      // Rendered first
  children: Unit[];  // Rendered after, in array order
  palette: Palette;
  // ...
};
```
Rendering order: Parent shape, then each child (recursively), in children array order.

#### 7. Sprite Sheet Layout (Gap #15, #20) - **RESOLVED**
**Decision**: Grid layout
- Vertical (rows): Different units/assets
- Horizontal (columns): Rotation angles (0°, 60°, 120°, etc.)

#### 8. Undo/Redo Granularity (Gap #26) - **RESOLVED**
**Decision**: Layer list operations only
- **Undoable**: Add/remove path, add/remove child unit, reorder layer list
- **NOT undoable**: Individual point operations (adding/moving/deleting points within a path)

#### 9. Path Type (Gap #2 partial) - **RESOLVED**
**Decision**: Straight line segments only (no Bezier curves)
- Paths can be closed or open
- Support for negative space paths (isNegative flag)
- Stroke and fill reference palette slots

```typescript
type Path = {
  points: Point[];
  closed: boolean;
  isNegative: boolean;        // Feature #10: Negative Space
  stroke?: PaletteSlotRef;    // Optional stroke
  fill?: PaletteSlotRef;      // Optional fill
  strokeWidth?: number;       // TBD: units?
};
```

#### 10. Shape Type (Gap #3) - **RESOLVED**
**Decision**: Simple container for paths
```typescript
type Shape = {
  paths: Path[];  // Empty array = empty shape
};
```
- Multiple paths in same shape union together (before negative space subtraction)
- Paths ordered for rendering

---

## Gaps Requiring New Decisions

### CRITICAL (Need Before Phase 0):

#### A. Point Coordinate System (Gap #1, #8)
**Current Understanding**: Triangular tessellation, flat-topped hexagon orientation

**Need to decide**:
1. **Coordinate system choice**:
   - **Option 1**: Cartesian (x, y) in pixels - Simplest
   - **Option 2**: Cartesian (x, y) in abstract grid units - Scale independent
   - **Option 3**: Hexagonal axial (q, r) - More complex but natural for hex grids

2. **Integer or float coordinates**?
   - Integers only (always on grid)?
   - Floats allowed (can be off-grid)?

3. **Coordinate conversion formulas**:
   - Need explicit: pixel ↔ grid coordinate
   - Need explicit: grid snap algorithm

**Recommendation**: Cartesian (x, y) in pixels with integer coordinates
- Simplest to implement
- Easy to visualize
- Snap-to-grid rounds to nearest valid grid point

#### B. Color Component Range (Gap #5)
**Need to decide**:
```typescript
type Color = {
  r: number;  // 0-255 or 0-1?
  g: number;
  b: number;
  a: number;  // 0-255 or 0-1?
};
```

**Recommendation**: 0-255 integers (standard RGB)
- More intuitive for users
- Standard in most tools
- No floating point rounding issues

#### C. Stroke Width Units (Gap #2)
**Need to decide**: When path has `strokeWidth: 2`, is that:
- 2 pixels in output?
- 2 grid units?
- 2 * gridScale pixels?

**Recommendation**: Pixels in output (simplest, WYSIWYG)

#### D. Path Validation Rules (Gap #2)
**Need to decide**:
- Minimum points for closed path? (Recommend: 3)
- Minimum points for open path? (Recommend: 2)
- Can path have 0 points? (Recommend: No - invalid)
- Can path have self-intersections? (Recommend: Yes, allowed but may render oddly)
- Is winding order significant? (Recommend: No for initial implementation)

#### E. Unit ID System (Gap #4)
**Need to decide**:
```typescript
type Unit = {
  id: string;  // Required? Auto-generated UUID? User-specified?
  // ...
};
```

**Recommendation**:
- Required: Yes (needed for references, debugging)
- Auto-generated: Yes (UUID v4)
- User can optionally provide `name` for display

#### F. PaletteSlotRef Details (Gap #6)
**Need to decide**:
1. Can paths use direct colors instead of palette slots?
2. What happens if slot reference is out of bounds (0-11)?

**Recommendation**:
- No direct colors (palette-only for consistency)
- Out of bounds = validation error on file load
- Include side in reference:
```typescript
type PaletteSlotRef = {
  slot: number;  // 0-11
  side: 'left' | 'right';
};
```

#### G. Complete JSON Schema (Gap #7, #16)
**Need to finalize**:
```typescript
type AssetFile = {
  version: string;  // "1.0.0"
  metadata: {
    name: string;
    created?: string;   // ISO 8601
    modified?: string;
    author?: string;
  };
  config: {
    gridScale: number;      // Current grid scale (power of 2)
    canvasSize: number;     // Fixed at 64 for v1
  };
  rootUnit: Unit;
};
```

### HIGH PRIORITY (Need Before Phase 1):

#### H. Center Calculation (Gap #12)
Since all components are centered at (0,0), do we even need center calculation?
- For editor display: Show geometric center of selected unit?
- For rendering: Not needed (everything at 0,0)

**Need to decide**: Is center calculation needed at all?

**Recommendation**: NOT needed for v1 since everything is at (0,0)

#### I. Negative Space Rendering Order (Gap #15)
**Need to decide**:
- **Option A**: Union all positive paths, then subtract all negative paths
- **Option B**: Process paths in order (can interleave positive/negative)

**Recommendation**: Option A (simpler, more predictable)

#### J. Palette Resolution Edge Cases (Gap #13)
**Need to decide**:
- What if root palette has empty slot? (Should be validation error?)
- How to prevent: Enforce "root must have all 12 LEFT and all 12 RIGHT slots filled" on save?

**Recommendation**: Validation rule: Root unit must have all 24 slots filled (12 LEFT + 12 RIGHT)

### MEDIUM PRIORITY (Need Before Phase 2):

#### K. Component Library (Gap #17)
**Need to decide**:
- Embedded in .trout file?
- Separate library files?
- No library (just copy/paste units)?

**Recommendation**: Start with no library (Phase 1), add embedded library later (Phase 2)

#### L. Rendering Pipeline Details (Gap #11)
Many steps need algorithms defined. Can defer some to implementation.

**Need for Phase 1**:
- Step 3: Symmetry application algorithm
- Step 4: Palette resolution algorithm (mostly defined)

**Can defer to Phase 2+**:
- Step 5: Lighting (nice-to-have feature)
- Step 6: Tri-pixel rasterization (nice-to-have)
- Step 7: Rotation generation (Phase 3)
- Step 8: Sprite sheet export (Phase 3)

### LOW PRIORITY (Can Decide During Implementation):

#### M. Editor Tool Behavior (Gap #18)
- Can design during Phase 1 implementation
- Start with basic: click to add point, drag to move, delete key to remove

#### N. Keyboard Shortcuts (Gap #19)
- Can define during implementation
- Start with standard: Ctrl+Z/Y, Ctrl+S, Delete, Esc

#### O. Visual Feedback (Gap #25)
- Can design during implementation
- Standard selection highlights, grid lines, etc.

#### P. Performance Limits (Gap #24)
- Can determine during testing
- Start with reasonable limits (100 units, 1000 points per path, etc.)

#### Q. Error Handling (Gap #23)
- Can specify per-feature during implementation
- General strategy: Validate on load, show error if invalid

---

## Recommended Decisions Summary

### Critical Decisions Needed Now:

1. **Point coordinates**: Cartesian (x, y) in pixels, integers only
2. **Color components**: 0-255 integers (RGBA)
3. **Stroke width**: Pixels in output
4. **Path validation**: Min 3 points closed, 2 points open, no 0-point paths
5. **Unit IDs**: Auto-generated UUIDs, optional user-facing `name`
6. **Palette refs**: Slot + side, no direct colors, out-of-bounds = error
7. **Root palette**: Must have all 24 slots filled (validation rule)

### High Priority (For Phase 1):

8. **Center calculation**: Not needed (everything at 0,0)
9. **Negative space**: Union positives, then subtract negatives
10. **Component library**: No library in Phase 1

### Can Defer:

- Editor UI details (design during implementation)
- Keyboard shortcuts (standard defaults)
- Performance limits (determine during testing)
- Error messages (define as needed)
- Advanced features (lighting, tri-pixels, etc.)

---

## Questions for User

Before I update the specification, please confirm or adjust these recommendations:

### 1. Coordinate System
Do you want:
- **A**: Cartesian (x, y) in pixels, integers (my recommendation)
- **B**: Cartesian (x, y) in abstract grid units (scale-independent)
- **C**: Something else?

And: What are the exact conversion formulas for snap-to-grid?

### 2. Color Format
- **A**: RGBA 0-255 integers (my recommendation)
- **B**: RGBA 0-1 floats
- **C**: Something else?

### 3. Stroke Width
- **A**: Pixels in final output (my recommendation)
- **B**: Grid units (scales with grid)
- **C**: Something else?

### 4. Center Calculation
Since everything renders at (0,0), do we need center calculation at all?
- **A**: No, not needed (my recommendation)
- **B**: Yes, for editor display purposes

### 5. Negative Space Rendering
- **A**: Union all positives first, then subtract all negatives (my recommendation)
- **B**: Process paths in sequential order

### 6. Root Palette Validation
- **A**: Require all 24 slots filled in root (12 LEFT + 12 RIGHT) (my recommendation)
- **B**: Only require slots that are actually referenced
- **C**: Something else?

### 7. Grid Coordinate Math
This is the most complex one. For a triangular tessellation with flat-topped hexagon:
- What is the exact formula to snap a pixel point (px, py) to nearest grid vertex?
- What is the spacing between grid points?
- Should I propose a specific coordinate system with full math?
