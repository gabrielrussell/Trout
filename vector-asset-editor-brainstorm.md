# Vector Game Asset Editor - Brainstorming Document

## Project Overview
A vector-based game asset editor targeting 2D top-down games with a focus on simplicity. Features will be evaluated for implementation complexity and dropped if they exceed the "simple project" threshold.

---

## Feature Ideas

### 1. Core Symmetry System
**Description:** Built-in symmetry as a fundamental feature rather than an optional tool.

**Key aspects:**
- Default symmetrical editing (mirror across axis/axes)
- Selective symmetry breaking for specific elements/details
- Symmetry as default state, not just a modifier

**Open questions:**
- Is symmetry broken at object level (entire shapes) or point level (individual control points)?
- Is symmetry enforcement live/continuous or operation-based?
- What is the default symmetry axis for new assets?

---

### 2. Triangular Grid Snapping System
**Description:** Construction aid using triangular/hexagonal grid for control point snapping.

**Key aspects:**
- Control points and vertices snap to triangular lattice
- Facilitates easy drawing of triangles and hexagons
- Grid-aligned geometry construction

**Open questions:**
- What is the grid scale/size?
- Is grid snapping always on, or toggleable?
- How does grid relate to final output resolution?

---

### 3. Triangular Pixel Rendering
**Description:** Aesthetic style using large visible triangular "pixels" instead of square pixels.

**Key aspects:**
- Rasterize with chunky triangular pixel look
- Output to standard image formats (PNG, etc.) but with tri-pixel aesthetic
- Each "tri-pixel" rendered as multiple regular pixels forming triangle

**Open questions:**
- What is the tri-pixel size relative to output resolution?
- Do tri-pixels align with construction grid (#2) if both features are used?
- Are tri-pixels anti-aliased or hard-edged?

---

### 4. Faux 3D Lighting
**Description:** Simulate dimensionality through lighting gradients across shapes.

**Key aspects:**
- Apply light-to-dark gradient on shapes
- Auto-generate rotated angle variants with adjusted lighting
- Creates pseudo-3D effect for 2D sprites

**Open questions:**
- Is lighting per-shape or global across scene?
- How many rotation angles are generated? (4-way? 8-way? 16-way?)
- Is angle generation automatic export or manual selection?
- Is light direction configurable per export?

---

### 5. Component/Layer Assembly System
**Description:** Build reusable components and assemble them into complete characters/objects.

**Key aspects:**
- Create library of reusable components (body parts, clothing, accessories)
- Mix and match components to create variants
- Assemble characters from base + additions (naked character + clothing)

**Open questions:**
- Is assembly runtime (editor view, export components) or bake-in (export composite)?
- Can components be positioned/scaled during assembly or use fixed attachment points?
- How are component layers ordered/managed?
- Can components reference other components (nested assembly)?

---

### 6. Dynamic Palette Inheritance
**Description:** Components with mixed coloring - some hard-coded, some inherited from parent.

**Key aspects:**
- Two-tier palette system: shared palette + local/unshared colors
- Child components inherit parent's shared palette
- Enables palette-driven variants without geometry duplication
- Example: trousers with fixed blue fabric but handkerchief color from parent's hair

**Open questions:**
- How many shared palette slots?
- What happens when child is used standalone (no parent to inherit from)?
- Is palette inheritance visualized in editor?
- Can shared palette be overridden at assembly time?

---

### 7. Left/Right Palette Variation
**Description:** Shared palette slots can have different values for left vs right side.

**Key aspects:**
- Enables asymmetric coloring on symmetric geometry
- Child components use left/right palette slots
- Example: red left glove, blue right glove on symmetric character

**Open questions:**
- How are left/right regions defined? (automatic from symmetry axis? manual painting?)
- Does this apply only to shared palette or also local colors?
- Can this support more than bilateral symmetry (e.g., radial with 4+ segments)?

---

### 8. TypeScript Vector Export
**Description:** Export vector data and rendering code as standalone TypeScript file.

**Key aspects:**
- Self-contained executable export format
- Includes point data, shapes, and rasterization logic
- Human-readable and hand-editable
- Ensures long-term asset accessibility
- Can regenerate rasters at build time

**Open questions:**
- Does export include just final geometry or all editor features (symmetry ops, assembly, palettes)?
- Does rendering code implement style features (tri-pixels, lighting) or just basic vector rendering?
- What rendering library/API does the TS code target (Canvas2D, WebGL, generic)?
- How are external dependencies handled?

---

## Open Design Questions

### Scope & Philosophy
1. What defines "super simple" for this project? Line count? Time to implement? Feature count?
2. What is the primary workflow: create many simple assets quickly, or create complex assets with fine control?
3. What game engines/frameworks is this targeting? (affects export format priorities)

### Symmetry Architecture
4. Should symmetry axis be movable/rotatable or fixed?
5. Can multiple symmetry types exist on one asset (e.g., bilateral + radial)?
6. How is symmetry preserved/broken when editing components vs assembled objects?

### Grid System
7. If using triangular grid (#2), does it also define canvas boundaries/alignment?
8. Should there be an option for rectangular grid, or is triangular grid the only option?

### Export & File Format
9. What is the native working file format?
10. What raster export formats are needed? (PNG, WebP, etc.)
11. Should editor support import of existing assets (SVG, PNG)?
12. Is there a need for sprite sheet export (multiple assets in one image)?

### Rendering & Performance
13. Is real-time preview of all features required, or can some be render-time only?
14. What is the expected asset complexity (vertex count, shape count)?
15. Should there be a batch export mode for generating all variants?

---

## Feature Compatibility Matrix

| Feature Pair | Compatible? | Notes |
|--------------|-------------|-------|
| **#1 Symmetry + #2 Tri Grid** | ✓ YES | Symmetry axis aligns to grid, triangles mirror cleanly |
| **#1 Symmetry + #3 Tri Pixels** | ✓ YES | As long as symmetry axis aligns to pixel grid |
| **#1 Symmetry + #4 Lighting** | ✓ YES | Symmetric shapes with symmetric lighting work naturally |
| **#1 Symmetry + #5 Components** | ? COMPLEX | Need to define: do components have symmetry, or only assemblies? |
| **#1 Symmetry + #6 Palettes** | ✓ YES | Orthogonal concerns - symmetry is geometry, palette is color |
| **#1 Symmetry + #7 L/R Palette** | ✓ YES | Actually synergistic - symmetric geometry, asymmetric color |
| **#1 Symmetry + #8 TS Export** | ✓ YES | Export would include symmetry metadata or baked symmetric geometry |
| **#2 Tri Grid + #3 Tri Pixels** | ✓ YES | Natural pairing if using same scale/alignment |
| **#2 Tri Grid + #4 Lighting** | ✓ YES | Grid is construction aid, lighting is post-process |
| **#2 Tri Grid + #5 Components** | ✓ YES | Components can be grid-aligned |
| **#2 Tri Grid + #6 Palettes** | ✓ YES | Independent systems |
| **#2 Tri Grid + #7 L/R Palette** | ✓ YES | Independent systems |
| **#2 Tri Grid + #8 TS Export** | ? UNCLEAR | Does export need grid metadata? Or just final geometry? |
| **#3 Tri Pixels + #4 Lighting** | ✓ YES | Lighting gradients applied to tri-pixel regions |
| **#3 Tri Pixels + #5 Components** | ✓ YES | Each component rasterizes with tri-pixels |
| **#3 Tri Pixels + #6 Palettes** | ✓ YES | Tri-pixels use palette colors |
| **#3 Tri Pixels + #7 L/R Palette** | ✓ YES | Tri-pixels use L/R palette variations |
| **#3 Tri Pixels + #8 TS Export** | ✓ YES | TS export would include tri-pixel rasterization code |
| **#4 Lighting + #5 Components** | ? COMPLEX | Does lighting apply per-component or to assembled whole? |
| **#4 Lighting + #6 Palettes** | ? COMPLEX | Does lighting affect palette colors, or is it separate layer? |
| **#4 Lighting + #7 L/R Palette** | ✓ YES | Can have different lighting on L/R sides |
| **#4 Lighting + #8 TS Export** | ? UNCLEAR | Does export include lighting code or pre-lit rasters? |
| **#5 Components + #6 Palettes** | ✓ YES | Components use palette system - natural integration |
| **#5 Components + #7 L/R Palette** | ✓ YES | Components can use L/R palette slots |
| **#5 Components + #8 TS Export** | ? COMPLEX | Does export include component assembly logic? |
| **#6 Palettes + #7 L/R Palette** | ✓ YES | #7 is extension of #6 |
| **#6 Palettes + #8 TS Export** | ✓ YES | Export includes palette data and application logic |
| **#7 L/R Palette + #8 TS Export** | ✓ YES | Export includes L/R palette data |

---

## Feature Complexity Assessment

### Likely Simple
- **#1 Symmetry** - Mirror operations are straightforward geometry transforms
- **#2 Tri Grid** - Grid snapping is well-understood feature
- **#6 Palettes** - Two-tier palette is just color indirection

### Moderate Complexity  
- **#3 Tri Pixels** - Custom rasterization, but algorithmically straightforward
- **#7 L/R Palette** - Adds spatial logic to palette system
- **#8 TS Export** - Code generation is moderate complexity

### Potentially Complex
- **#4 Lighting** - Gradient application + multi-angle generation could be involved
- **#5 Components** - Assembly system with positioning/layering adds UI and data structure complexity

### Complexity Multipliers
Combining features may create emergent complexity:
- **Components + Lighting** - Need to decide lighting model for assembled objects
- **Components + TS Export** - Need to decide what gets exported (assembly logic vs baked)
- **All features together** - UI complexity to manage all controls/settings

---

## Next Steps

1. **Prioritize features** - Which are must-have vs nice-to-have?
2. **Define MVP scope** - What is the minimum viable product?
3. **Resolve architecture questions** - Make decisions on "?" items in compatibility matrix
4. **Prototype core features** - Test feasibility of complex items (#4, #5)
5. **Design file format** - Define working file structure
6. **UI mockups** - Sketch interface for selected features

---

## Project Name Ideas

1. **TriSprite** - Emphasizes triangular pixel aesthetic
2. **SymmetryForge** - Highlights symmetry as core feature
3. **VectorSmith** - Generic but professional vector editor name
4. **TriGrid** - Focus on triangular grid system
5. **MirrorSprite** - Emphasizes symmetry and sprite creation
6. **HexForge** - References hexagonal/triangular geometry
7. **ComponentCraft** - Highlights component assembly system
8. **PixelTri** - Simple combination of pixel art and triangles
9. **AssetSmith** - Generic game asset creation tool
10. **TriFacet** - Mathematical term referencing triangular faces
11. **SpriteSymmetry** - Direct description of core features
12. **VectorAssembly** - Emphasizes component system
13. **TriForge** - Short, punchy, triangle-focused
14. **MirrorVec** - Combines symmetry and vector editing
15. **GridSprite** - Generic but clear purpose
16. **FacetForge** - More abstract geometric term
17. **ComponentVector** - Descriptive of assembly system
18. **SymVec** - Abbreviated symmetry + vector
19. **TriMesh** - References triangular tessellation
20. **SpriteForge** - Generic asset creation tool name

## Notes

- Target: 2D top-down games
- Philosophy: Drop features that exceed "simple project" threshold
- All features are candidates, not commitments
- Compatibility issues are opportunities to simplify or clarify, not necessarily blockers
