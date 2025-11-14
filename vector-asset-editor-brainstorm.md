# Trout, a symmetrical triangular vector Game Asset Editor - Brainstorming Document

## Project Overview
A vector-based game asset editor targeting 2D top-down games with a focus on simplicity. Features will be evaluated for implementation complexity and dropped if they exceed the "simple project" threshold.

---

## Architecture Overview

### Core Rendering Architecture
**TypeScript Rendering Library as Foundation**: The entire project is built around a standalone TypeScript rendering library that generates PNG/WebP files from vector data. This library is used by both:
- The editor itself (for real-time preview)
- Game projects (for asset generation at build time or runtime)

This architectural decision ensures long-term asset accessibility and creates a clean separation between the editor UI and the rendering engine.

### Data Model
**Unit Tree Structure**: Assets are composed of recursive "units" where each unit contains:
- Exactly one shape (a set of closed vector paths, possibly empty)
- Zero or more child units
- A palette (with optional empty slots for inheritance)
- Z-ordering determined by list position (later items obscure earlier ones)

**Half-Geometry Storage**: Vector paths are stored un-mirrored (only half the shape). Symmetry is applied at render time, not in the data model. This keeps working files compact and makes asymmetric coloring straightforward.

### Coordinate System
**Triangular Grid Foundation**:
- All control points snap to a triangular/hexagonal lattice
- Power-of-2 scaling system (base size ~2px, then 4px, 8px, 16px, etc.)
- Hexagonal canvas boundary aligned to grid
- Grid spacing measured in real output pixels
- Always-on snapping (no toggle)

### Color System
**Tree-Walking Palette Inheritance**:
- Each unit has a palette with slots that can be filled (hard-coded color) or empty (inherit from parent)
- Color resolution walks up the unit tree to find first non-empty slot
- Global palette at root must have all slots filled (no empties)
- Each palette slot can have separate left/right colors for asymmetric coloring
- Left color lookup: walk up tree for left value; if not found, walk up tree for right value

### Symmetry Model
**Render-Time Bilateral Symmetry**:
- Fixed vertical center line (not movable/rotatable)
- Only one symmetry axis (bilateral only, no radial)
- Geometry is always symmetric (edit one side, both render)
- Asymmetry achieved through color (left/right palette variation)

### Rendering Pipeline
The rendering process flows as:
1. **Vector Construction**: Build shapes on triangular grid (construction-time only)
2. **Component Assembly**: Resolve unit tree, all centers aligned
3. **Symmetry Application**: Mirror geometry across vertical axis
4. **Palette Resolution**: Walk tree to resolve all colors (including left/right)
5. **Lighting Application**: Apply global lighting gradient (if enabled)
6. **Tri-Pixel Rasterization**: Render to triangular pixel grid with anti-aliasing
7. **Rotation Generation**: Create 6-way or 12-way rotated variants (if enabled)
8. **Sprite Sheet Export**: Composite all variants into PNG/WebP sprite sheet

### Editor Architecture
**Separate Edit and Render Areas**:
- Editing UI provides construction tools (grid-snapped vector editing, component assembly, palette management)
- Rendering area shows live preview using the same rendering library that exports use
- Real-time preview required for all features (no render-time-only features)

### Export Strategy
**Sprite Sheet Focus**: Export primarily or exclusively as sprite sheets (multiple assets/variants in single image file). Batch generation of all variants is likely the only export mode.

---

## Feature Ideas

### 1. Core Symmetry System
**Description:** Built-in symmetry as a fundamental feature rather than an optional tool. Symmetry is applied at render time, with only half the geometry stored and edited.

**Key aspects:**
- **Vertical center line symmetry** as default for all new assets (fixed, non-rotatable)
- **Edit one side, render both sides**: Only half of the shape is editable; the mirrored half is visible but locked during editing
- **Storage model**: Coordinate paths stored un-mirrored in working files; symmetry applied during rendering (both on-screen preview and PNG export)
- **Symmetry breaking**: Primarily through color asymmetry (left/right palette variation) rather than geometric breaking
- Single bilateral symmetry only (no radial or multi-axis symmetry)

---

### 2. Triangular Grid Snapping System
**Description:** Construction aid using triangular/hexagonal grid for control point snapping. Grid snapping is always enabled.

**Key aspects:**
- **Control points snap to triangular lattice** - facilitates easy drawing of triangles and hexagons
- **Power-of-2 scaling system**: Base triangle side length (e.g., 2px), with selectable multiples at powers of 2 (2px, 4px, 8px, 16px, etc.)
- **Grid alignment**: Tessellated triangular grids automatically align with each other regardless of scale level
- **Always-on snapping**: Grid snapping cannot be disabled
- **Hexagonal canvas boundary**: The grid defines canvas boundaries as a hexagonal shape
- **Real pixel measurements**: Grid side lengths specified in actual output pixels

---

### 3. Triangular Pixel Rendering
**Description:** Aesthetic style using large visible triangular "pixels" instead of square pixels. Creates a distinctive low-res triangular look baked into standard image formats.

**Key aspects:**
- **Chunky tri-pixel aesthetic**: Each "tri-pixel" rendered as multiple regular pixels forming a triangle
- **Output to standard formats**: PNG/WebP with tri-pixel look baked in (no special file format needed)
- **Power-of-2 sizing**: Tri-pixel size uses the same power-of-2 side length system as the grid
- **Grid alignment**: Tri-pixels align with construction grid, but may differ by one or more powers of 2 in scale
- **Anti-aliased edges**: Tri-pixels have smooth edges rather than hard pixel boundaries

---

### 4. Faux 3D Lighting
**Description:** Simulate dimensionality through lighting gradients across shapes. Automatically generates rotated angle variants with adjusted lighting for different viewing directions.

**Key aspects:**
- **Global lighting**: Light direction applies across entire scene (not per-shape)
- **Z-axis height**: Single height value specifying elevation (applied to center/middle of asset)
- **Light-to-dark gradients**: Applied across shapes to create pseudo-3D effect
- **Automatic rotation generation**: 6-way minimum (every 60°), with 12-way (every 30°) option for smoother animation
- **Rotation independent of lighting**: Shape rotation should work even without faux 3D lighting enabled
- **Configurable light direction**: Light direction can be set per export

---

### 5. Component/Layer Assembly System
**Description:** Build reusable components ("units") and assemble them into complete characters/objects using a recursive tree structure.

**Key aspects:**
- **Unit structure**: Each unit contains exactly one shape (which may be empty) + zero or more child units
- **Recursive/nested assembly**: Units can reference other units to arbitrary depth
- **Runtime assembly**: Components assembled in editor view and during export (not baked into single geometry)
- **Fixed positioning**: All components overlaid unscaled with centers exactly aligned (no per-component positioning/scaling)
- **Z-ordering by list order**: Child units and shapes can be arbitrarily ordered; order alone determines visibility (later items obscure earlier ones)
- **Library-based workflow**: Create reusable components (body parts, clothing, accessories) and mix/match to create variants

---

### 6. Dynamic Palette Inheritance
**Description:** Color palette system with inheritance up the component tree. Enables palette-driven variants without geometry duplication.

**Key aspects:**
- **Empty slot inheritance**: Palettes can have empty slots; when a color is needed for an empty slot, walk up the parent tree to find the first non-empty value
- **Global palette root**: A global palette exists at the root of the inheritance tree with no empty slots allowed (ensures all colors resolve eventually)
- **Mixed coloring per component**: Each component can have some colors hard-coded (filled slots) and others inherited (empty slots)
- **Tree-walk resolution**: Color lookup walks up parent chain until a filled slot is found
- **Visualized in editor**: Palette inheritance is shown visually in the editor
- **Example use**: Trousers with fixed blue fabric (filled slot) but handkerchief color from parent's hair color (empty slot)

---

### 7. Left/Right Palette Variation
**Description:** Palette slots can have different values for left vs right side of the symmetry axis, enabling asymmetric coloring on symmetric geometry.

**Key aspects:**
- **Automatic region definition**: Left/right regions determined automatically by symmetry axis (no manual painting)
- **Right-side default**: Each palette slot's "right color" is the default value
- **Optional left override**: Palette slots can optionally specify a different "left color"
- **Tree-walk for left colors**: When looking up left color, walk up parent tree to find first non-empty left value; if none found, walk up tree again for right value
- **Bilateral only**: Only works with single-axis bilateral symmetry (no radial/multi-segment support)
- **Example use**: Red left glove, blue right glove on symmetric character; symmetric geometry, asymmetric coloring

---

### 8. TypeScript Rendering Library
**Description:** Core rendering system implemented as a standalone TypeScript library that both the editor and games can use. Generates rasterized images (PNG/WebP) from vector data.

**Key aspects:**
- **Architectural shift**: Not just an "export feature" - this is THE rendering library that powers the entire tool
- **Dual usage**: Editor uses this library for real-time preview; games use it for asset generation
- **Full feature support**: Includes all editor features (symmetry operations, component assembly, palette inheritance, tri-pixels, lighting)
- **Image generation**: Produces PNG/WebP files (not screen rendering - actual file output)
- **Human-readable format**: Vector data and rendering code are readable and hand-editable TypeScript
- **Long-term accessibility**: Assets remain usable even if editor disappears
- **Standard dependency management**: Uses normal TypeScript/npm dependency handling

---

### 9. Client-Side Web Application
**Description:** Web-based HTML application that runs entirely in the browser with minimal or no server-side dependencies.

**Key aspects:**
- **Browser-based**: Entire editor runs in the browser (HTML/CSS/JavaScript/TypeScript)
- **No server required**: All editing, rendering, and processing happens client-side
- **Local file operations**: Uses browser APIs for file I/O (File System Access API, downloads, or local storage)
- **Standalone deployment**: Can be hosted as static files on any web server or CDN
- **GitHub Pages compatible**: Ideal for hosting on GitHub Pages (static site hosting)
- **Offline capable**: Potentially works offline once loaded (Progressive Web App)
- **Cross-platform**: Works on any device with a modern browser

---

## Design Decisions

### Scope & Philosophy
- **"Simple" means low line count**: Primary simplicity metric is keeping codebase small
- **Workflow focus**: Create many simple assets quickly (not complex assets with fine control)
- **Target platform**: TypeScript/HTML5 games (primary: Excalibur.js; general: any TypeScript game engine)

### Symmetry Architecture
- **Fixed vertical symmetry axis**: Not movable or rotatable
- **Single symmetry type**: Only bilateral symmetry (no radial or multi-axis)
- **Color-based asymmetry**: Symmetry breaking primarily through left/right palette variation, not geometry

### Grid System
- **Triangular grid only**: No rectangular grid option
- **Hexagonal canvas boundary**: Grid defines canvas shape
- **Grid-aligned editing**: Construction grid defines all geometric constraints

### Export & File Format
- **Working file format**: JSON (TBD exact schema)
- **Raster formats**: PNG and WebP
- **No import support**: Editor does not import existing SVG/PNG assets
- **Sprite sheet focus**: Export primarily or exclusively as sprite sheets (multiple assets in one image)

### Rendering & Performance
- **Real-time preview required**: All features must render in real-time during editing
- **Separate edit/render areas**: Editing UI separate from rendering preview area
- **Rendering library-based**: Editor uses the same rendering library that exports use
- **Expected complexity**: Dozens of units, each with dozens of vertices
- **Batch export**: Likely the only export mode (generate all variants at once)

---

## Feature Compatibility Matrix

| Feature Pair | Compatible? | Notes |
|--------------|-------------|-------|
| **#1 Symmetry + #2 Tri Grid** | ✓ YES | Symmetry axis aligns to grid, triangles mirror cleanly |
| **#1 Symmetry + #3 Tri Pixels** | ✓ YES | As long as symmetry axis aligns to pixel grid |
| **#1 Symmetry + #4 Lighting** | ✓ YES | Symmetric shapes with symmetric lighting work naturally |
| **#1 Symmetry + #5 Components** | ✓ YES | Symmetry applied at render time to final assembled result; components don't have individual symmetry |
| **#1 Symmetry + #6 Palettes** | ✓ YES | Orthogonal concerns - symmetry is geometry, palette is color |
| **#1 Symmetry + #7 L/R Palette** | ✓ YES | Actually synergistic - symmetric geometry, asymmetric color |
| **#1 Symmetry + #8 TS Export** | ✓ YES | Export would include symmetry metadata or baked symmetric geometry |
| **#2 Tri Grid + #3 Tri Pixels** | ✓ YES | Natural pairing if using same scale/alignment |
| **#2 Tri Grid + #4 Lighting** | ✓ YES | Grid is construction aid, lighting is post-process |
| **#2 Tri Grid + #5 Components** | ✓ YES | Components can be grid-aligned |
| **#2 Tri Grid + #6 Palettes** | ✓ YES | Independent systems |
| **#2 Tri Grid + #7 L/R Palette** | ✓ YES | Independent systems |
| **#2 Tri Grid + #8 TS Export** | ✓ YES | Grid is construction aid only; rendering library works with final vector paths |
| **#3 Tri Pixels + #4 Lighting** | ✓ YES | Lighting gradients applied to tri-pixel regions |
| **#3 Tri Pixels + #5 Components** | ✓ YES | Each component rasterizes with tri-pixels |
| **#3 Tri Pixels + #6 Palettes** | ✓ YES | Tri-pixels use palette colors |
| **#3 Tri Pixels + #7 L/R Palette** | ✓ YES | Tri-pixels use L/R palette variations |
| **#3 Tri Pixels + #8 TS Export** | ✓ YES | TS export would include tri-pixel rasterization code |
| **#4 Lighting + #5 Components** | ✓ YES | Global lighting applies to assembled whole; single z-height for entire asset |
| **#4 Lighting + #6 Palettes** | ✓ YES | Lighting applied as separate post-process layer on top of palette-resolved colors |
| **#4 Lighting + #7 L/R Palette** | ✓ YES | Can have different lighting on L/R sides |
| **#4 Lighting + #8 TS Export** | ✓ YES | Rendering library includes full lighting implementation; can generate with different light directions |
| **#5 Components + #6 Palettes** | ✓ YES | Components use palette system - natural integration |
| **#5 Components + #7 L/R Palette** | ✓ YES | Components can use L/R palette slots |
| **#5 Components + #8 TS Export** | ✓ YES | Rendering library includes full component assembly logic; runtime assembly not baked |
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

## Feature Priorities

### Critical (Must-Have)
1. **Core Symmetry System** (#1)
2. **Component/Layer Assembly System** (#5)
3. **Basic Palettes** (simplified version of #6 - no inheritance)

### Medium Priority
4. **Triangular Grid Snapping System** (#2)
5. **Triangular Pixel Rendering** (#3)
6. **Full Palette Inheritance** (advanced version of #6)
7. **Left/Right Palette Variation** (#7)

### Nice-to-Have
8. **Faux 3D Lighting** (#4)

### Infrastructure (Will Get Done)
9. **TypeScript Rendering Library** (#8)
10. **Client-Side Web Application** (#9)

---

## Feature Breakdown & Dependencies

### Feature 1: Core Symmetry System
**Sub-features:**
- 1a. Store half-geometry in data model
- 1b. Mirror geometry at render time
- 1c. Display mirrored (read-only) side in editor
- 1d. Constrain editing to editable half

**Dependencies:**
- Requires: Vector path data structure
- Enables: L/R Palette Variation (#7)

**Priority:** CRITICAL

---

### Feature 2: Triangular Grid Snapping System
**Sub-features:**
- 2a. Grid geometry/math (triangular tessellation)
- 2b. Grid display/visualization in editor
- 2c. Point-to-grid snapping logic
- 2d. Power-of-2 scale selection UI
- 2e. Hexagonal canvas boundary enforcement

**Dependencies:**
- Requires: Basic vector editing capability
- Independent of: Rendering pipeline

**Priority:** MEDIUM

---

### Feature 3: Triangular Pixel Rendering
**Sub-features:**
- 3a. Triangular rasterization algorithm
- 3b. Anti-aliasing for triangle edges
- 3c. Tri-pixel scale configuration

**Dependencies:**
- Requires: Vector paths to rasterize, basic rendering pipeline
- Works with: Any vector source (symmetry, components, etc.)

**Priority:** MEDIUM

---

### Feature 4: Faux 3D Lighting
**Sub-features:**
- 4a. Lighting gradient calculation (light direction → color gradient)
- 4b. Z-height property and application
- 4c. Rotation generation (6-way or 12-way)
- 4d. Light direction configuration UI
- 4e. Lighting application to assembled geometry

**Dependencies:**
- Requires: Basic rendering pipeline, assembled geometry
- Independent of: Tri-pixel rendering (can apply to any rasterization)

**Priority:** NICE-TO-HAVE

---

### Feature 5: Component/Layer Assembly System
**Sub-features:**
- 5a. Unit data structure (shape + children + palette)
- 5b. Recursive tree traversal
- 5c. Z-order rendering (list order → draw order)
- 5d. Component library management (save/load/reference units)
- 5e. Assembly UI (viewing/editing unit tree)

**Dependencies:**
- Requires: Basic shape rendering
- Enables: Palette inheritance (#6, #7)

**Priority:** CRITICAL

---

### Feature 6: Dynamic Palette Inheritance
**Sub-features:**
- 6a. Palette data structure (array of color slots, some empty)
- 6b. Tree-walking color resolution algorithm
- 6c. Global palette (root with no empties)
- 6d. Palette editor UI
- 6e. Inheritance visualization in editor

**Dependencies:**
- Requires: Component tree (#5)
- Enables: L/R Palette Variation (#7)

**Priority:** CRITICAL (basic palettes only) / NICE-TO-HAVE (full inheritance)

---

### Feature 7: Left/Right Palette Variation
**Sub-features:**
- 7a. L/R palette slot data structure (each slot has optional left + right colors)
- 7b. Determine left/right regions from symmetry axis
- 7c. Tree-walking for L/R color resolution (left first, fallback to right)
- 7d. L/R palette editor UI

**Dependencies:**
- Requires: Symmetry (#1), Palette inheritance (#6)
- Enhances: Asymmetric coloring on symmetric geometry

**Priority:** NICE-TO-HAVE

---

### Feature 8: TypeScript Rendering Library
**Sub-features:**
- 8a. Core vector rendering (paths → pixels)
- 8b. PNG/WebP file generation
- 8c. Library API design
- 8d. Integration with editor (real-time preview)
- 8e. Sprite sheet layout and export

**Dependencies:**
- Foundation for: Everything else
- Must include: All other features' rendering logic

**Priority:** INFRASTRUCTURE (foundational - will get done)

---

## Dependency Graph

```
Foundation Layer:
  [8. TS Rendering Library] ← EVERYTHING depends on this
       ↓

Basic Rendering:
  [Vector paths + basic rasterization] ← needed for anything visual
       ↓
       ├─→ [1. Symmetry] ← can be added early, relatively simple
       ├─→ [2. Tri Grid] ← editor-only, doesn't affect rendering
       └─→ [3. Tri Pixels] ← rendering technique, independent

Component System:
  [5. Component Assembly]
       ↓
  [6. Palette Inheritance]
       ↓
  [7. L/R Palette Variation] ← requires both symmetry (#1) and palettes (#6)

Advanced Rendering:
  [4. Faux 3D Lighting] ← can be added late, works on assembled result
       ├─→ Lighting gradients
       └─→ Rotation generation
```

---

## Development Phases

### Phase 0: Ultra-Minimal MVP (Proof of Concept)
**Goal**: Prove the core architecture works end-to-end

**Editor:**
- Create/edit a few vector points (simplest possible UI, or even hard-coded points)
- No grid yet, just raw point manipulation

**Renderer:**
- Take points, mirror them across vertical axis
- Basic rasterization (filled shapes, nothing fancy)
- Output single PNG file

**Validation**: Confirms that data → renderer → image pipeline works, and that editor can invoke renderer for preview.

---

### Phase 1: Make It Actually Usable
**Goal**: Editor becomes functional for real work

**Editor additions:**
- Triangular grid (#2) with snapping
- Basic UI for add/remove/move points on grid
- Save/load JSON files
- Live preview of renderer output

**Renderer improvements:**
- Better rasterization quality
- Possibly add triangular pixels (#3) for distinctive aesthetic

**Deliverable**: Can create and save simple symmetric sprites

---

### Phase 2: Component System
**Goal**: Enable reusable components and composition

**Features:**
- Component tree structure (#5)
- Basic palettes (#6) - simple colors first, no inheritance complexity yet
- Component library (save/load/reference units)
- UI for managing component hierarchy

**Deliverable**: Can build characters from reusable parts (body, clothes, etc.)

---

### Phase 3: Polish & Advanced Features
**Goal**: Add sophisticated features that enhance workflow

**Features:**
- Palette inheritance and L/R variation (#7)
- Faux 3D lighting (#4)
- Rotation generation
- Sprite sheet export

**Deliverable**: Full-featured tool with all planned capabilities

---

## Next Steps

1. **Design Phase 0 data structures** - Define minimal JSON schema for vector paths
2. **Prototype renderer** - Build standalone rendering function
3. **Prototype editor** - Simplest possible UI for point editing
4. **Integrate** - Connect editor → renderer → preview

## Notes

- Target: 2D top-down games
- Philosophy: Drop features that exceed "simple project" threshold
- All features are candidates, not commitments
- Compatibility issues are opportunities to simplify or clarify, not necessarily blockers
