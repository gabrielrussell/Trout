# Trout Specification - Second Review

## Executive Summary

**Overall Assessment**: The specification has improved dramatically from v1.0 to v2.0. The Architecture Overview section answers most critical path questions, and the feature prioritization provides clear implementation guidance.

**Major Improvements**:
- ✅ Platform decision made (client-side web app)
- ✅ File format decision made (JSON)
- ✅ Coordinate system defined (triangular grid)
- ✅ Data model defined (unit tree structure)
- ✅ Palette system fully specified (tree-walking inheritance with L/R variation)
- ✅ Symmetry model clarified (render-time bilateral only)
- ✅ Clear feature prioritization and phasing

**Status**: Specification is approaching implementation-ready for Phase 0-1. Some details still need clarification for Phase 2-3 features.

---

## Critical Issues (Must Resolve Before Coding)

### 1. Vector Path Data Structure Undefined

**Issue**: The specification mentions "vector paths" and "shapes" extensively but never defines the actual data structure.

**Questions**:
- What is a "path"? Array of points? `{x: number, y: number}[]`?
- Are paths always closed loops, or can they be open?
- Do paths support curves (Bezier/quadratic), or only straight line segments?
- How is the "shape" boundary defined - implicit from point order, or explicit?

**Impact**: This is foundational - affects file format, rendering algorithm, editor UI, everything.

**Recommendation**: Add to Architecture Overview:
```typescript
// Proposed data structure
type Point = { x: number; y: number }; // In grid coordinates
type Path = {
  points: Point[];           // Ordered vertices
  closed: boolean;           // If true, connect last to first
  isNegative: boolean;       // For Feature #10 negative space
};
type Shape = {
  paths: Path[];             // Multiple paths can compose a shape
  fill?: PaletteSlotRef;     // Optional fill color reference
  stroke?: PaletteSlotRef;   // Optional stroke color reference
  strokeWidth?: number;      // In pixels
};
```

---

### 2. Grid Coordinate System Ambiguity

**Issue**: Grid is described as "triangular/hexagonal lattice" but the actual coordinate math is undefined.

**Questions**:
- What coordinate system? Axial coordinates? Cube coordinates? Offset coordinates?
- How are triangular grid points addressed numerically?
- What's the relationship between grid coordinates and pixel coordinates?
- Does the grid use "pointy-top" or "flat-top" hexagons (affects orientation)?

**Example Issue**:
```
If I place a point at grid position (0, 0), where is it in pixel space?
If I move to the next grid point, what are its coordinates?
```

**Impact**: Cannot implement grid snapping or rendering without this.

**Recommendation**: Define explicit coordinate system with diagrams. Suggest:
- Axial coordinates (q, r) for hexagons
- Map triangles to hex vertices
- Provide pixel conversion formula: `pixelX = size * (sqrt(3) * q + sqrt(3)/2 * r)`

---

### 3. "Half-Geometry Storage" Contradicts "Fixed Vertical Center Line"

**Issue**: Specification says:
- "Half-Geometry Storage: Vector paths are stored un-mirrored (only half the shape)"
- "Fixed vertical center line (not movable/rotatable)"

**Problem**: If the symmetry line is vertical and centered, and you store the right half, what happens to points that fall ON the line?

**Questions**:
- Which half is stored - left or right?
- Are center-line points stored once or duplicated?
- If a path crosses the center line multiple times, how is it segmented?

**Example Edge Case**:
```
Shape: Rectangle from x=-10 to x=+10
If storing right half only, you'd store x=0 to x=+10
But the left edge (x=0) belongs to BOTH halves
```

**Recommendation**: Clarify in Architecture Overview:
- "Store right half (x ≥ 0) including points exactly on symmetry axis (x = 0)"
- "Points at x=0 are stored once and rendered once (not mirrored)"
- "During editing, left half (x < 0) is locked/read-only"

---

### 4. Unit Tree Z-Ordering vs. Shape Ordering Confusion

**Issue**: Two different ordering concepts are mentioned:
- "Z-ordering determined by list position (later items obscure earlier ones)" [line 22]
- "Child units and shapes can be arbitrarily ordered" [line 126]

**Questions**:
- Is there a separate ordering for shapes WITHIN a unit vs. child units?
- Can shapes and child units be interleaved in a single ordered list?
- If a unit has 2 shapes and 2 child units, how are they ordered?

**Recommendation**: Clarify data structure. Suggest either:

**Option A: Separate lists**
```typescript
type Unit = {
  shapes: Shape[];      // Ordered, drawn first to last
  children: Unit[];     // Ordered, drawn first to last
  // All shapes drawn, then all children drawn
};
```

**Option B: Interleaved list**
```typescript
type RenderItem = Shape | Unit;
type Unit = {
  items: RenderItem[];  // Shapes and child units interleaved, drawn in order
};
```

---

### 5. "All Centers Aligned" Positioning Model Incomplete

**Issue**: Specification says "Fixed positioning: All components overlaid unscaled with centers exactly aligned" but doesn't define what "center" means.

**Questions**:
- Is center calculated from bounding box? Centroid? Explicit center point property?
- If a component has no shapes (empty shape), where is its center?
- How is center calculated when component has child units at different positions?
- Can user manually set the center point, or is it always computed?

**Example Problem**:
```
Unit A: Shapes spanning x=-10 to x=+10, y=-5 to y=+5
  Bounding box center: (0, 0)
Unit B: Shapes spanning x=0 to x=20, y=0 to y=10
  Bounding box center: (10, 5)

When B is a child of A, where does B actually render?
```

**Recommendation**: Define center calculation explicitly. Suggest:
- "Center = bounding box center of all shapes in THIS unit (excluding child units)"
- "Empty-shape units inherit center from parent"
- OR "Each unit has an explicit `center: Point` property"

---

## High Priority Issues (Needed for Phase 1-2)

### 6. Palette Slot Count Not Specified

**Status**: Mentioned as open question but never answered.

**Question**: How many palette slots exist per palette? (Feature #6 asks "2? 4? 8? 16?")

**Impact**:
- Affects UI design (how many color pickers to show)
- Affects file format (palette array size)
- Affects rendering performance (number of lookups)

**Recommendation**: Make a decision. Suggest:
- **Start with 8 slots** - good balance of flexibility vs. complexity
- Can expand to 16 later if needed
- Document as: "Each palette has exactly 8 slots (indexed 0-7)"

---

### 7. Stroke Width Units Undefined

**Issue**: Feature #6 mentions "Stroke width configurable per path" but units not specified.

**Questions**:
- Is stroke width in pixels, grid units, or percentage of shape size?
- Is stroke width affected by zoom level, or constant in screen space?
- What's the default stroke width?
- Is there a minimum/maximum stroke width?

**Recommendation**: Specify units clearly. Suggest:
- "Stroke width in output pixels (not affected by grid scale or zoom)"
- "Default: 1px"
- "Range: 0.5px to 10px"

---

### 8. "Hexagonal Canvas Boundary" Interaction with Centering

**Issue**: If canvas is hexagonal, and all components center-align, how does canvas size adapt?

**Questions**:
- Is canvas size fixed per asset, or does it grow to fit content?
- When composing units, does the canvas size change?
- If canvas is hexagonal, what are its dimensions (radius? side length?)?
- How does the hexagonal canvas relate to rectangular PNG export dimensions?

**Example Problem**:
```
Asset 1: Small character fitting in 64px hex
Asset 2: Giant boss needing 256px hex

When exporting sprite sheet, do all assets use same canvas size?
If not, how is size determined per-asset?
```

**Recommendation**:
- Specify: "Canvas size is per-asset, specified as hexagon radius in pixels"
- Or: "Canvas auto-sizes to content bounding box + padding"
- Define hex-to-rectangle conversion for PNG export

---

### 9. Grid Scale Selection Interaction with Existing Geometry

**Issue**: Specification says grid uses "power-of-2 scaling" (2px, 4px, 8px...) but doesn't say what happens when you change scale mid-editing.

**Questions**:
- If I draw shapes at 2px grid, then switch to 4px grid, do existing points snap to new grid?
- Are points stored in grid coordinates or pixel coordinates?
- Can points exist off-grid if you scale down? (e.g., created at 4px grid, then switch to 8px grid)

**Recommendation**:
- Store points in "abstract grid coordinates" not pixels
- When grid scale changes, points remain at same grid coordinates (visual positions stay the same)
- Document as: "Grid scale affects only NEW points; existing points unchanged"

---

### 10. Tri-Pixel and Grid Alignment Details

**Issue**: Specification says tri-pixels "may differ by one or more powers of 2 in scale" from grid, but rendering algorithm not explained.

**Questions**:
- If grid is 2px and tri-pixels are 8px, how are they aligned?
- Do tri-pixel triangles have same orientation as grid triangles?
- Can tri-pixel size be non-power-of-2? (e.g., grid=4px, tri-pixel=6px?)
- What happens at canvas edges where tri-pixels don't fit evenly?

**Recommendation**: Add to Feature #3:
- "Tri-pixel size must be power-of-2 multiple of grid size (1x, 2x, 4x, 8x...)"
- "Tri-pixel grid perfectly overlays construction grid at integer multiples"
- "Partial tri-pixels at edges are rendered (anti-aliased)"

---

## Medium Priority Issues (Phase 2-3 Features)

### 11. Rotation Generation Details Missing

**Issue**: Feature #4 says "6-way minimum (every 60°), with 12-way (every 30°) option" but mechanism unclear.

**Questions**:
- Does rotation affect the underlying geometry, or just the render output?
- Are rotated versions stored separately, or generated on-demand?
- How does rotation interact with the fixed vertical symmetry axis?
  - If I rotate 60°, is the symmetry axis now diagonal?
  - Or does symmetry axis stay vertical and geometry rotates relative to it?
- How are rotation angles specified - relative to asset's canonical orientation?

**Critical Design Question**:
```
Asset facing "north" with vertical symmetry
Rotate 60° clockwise → now facing "northeast"

Option A: Symmetry axis rotates with asset (now diagonal left-to-right)
Option B: Symmetry axis stays vertical (asset now asymmetric in view)

Which one?
```

**Recommendation**: Clarify rotation model. Suggest:
- "Rotation is render-time only; source geometry unchanged"
- "Symmetry axis rotates with geometry (maintains bilateral symmetry in rotated view)"
- "Rotation angles: 0°, 60°, 120°, 180°, 240°, 300° for 6-way"

---

### 12. Lighting Gradient Direction vs. Rotation

**Issue**: Feature #4 says lighting can be configured per-export, and rotation generates multiple angles, but interaction is unclear.

**Scenario**:
```
Light direction: top-left (135°)
Generate 6 rotation angles

Question: Does light direction rotate with asset, or stay world-fixed?

If rotating to 60°:
  Option A: Light still from top-left (world-fixed) → asset is darker on different side
  Option B: Light rotates to maintain relative direction → always same lighting on same parts
```

**Recommendation**: Specify lighting behavior. Suggest:
- "Light direction is world-fixed (e.g., always from top-left)"
- "As asset rotates, lighting changes relative to asset surfaces"
- "This creates natural top-down game lighting where light source is environmental"

---

### 13. File Format Schema Not Specified

**Issue**: Specification says "JSON (TBD exact schema)" but schema is critical for implementation.

**Recommendation**: Define JSON schema in specification. Suggest starting point:

```typescript
type AssetFile = {
  version: string;              // "1.0"
  name: string;                 // Asset name
  canvasSize: number;           // Hexagon radius in pixels
  gridScale: number;            // Grid triangle side length in pixels (power of 2)
  rootUnit: Unit;               // Root of unit tree
  globalPalette: Palette;       // Root palette with no empty slots
};

type Unit = {
  id: string;                   // Unique identifier
  name: string;                 // Display name
  shape: Shape;                 // Exactly one shape (may be empty)
  children: Unit[];             // Child units (ordered)
  palette: Palette;             // Palette for this unit
  center?: Point;               // Optional explicit center (else computed)
};

type Palette = {
  slots: (Color | null)[];      // Array of 8 slots; null = empty/inherit
  leftOverrides?: (Color | null)[];  // Optional L/R variation (Feature #7)
};

type PaletteSlotRef = {
  slot: number;                 // 0-7
  side?: 'left' | 'right';      // Optional for L/R palette
};

type Shape = {
  paths: Path[];
  fill?: PaletteSlotRef;
  stroke?: PaletteSlotRef;
  strokeWidth?: number;
};

type Path = {
  points: Point[];
  closed: boolean;
  isNegative: boolean;          // Feature #10
};

type Point = {
  q: number;                    // Grid coordinate (axial q)
  r: number;                    // Grid coordinate (axial r)
};

type Color = {
  r: number;                    // 0-255
  g: number;                    // 0-255
  b: number;                    // 0-255
  a: number;                    // 0-1
};
```

---

### 14. Negative Space Path Rendering Order

**Issue**: Feature #10 says "negative paths" create cutouts, but interaction with multiple paths and Z-ordering is unclear.

**Questions**:
- If a shape has paths [A, B, C] where B is negative, is the render order:
  - Fill A, subtract B, fill C? (order-dependent)
  - Fill (A + C), then subtract B? (positives compose first)
- Can negative paths overlap? What happens to doubly-subtracted regions?
- Can negative path create holes in shapes from DIFFERENT units in the tree?

**Recommendation**: Specify clearly. Suggest:
- "Within a shape, all positive paths compose first (union), then all negative paths subtract"
- "Negative paths only affect their parent shape, not other units"
- "Overlapping negative paths create union of subtracted regions (OR operation)"

---

### 15. Sprite Sheet Layout Algorithm Undefined

**Issue**: Export strategy says "sprite sheet focus" but layout algorithm not specified.

**Questions**:
- How are sprites arranged in the sheet? Grid? Packed?
- How is sprite sheet size determined? Fixed size? Grow to fit?
- What padding/spacing between sprites?
- If exporting rotation variants, how are they grouped/laid out?

**Recommendation**: Specify layout algorithm. Suggest:
- "Simple grid layout: sprites arranged left-to-right, top-to-bottom"
- "Sheet size: minimum power-of-2 dimensions to fit all sprites (512x512, 1024x1024, etc.)"
- "Padding: 2px transparent border around each sprite"
- "Rotation sets: grouped horizontally (6 or 12 frames per row)"

---

### 16. Component Library File Organization

**Issue**: Feature #5 says "library-based workflow" but library storage not defined.

**Questions**:
- Is the component library a separate file, or embedded in each asset file?
- How are components referenced? By ID? By file path? By name?
- Can components be shared across projects?
- How is component versioning handled (if you edit a component, do all uses update)?

**Recommendation**: Specify library model. Options:

**Option A: Embedded Library**
- Each asset file contains its own component definitions
- Components referenced by ID within the file
- Pro: Self-contained files
- Con: Can't share components across assets

**Option B: Separate Library Files**
- Component library is separate .json file(s)
- Assets reference components by path or ID
- Pro: Reusable components
- Con: File dependencies to manage

---

## Low Priority Issues (Nice-to-Have Clarifications)

### 17. Undo/Redo Granularity Undefined

**Issue**: Feature #13 mentions undo/redo but doesn't specify what constitutes an "action".

**Questions**:
- Is each point move a separate undoable action, or only when you release mouse?
- Is palette color change immediately undoable, or only after confirmation?
- Are operations grouped (e.g., "Add 5 points" as one action vs. five actions)?

**Recommendation**: Define action granularity. Suggest:
- "Action = one mouse drag completion (e.g., move point from A to B)"
- "Action = one palette color change commit"
- "Action = one UI operation completion (not intermediate states)"

---

### 18. Auto Palette Generation Parameters Underspecified

**Issue**: Feature #12 mentions "hexadic color scheme" but algorithm details missing.

**Questions**:
- What exact algorithm? HSV color wheel with 60° spacing?
- How many shades per color?
- How are shades calculated (darken/lighten by what percentage)?
- If palette has 8 slots but hexadic gives 6 colors, what fills slots 7-8?

**Recommendation**: Specify algorithm precisely or reference a color theory library.

---

### 19. Browser Compatibility and File API Details

**Issue**: Feature #9 says "client-side web app" using "File System Access API" but compatibility concerns.

**Questions**:
- File System Access API is not supported in all browsers (Safari, Firefox have partial/no support)
- Fallback strategy? Use download API + file picker?
- Local storage size limits? (Browser localStorage typically ~5-10MB limit)

**Recommendation**: Specify browser requirements and fallback:
- "Target: Chrome/Edge (File System Access API)"
- "Fallback: Download + file picker for other browsers"
- "Consider IndexedDB for larger project storage (no 5MB limit)"

---

### 20. Real-Time Preview Performance Targets

**Issue**: Architecture says "real-time preview required" but no performance targets.

**Questions**:
- What framerate for preview? 60fps? 30fps? On-change only?
- What's the maximum asset complexity for smooth preview? (# of points, # of units)
- Should editor throttle preview updates during rapid editing?

**Recommendation**: Set performance targets:
- "Target: 30fps preview for assets up to 100 total shapes"
- "Preview updates: Throttled to max 30 updates/second during continuous editing"
- "For larger assets, offer 'low-quality preview' mode (skip anti-aliasing, lower resolution)"

---

## Internal Contradictions

### 21. Simplification Opportunities vs. Feature List

**Issue**: "Simplification Opportunities" section suggests cutting features, but those features appear in final feature list with priorities.

**Examples**:
- Simplification says cut "Feature #4 (Lighting)" - but it's listed as "Nice-to-Have" priority
- Simplification says cut "Nested components" - but Feature #5 explicitly says "recursive/nested assembly"

**Resolution**: Clarify whether "Simplification Opportunities" is:
- Old guidance (from v1.0) now superseded by decisions in v2.0?
- OR still active guidance (in which case update feature list to match)?

**Recommendation**: Remove or update "Simplification Opportunities" section to reflect final decisions.

---

### 22. "No Import Support" vs. Long-Term Goals

**Issue**: Design Decisions say "No import support: Editor does not import existing SVG/PNG assets" but this seems like a significant limitation.

**Question**: Is this a Phase 1 limitation (to keep scope small) or a permanent decision?

**Consideration**: For an editor ecosystem, being unable to import existing work can be a deal-breaker for adoption. Even basic SVG import would greatly help.

**Recommendation**: Clarify:
- "Phase 1-2: No import support (to minimize scope)"
- "Phase 3+: Consider adding SVG path import for existing artwork integration"

---

## Positive Highlights

### What's Working Well

1. **Clear Architectural Foundation**: The TypeScript rendering library as core architecture is excellent - clean separation of concerns.

2. **Unit Tree Model**: The unit-tree data structure is elegant and handles composition well.

3. **Palette Inheritance**: The tree-walking palette system is well-thought-out and handles the trousers/handkerchief example perfectly.

4. **Grid-First Approach**: Committing to triangular grid as primary constraint simplifies many decisions.

5. **Phased Development**: Phase 0-3 breakdown provides realistic implementation path.

6. **Feature Prioritization**: Critical/Medium/Nice-to-Have classification guides development focus.

7. **Symmetry Model**: Render-time symmetry with half-geometry storage is clean and efficient.

---

## Recommendations for Next Steps

### Before Starting Phase 0

1. **Define vector path data structure** (Critical Issue #1)
2. **Define grid coordinate system** with math/formulas (Critical Issue #2)
3. **Clarify half-geometry storage** for center-line points (Critical Issue #3)
4. **Specify unit tree ordering** (shapes vs. children) (Critical Issue #4)
5. **Define center calculation** for alignment (Critical Issue #5)

### Before Starting Phase 1

6. **Decide palette slot count** (#6)
7. **Define JSON file format schema** (#13)
8. **Specify grid scale switching behavior** (#9)
9. **Define stroke width units** (#7)
10. **Specify canvas size model** (#8)

### Before Starting Phase 2

11. **Define component library file organization** (#16)
12. **Specify rotation + symmetry interaction** (#11)
13. **Clarify lighting + rotation interaction** (#12)
14. **Define negative space rendering order** (#14)
15. **Specify sprite sheet layout algorithm** (#15)

---

## Overall Readiness Assessment

**Phase 0 (Proof of Concept)**:
- **Readiness**: 70% - Need Critical Issues #1, #2, #3 resolved
- **Blockers**: Vector path structure, grid coordinates, half-geometry details

**Phase 1 (Actually Usable)**:
- **Readiness**: 60% - Need High Priority Issues #6-10 resolved
- **Blockers**: File format schema, palette slot count, grid scale behavior

**Phase 2 (Component System)**:
- **Readiness**: 50% - Need Medium Priority Issues #11-16 resolved
- **Blockers**: Component library model, tree ordering, center alignment

**Phase 3 (Polish)**:
- **Readiness**: 80% - Most features well-specified
- **Needs**: Performance targets, browser compatibility plan

---

## Conclusion

The v2.0 specification is a massive improvement and provides a solid foundation for implementation. The architecture is sound, and the core concepts are well thought out.

**To proceed with confidence, prioritize resolving:**
1. Vector path data structure definition
2. Grid coordinate system mathematics
3. JSON file format schema

Once these three foundational pieces are defined, Phase 0 development can begin with minimal ambiguity.

The remaining issues are important but can be resolved during implementation or deferred to later phases.

**Overall Grade**: B+ → A- (with critical issues resolved)
- Excellent architectural vision
- Strong feature prioritization
- Some critical implementation details still pending
- Very close to implementation-ready
