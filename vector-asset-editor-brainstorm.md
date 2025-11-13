# Trout, a symmetrical triangular vector Game Asset Editor - Brainstorming Document

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
ANSWER: I think that the color asymmetry discussed elsewhere may be good enough
- Is symmetry enforcement live/continuous or operation-based?
ANSWER: I don't understand the question. But I'll take a stab at it. This tool will have two main products. The first is editable working files, which should contain atleast: A sets of coordinate paths describing vector paths. And some color information for the shapes. Composite unit descriptions, etc. The second is the rendered PNGs. The coordinate paths would not be stored mirrored. Only during onscreen rendering and in the rendered PNGS would the symmetry be fully relaized, and during editing only half of the shape would be editable, while the other would be visible but uneditable.
- What is the default symmetry axis for new assets?
ANSWER: vertical center line

---

### 2. Triangular Grid Snapping System
**Description:** Construction aid using triangular/hexagonal grid for control point snapping.

**Key aspects:**
- Control points and vertices snap to triangular lattice
- Facilitates easy drawing of triangles and hexagons
- Grid-aligned geometry construction

**Open questions:**
- What is the grid scale/size?
ANSWER: triangular grids ( tessellated triangles ) automatically align with triangular grids with side lengths that are powers of two multiples of the first grid. We'll decide on the side length of the very smallest triangle side length ( maybe 2 ) and then the choices will be powers of two multiples of that. In any case, the power of two side length will be selectable.
- Is grid snapping always on, or toggleable?
ANSWER: always on, I think.
- How does grid relate to final output resolution?
ANSWER: I think that we'll discuss side lengths in real pixels.

---

### 3. Triangular Pixel Rendering
**Description:** Aesthetic style using large visible triangular "pixels" instead of square pixels.

**Key aspects:**
- Rasterize with chunky triangular pixel look
- Output to standard image formats (PNG, etc.) but with tri-pixel aesthetic
- Each "tri-pixel" rendered as multiple regular pixels forming triangle

**Open questions:**
- What is the tri-pixel size relative to output resolution?
ANSWER: it will be one of the powers of two side lengths discussed above.
- Do tri-pixels align with construction grid (#2) if both features are used?
ANSWER: yes, by they may differ one or more powers of two
- Are tri-pixels anti-aliased or hard-edged?
ANSWER anti-aliased.

---

### 4. Faux 3D Lighting
**Description:** Simulate dimensionality through lighting gradients across shapes.

**Key aspects:**
- Apply light-to-dark gradient on shapes
- Auto-generate rotated angle variants with adjusted lighting
- Creates pseudo-3D effect for 2D sprites

**Open questions:**
- Is lighting per-shape or global across scene?
ANSWER. Global I guess. I guess I was thinking of having a single z-access height that specified the height of just the middle.
- How many rotation angles are generated? (4-way? 8-way? 16-way?)
ANSWER: your question written wrong, the example answers should say (3-way, 6-way, 12-way, etc)?, but I would assume 6 way as a minimum, but 12 way might be nice for an animated effect.
- Is angle generation automatic export or manual selection?
ANSWER: I don't understand the question. We should likely do automatic rotation of shapes regardless of faux 3d lighting. And it would be automatic.
- Is light direction configurable per export?
ANSWER: sure.

---

### 5. Component/Layer Assembly System
**Description:** Build reusable components and assemble them into complete characters/objects.

**Key aspects:**
- Create library of reusable components (body parts, clothing, accessories)
- Mix and match components to create variants
- Assemble characters from base + additions (naked character + clothing)

**Open questions:**
- Is assembly runtime (editor view, export components) or bake-in (export composite)?
ANSWER: runtime.
- Can components be positioned/scaled during assembly or use fixed attachment points?
ANSWER: all components will overlaid unscaled with their positions and centers all exactly aligned.
- How are component layers ordered/managed?
ANSWER: I don't understand the question. I am thinking that every "unit" will have exactly one, possibly empty, shape, which is a set of closed paths. And that the unit will have 0 or more child units. The child units and the shape can be arbitrarily ordered, and order alone will dictate visibility with upper unts or shapes obscuring lower units or shapes.
- Can components reference other components (nested assembly)?
ANSWER: yes. Recursive.

---

### 6. Dynamic Palette Inheritance
**Description:** Components with mixed coloring - some hard-coded, some inherited from parent.

**Key aspects:**
- Two-tier palette system: shared palette + local/unshared colors
XXX maybe we can just have the palette have empty slots. And that when selecting empty slot colors the parents pallet slot gets used.
- Child components inherit parent's shared palette
- Enables palette-driven variants without geometry duplication
- Example: trousers with fixed blue fabric but handkerchief color from parent's hair

**Open questions:**
- How many shared palette slots?
- What happens when child is used standalone (no parent to inherit from)?
ANSWER: maybe we make a global palette that all of the other palettes inherit from. Then disallow empty slots in the global palette.
- Is palette inheritance visualized in editor?
ANSWER: because palettes are seen in the editor, then yes, some amount of visualization should be seen.
- Can shared palette be overridden at assembly time?
ANSWER: why? I'm not sure.

---

### 7. Left/Right Palette Variation
**Description:** Shared palette slots can have different values for left vs right side.

**Key aspects:**
- Enables asymmetric coloring on symmetric geometry
- Child components use left/right palette slots
- Example: red left glove, blue right glove on symmetric character

**Open questions:**
- How are left/right regions defined? (automatic from symmetry axis? manual painting?)
ANSWER: automatic from symmetry.
- Does this apply only to shared palette or also local colors?
ANSWER: good question. Maybe you do it like the empty color slot described above. The default color is the right color. And if there is no left color then the right color gets used. The only caveat is that you need to walk up the inheritance tree looking for a left color before defaulting to the right color, which itself might require walking up the inheritance tree.
- Can this support more than bilateral symmetry (e.g., radial with 4+ segments)?
ANSWER: I'm only planning on one axis of symmetry for now.

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
ANSWER: the latter. In fact this whole feature should basically be rethought of just making a rendering library that gets used in the tool, but can also be used independently. 
- Does rendering code implement style features (tri-pixels, lighting) or just basic vector rendering?
ANSWER: see above
- What rendering library/API does the TS code target (Canvas2D, WebGL, generic)?
ANSWER: uhh, I don't understand this question. The rendering library is for producing like pngs or such. not for drawing to the screen.
- How are external dependencies handled?
ANSWER: how do typescript dependencies get handled normally?

---

## Open Design Questions

### Scope & Philosophy
1. What defines "super simple" for this project? Line count? Time to implement? Feature count?
ANSWER: mostly line count.
2. What is the primary workflow: create many simple assets quickly, or create complex assets with fine control?
ANSWER: more the former. Simple assets quickly.
3. What game engines/frameworks is this targeting? (affects export format priorities)
ANSWER: any typescript html game. I'm going to be working with Excalibur.js, but it should work outside of that as well.

### Symmetry Architecture
4. Should symmetry axis be movable/rotatable or fixed?
ANSWER: fixed.
5. Can multiple symmetry types exist on one asset (e.g., bilateral + radial)?
ANSWER: no
6. How is symmetry preserved/broken when editing components vs assembled objects?
ANSWER: maybe just by color

### Grid System
7. If using triangular grid (#2), does it also define canvas boundaries/alignment?
ANSWER: yes. Lets have a hexagonal boundary.
8. Should there be an option for rectangular grid, or is triangular grid the only option?
ANSWER: no

### Export & File Format
9. What is the native working file format?
ANSWER: TDB, but probably just json, right.
10. What raster export formats are needed? (PNG, WebP, etc.)
ANSWER: probably those two for now.
11. Should editor support import of existing assets (SVG, PNG)?
ANSWER: no.
12. Is there a need for sprite sheet export (multiple assets in one image)?
ANSWER: yes. maybe ONLY this.

### Rendering & Performance
13. Is real-time preview of all features required, or can some be render-time only?
ANSWER: real-time. We should be utilizing the actual rendering library to render in real time during editing. Though the editing area should be a separate from the rendering area.
14. What is the expected asset complexity (vertex count, shape count)?
ANSWER: dozens of units with dozens of vertexes.
15. Should there be a batch export mode for generating all variants?
ANSWER: maybe only this.

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

phasizes triangular pixel aesthetic
- Highlights symmetry as core feature
Generic but professional vector editor name
s on triangular grid system
 Emphasizes symmetry and sprite creation
erences hexagonal/triangular geometry
 - Highlights component assembly system
ple combination of pixel art and triangles
eneric game asset creation tool
thematical term referencing triangular faces
* - Direct description of core features
* - Emphasizes component system
ort, punchy, triangle-focused
ombines symmetry and vector editing
Generic but clear purpose
More abstract geometric term
** - Descriptive of assembly system
eviated symmetry + vector
erences triangular tessellation
 Generic asset creation tool name

## Notes

- Target: 2D top-down games
- Philosophy: Drop features that exceed "simple project" threshold
- All features are candidates, not commitments
- Compatibility issues are opportunities to simplify or clarify, not necessarily blockers
