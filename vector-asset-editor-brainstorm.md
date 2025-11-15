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

**⚠️ SPECIFICATION GAP**: Vector path data structure not defined
- What is a "path"? Array of points? Data structure?
- Are paths always closed loops or can they be open?
- Bezier curves supported or only straight line segments?
- How are points stored? Grid coordinates or pixels?
- Need: Explicit TypeScript type definition for `Path`, `Point`, `Shape`

**⚠️ SPECIFICATION GAP**: Z-ordering model ambiguous
- Are shapes and child units in separate ordered lists or interleaved?
- If unit has 2 shapes + 2 children, what's the render order?
- Need: Clarify if `Unit` has `shapes: Shape[], children: Unit[]` or `items: (Shape|Unit)[]`

**Half-Geometry Storage**: Vector paths are stored un-mirrored (only half the shape). Symmetry is applied at render time, not in the data model. This keeps working files compact and makes asymmetric coloring straightforward.

**⚠️ SPECIFICATION GAP**: Half-geometry storage edge cases
- Which half is stored? Left (x < 0), right (x > 0), or right including center (x ≥ 0)?
- How are points exactly ON the symmetry axis (x = 0) handled? Stored once? Duplicated?
- If path crosses center line multiple times, how is it segmented?
- Need: Explicit rule for center-line point storage

### Coordinate System
**Triangular Grid Foundation**:
- All control points snap to a triangular/hexagonal lattice
- Power-of-2 scaling system (base size ~2px, then 4px, 8px, 16px, etc.)
- Hexagonal canvas boundary aligned to grid
- Grid spacing measured in real output pixels
- Always-on snapping (no toggle)

**⚠️ CRITICAL SPECIFICATION GAP**: Grid coordinate system undefined
- What coordinate system? Axial? Cube? Offset? Cartesian?
- How are triangular grid points addressed numerically?
- What's the formula to convert grid coordinates to pixel coordinates?
- Pointy-top or flat-top hexagons? (Affects orientation)
- What is grid origin (0,0) - center, corner, or user-defined?
- Need: Explicit coordinate system with conversion formulas
- Example needed: "Point at grid (q=1, r=0) → pixel (x=?, y=?)"

### Color System
**Tree-Walking Palette Inheritance**:
- Each unit has a palette with slots that can be filled (hard-coded color) or empty (inherit from parent)
- Color resolution walks up the unit tree to find first non-empty slot
- Global palette at root must have all slots filled (no empties)
- Each palette slot can have separate left/right colors for asymmetric coloring
- Left color lookup: walk up tree for left value; if not found, walk up tree for right value

**⚠️ SPECIFICATION GAP**: Palette slot count not specified
- How many slots per palette? 2? 4? 8? 16?
- Recommendation: Start with 8 slots (good balance of flexibility vs. UI complexity)
- Need: Explicit decision on palette array size

**Stroke and Fill Support**:
- Each path can have both stroke (outline) and fill (interior) colors
- Stroke and fill reference palette slots independently
- Stroke width configurable per path
- Both stroke and fill respect the palette inheritance system
- Either stroke or fill can be omitted (stroke-only or fill-only paths)

**⚠️ SPECIFICATION GAP**: Stroke width units undefined
- Is stroke width in pixels, grid units, or percentage?
- What's the default stroke width?
- Minimum/maximum stroke width?
- Recommendation: "Stroke width in output pixels; default 1px; range 0.5-10px"

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

**IMPLEMENTATION QUESTIONS:**
- **Data model:** How is symmetry stored? Is each mirrored point stored twice, or do we store only one half and compute the mirror on-the-fly?
- **Breaking symmetry:** When symmetry is broken for a point/shape, does the mirrored copy persist as independent geometry, or does it disappear?
- **Undo/redo:** If I break symmetry, edit the independent halves, can I re-enable symmetry? What happens to the asymmetric edits?
- **Selection model:** Can users select mirrored pairs together, or only individual points? How is this indicated visually?
- **Symmetry axis manipulation:** Can users move/rotate the symmetry axis after creation? Does moving the axis re-mirror existing geometry?
- **Multiple symmetry types:** Can a single asset have both horizontal AND vertical symmetry (quadrant symmetry)? How is this toggled?
- **Copy/paste behavior:** If I copy a symmetric shape and paste it, is the paste also symmetric?

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

**IMPLEMENTATION QUESTIONS:**
- **Grid topology:** Is it a regular triangular tessellation, or a hexagonal grid with triangle subdivision? (These are different grid structures)
- **Grid origin:** Where is (0,0) on the grid? Center of canvas, corner, or user-defined?
- **Grid orientation:** Do triangles point up by default, or is there a rotation parameter?
- **Coordinate system:** What coordinate space is the grid in? Canvas pixels, world units, or abstract grid units?
- **Snapping tolerance:** How close must the cursor be to a grid point to snap? Is this configurable?
- **Sub-grid precision:** Can users place points between grid points (with modifier key), or is the grid absolute?
- **Grid visibility:** Is the grid always visible, or only when snapping is enabled? Can opacity be adjusted?
- **Zoom behavior:** Does the grid scale with zoom, or is it defined in canvas space and thus changes visual size with zoom?
- **Grid scaling UI:** How does the user change grid size? Numeric input, preset sizes, or slider?
- **Edge cases:** What happens at canvas boundaries - does the grid tile infinitely or stop at edges?

---

### 3. Triangular Pixel Rendering
**Description:** Aesthetic style using large visible triangular "pixels" instead of square pixels. Creates a distinctive low-res triangular look baked into standard image formats.

**Key aspects:**
- **Chunky tri-pixel aesthetic**: Each "tri-pixel" rendered as multiple regular pixels forming a triangle
- **Output to standard formats**: PNG/WebP with tri-pixel look baked in (no special file format needed)
- **Power-of-2 sizing**: Tri-pixel size uses the same power-of-2 side length system as the grid
- **Grid alignment**: Tri-pixels align with construction grid, but may differ by one or more powers of 2 in scale
- **Anti-aliased edges**: Tri-pixels have smooth edges rather than hard pixel boundaries

**IMPLEMENTATION QUESTIONS:**
- **Rasterization algorithm:** How do we determine which tri-pixel a vector path fills? Scanline conversion? Coverage sampling?
- **Tri-pixel tessellation:** Do upward and downward triangles alternate in a regular pattern, or is there a different approach?
- **Color sampling:** How is color determined per tri-pixel? Majority coverage, center point sampling, or average of covered area?
- **Gradients/shading:** If a vector shape has a gradient, how does it map to discrete tri-pixels? Per-pixel sampling or dithering?
- **Overlapping shapes:** How are layers/overlaps handled? Standard painter's algorithm, or something specific to triangular pixels?
- **Transparency:** How is alpha blending handled with tri-pixels? Per tri-pixel alpha, or anti-aliased edges between tri-pixels?
- **Size specification:** Is tri-pixel size defined in absolute pixels (e.g., "10px wide triangles") or relative to canvas (e.g., "64 tri-pixels across")?
- **Output resolution relationship:** If I want a 256x256 PNG output with tri-pixels, how many tri-pixels fit? (This affects aspect ratio since triangles aren't square)
- **Edge rendering:** At shape boundaries, do partial tri-pixels get rendered, or only fully covered ones?
- **Performance:** Is this a real-time preview feature, or export-only? Large tri-pixel counts could be expensive to rasterize.

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

**IMPLEMENTATION QUESTIONS:**
- **Lighting model:** Is this a simple directional gradient, or does it consider shape normals/depth? (Simple = light direction projects gradient; complex = simulate actual 3D lighting)
- **Gradient application:** How is the gradient applied to arbitrary vector shapes? Along longest axis? Perpendicular to light direction?
- **Multi-shape scenes:** If an asset has multiple shapes, does each shape get independently lit, or do they shadow each other?
- **Light source position:** Is the light infinitely far (parallel rays) or at a specific position (radial lighting)?
- **Rotation mechanics:** When generating angle variants, do you rotate the geometry or the light source? (These produce different results)
- **Angle specification:** Are rotation angles evenly spaced (0°, 45°, 90°...) or user-defined? How is this configured?
- **Preview vs export:** Can the user preview all angles in the editor, or is this an export-time batch process?
- **Color interaction:** How does lighting interact with base colors? Multiply blend? Overlay? Additive/subtractive?
- **Intensity control:** Can users control lighting intensity (subtle to dramatic)? Is there ambient light + directional light?
- **Flat shading vs smooth:** Is each shape uniformly lit, or can gradients be smooth across shape area?
- **Shape depth ordering:** For multi-shape assets, is there a Z-order that affects lighting (front shapes lit differently than back)?
- **Export format:** Are angle variants exported as separate files, sprite sheet frames, or something else?

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

**⚠️ SPECIFICATION GAP**: "Center alignment" model undefined
- How is unit center calculated? Bounding box center? Explicit center property?
- If unit has empty shape, where is its center?
- How is center calculated when unit has child units at different positions?
- Example: Unit A spans x=(-10,10), Unit B spans x=(0,20). When B is child of A, where does B render?
- Need: Explicit center calculation formula or property definition

**IMPLEMENTATION QUESTIONS:**
- **Component storage:** Are components separate files, or stored within a project file? How is the component library organized?
- **Component boundaries:** What defines a component? A single shape, a group of shapes, or can it include metadata (attachment points, palette slots)?
- **Assembly data structure:** How is an assembly stored? List of component references + transforms? Flat merged geometry?
- **Transform constraints:** If components can be positioned, what transformations are allowed? Translation only? Rotation? Scale? Skew?
- **Attachment points:** If using fixed attachment, how are they defined? Named points? Socket system? Automatic bounds-based?
- **Layer ordering UI:** How does the user reorder layers? Drag-and-drop list? Z-index numbers? Bring-to-front/send-to-back commands?
- **Component updates:** If I edit a component used in 10 assemblies, do all assemblies update automatically? Or are they baked/frozen?
- **Nesting depth:** If allowing nested components, what's the maximum depth? How is circular reference prevented?
- **Component visibility:** Can components be hidden in an assembly without removing them? Toggled on/off?
- **Local overrides:** Can individual instances override component properties (color, transform) without affecting the source component?
- **Export options:** When exporting an assembly, can users choose to export: (a) flattened raster, (b) separate component layers, (c) vector with component metadata?
- **Component variants:** Can a component have variants (e.g., "shirt" component with "short sleeve" and "long sleeve" variants)? How is this managed?
- **Clipping/masking:** Can components mask or be masked by other components?

---

### 6. Dynamic Palette Inheritance
**Description:** Color palette system with inheritance up the component tree. Enables palette-driven variants without geometry duplication.

**Key aspects:**
- **Empty slot inheritance**: Palettes can have empty slots; when a color is needed for an empty slot, walk up the parent tree to find the first non-empty value
- **Global palette root**: A global palette exists at the root of the inheritance tree with no empty slots allowed (ensures all colors resolve eventually)
- **Mixed coloring per component**: Each component can have some colors hard-coded (filled slots) and others inherited (empty slots)
- **Tree-walk resolution**: Color lookup walks up parent chain until a filled slot is found
- **Stroke and fill support**: Paths reference palette slots separately for stroke (outline) and fill (interior) colors
- **Visualized in editor**: Palette inheritance is shown visually in the editor
- **Example use**: Trousers with fixed blue fabric (filled slot) but handkerchief color from parent's hair color (empty slot)

**IMPLEMENTATION QUESTIONS:**
- **Palette slot count:** Exactly how many shared slots? (2? 4? 8? 16?) More slots = more flexibility but more UI complexity
- **Slot semantics:** Are slots named/semantic ("primary", "secondary", "accent") or just numbered (1, 2, 3...)?
- **Color assignment UI:** How does a user assign a shape region to a palette slot vs a hard-coded color? Color picker with "use slot X" option? Separate tools?
- **Default/fallback colors:** When a child component uses palette slot 2 but has no parent, what color is used? Per-component defaults? First color in child's palette? Error state?
- **Visual indicators:** How does the editor show which regions use which palette slot? Color-coding? Overlay mode? Inspector panel?
- **Inheritance chain:** If component A uses component B which uses component C, does C inherit from A's palette or B's palette? Or both?
- **Override mechanism:** At assembly time, if the user wants to override a specific palette slot for one component instance, how is that specified? Per-instance palette object?
- **Palette editing workflow:** When editing a component in isolation, what colors does the shared palette show? Placeholder colors? Last-used assembly's colors? Configurable preview palette?
- **Export behavior:** When exporting an assembly, are palette values baked in, or is palette metadata preserved (for runtime swapping)?
- **Palette animation:** Could palette values change at runtime (e.g., character turning red when damaged)? Is this in scope?
- **Mixed slot types:** Can a single shape use both shared palette AND local colors? (e.g., gradient from slot 1 to hard-coded blue)
- **Palette UI placement:** Where is palette configuration in the UI? Per-component properties? Global assembly panel? Both?

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

**IMPLEMENTATION QUESTIONS:**
- **Region determination:** How does the system know which parts are "left" vs "right"? Computed from symmetry axis? Per-point metadata? Per-shape metadata?
- **Palette slot expansion:** Does this double the palette (slot 1 becomes slot 1-left + slot 1-right)? Or is it a modifier on existing slots?
- **Data structure:** How is L/R color stored? `{ slot: 1, leftColor: red, rightColor: blue }`? Separate L/R palette objects?
- **Center region handling:** What happens to geometry exactly ON the symmetry axis? Use left color? Right color? Blend? Separate "center" color?
- **Asymmetric shapes:** If a shape is marked "symmetry broken" and is asymmetric, how does L/R palette work? Based on which side of axis its center is on?
- **Multi-axis symmetry:** If using both horizontal and vertical symmetry (4 quadrants), does this become Top-Left/Top-Right/Bottom-Left/Bottom-Right? Or still just L/R?
- **UI for assignment:** How does a user specify "this shape uses palette slot 2-LEFT"? Dropdown? Separate L/R color pickers? Automatic based on position?
- **Inheritance with L/R:** When a child component inherits L/R palette from parent, does it inherit both values and automatically apply them based on position?
- **Preview mode:** Is there a way to preview "show all left colors" vs "show all right colors" to verify the distinction is working?
- **Export implications:** For angle variants (#4 lighting), do L/R colors stay fixed to world left/right, or rotate with the asset?
- **Interaction with hard-coded colors:** Can a component have: hard-coded color A, shared palette slot 1, AND L/R palette slot 2, all in different regions?
- **Naming:** If slots are semantic, how do you name L/R variants? "Primary-Left"? Or keep numbers and add L/R as separate dimension?

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

**IMPLEMENTATION QUESTIONS:**
- **Data representation:** How are vector paths encoded in TS? As arrays of points? Objects with typed path commands? SVG-path-like strings?
- **Code structure:** Is it a class, functions, or just data + render function? Example: `export class CharacterAsset { render(ctx: CanvasRenderingContext2D): void }`
- **Feature preservation:** Which features are preserved in export?
  - Symmetry: Export mirrored geometry or symmetry operations?
  - Components: Export flattened or preserve assembly structure?
  - Palettes: Export baked colors or palette lookup system?
  - Lighting: Export pre-lit or lighting computation code?
  - Tri-pixels: Export rasterization algorithm or pre-rasterized?
- **Rendering backend:** Does exported code assume Canvas2D? Could it target WebGL? SVG? Abstract rendering interface?
- **Dependencies:** Does exported code import any libraries, or is it pure TS with only standard browser APIs?
- **Configuration parameters:** Can render() function accept parameters? (size, palette override, angle for lighting, etc.)
- **Multiple assets:** If exporting a project with many assets, is it one TS file with multiple classes/functions, or separate files?
- **Type definitions:** Are there shared types exported (Point, Color, Shape, etc.) or are they defined per-asset?
- **Optimization:** Is exported code optimized (minified, tree-shaken) or readable (formatted, commented)?
- **Versioning:** How is export format versioned? If export format changes, can old TS files still be used?
- **Build integration:** How does this integrate with build systems? Is there a companion npm package? Or just copy-paste the TS file?
- **Runtime usage:** Is the expected usage: (a) call render() to get pixels in-game, or (b) call render() at build time to pre-generate PNGs?
- **Edit round-tripping:** If someone edits the TS file by hand, can it be imported back into the editor? Or is it one-way export?
- **Asset variants:** If using component assembly to create many variants, are all variants exported in one file, or separate exports?

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

### 10. Negative Space Paths
**Description:** Ability to create transparent cutouts within solid shapes by drawing negative space (subtractive) paths.

**Key aspects:**
- **Path subtraction**: Draw paths within a solid shape to create transparent holes
- **Layered composition**: Multiple negative paths can be combined within a single shape
- **Render-time processing**: Negative paths are applied during rendering (not in storage)
- **Symmetry compatible**: Negative paths respect the same bilateral symmetry as positive paths
- **Use cases**: Creating hollow shapes, windows, complex silhouettes, and decorative cutouts
- **Boolean operation**: Essentially a subtract/difference operation in traditional vector graphics terms

### 11. Sprite Sheet Metadata Export
**Description:** Export JSON documents alongside sprite sheets containing comprehensive metadata for all images including location, rotation, and composition data.

**Key aspects:**
- **Frame metadata**: Position and dimensions of each sprite in the sheet (x, y, width, height)
- **Rotation data**: Rotation angle for each frame (for auto-generated rotation sets)
- **Composition information**: Component hierarchy, palette data, symmetry settings
- **Asset references**: Links between sprites and their source asset definitions
- **Animation sequences**: Grouping of related frames (rotation sets, animation frames)
- **Game engine integration**: Format compatible with TypeScript game engines (especially Excalibur.js)
- **Reverse lookup**: Ability to trace rendered sprite back to its source vector definition

### 12. Automatic Palette Generation
**Description:** Tool to automatically generate a complete color palette from a single input color using color theory algorithms.

**Key aspects:**
- **Single color input**: User picks one base color
- **Hexadic color scheme**: Generates palette using hexadic (6-color) harmony based on color wheel
- **Shade variations**: Creates multiple shades/tints of each color (lighter and darker variants)
- **Instant palette population**: Automatically fills palette slots with generated colors
- **Customizable parameters**: Optional controls for saturation, brightness adjustments
- **Quick start workflow**: Enables fast palette creation without manual color selection
- **Override friendly**: Generated colors can be individually adjusted after generation

### 13. Undo/Redo System
**Description:** Editor history system allowing users to undo and redo editing actions.

**Key aspects:**
- **Action history**: Track editing operations (add/delete/modify paths, palette changes, etc.)
- **Undo**: Step backward through history to reverse actions
- **Redo**: Step forward through undone actions
- **History limit**: Reasonable cap on history depth (memory management)
- **State preservation**: Maintain undo stack during editing session
- **Session-only**: History not persisted to file (in-memory only)

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

**⚠️ CRITICAL SPECIFICATION GAP**: JSON file format schema undefined
- File format is marked "TBD exact schema" but is critical for Phase 0-1 implementation
- Need TypeScript type definitions for all data structures
- Need: AssetFile, Unit, Palette, Shape, Path, Point, Color types
- Need: Decision on component library storage (embedded vs. separate files)
- See specification-review-round-2.md section #13 for proposed schema

### Rendering & Performance
- **Real-time preview required**: All features must render in real-time during editing
- **Separate edit/render areas**: Editing UI separate from rendering preview area
- **Rendering library-based**: Editor uses the same rendering library that exports use
- **Expected complexity**: Dozens of units, each with dozens of vertices
- **Batch export**: Likely the only export mode (generate all variants at once)

### Additional Critical Questions

#### File Format & Persistence
16. **Project structure:** Single monolithic file, or directory with multiple files?
17. **Binary vs text:** Should the native format be JSON (human-readable, easier to edit/diff) or binary (smaller, faster)?
18. **Version control friendly:** Should the format be designed for git diffing (separate files per asset, stable serialization order)?
19. **Asset references:** How are component references stored? Relative paths? UUIDs? Names?
20. **Undo/redo:** Is undo stored in-memory only, or persisted with the file? How many undo levels?

#### Canvas & Workspace
21. **Canvas size:** Is canvas size fixed per-asset, or infinite with dynamic bounds?
22. **Units:** What units are used for coordinates? Pixels? Abstract units? Grid cells?
23. **Origin:** Where is (0, 0)? Top-left, center, or bottom-left?
24. **Zoom & pan:** What are the zoom limits? Can you zoom to pixel-level precision?
25. **Multi-asset workflow:** Can you have multiple assets open at once? Switch between them?

#### Drawing & Editing Tools
26. **Shape primitives:** What shapes are supported? Only paths? Circles, rectangles, polygons as first-class shapes?
27. **Bezier curves:** Are curves supported, or only straight-edged polygons?
28. **Fill & stroke:** Can shapes have both fill and stroke? Stroke width? Line caps/joins?
29. **Drawing tools:** Freehand pen? Click-to-place points? Shape tools (rectangle, circle)? All of the above?
30. **Selection tools:** Lasso, rectangle select, click-to-select? Can you select multiple shapes?
31. **Transform tools:** Dedicated move, rotate, scale tools? Or just manipulate points directly?
32. **Color picker:** How are colors specified? RGB sliders? Hex input? Preset swatches? Eyedropper from reference image?

#### Component System Workflow
33. **Creating components:** Is there a "convert selection to component" action? Or do you create components in a separate mode?
34. **Component editor:** Is there a separate UI for editing a component vs editing an assembly?
35. **Preview context:** When editing a component, can you preview it in the context of an assembly?
36. **Component library UI:** How is the component library displayed? Thumbnail grid? Tree view? Searchable list?

#### Export Workflows
37. **Export profiles:** Can users save export presets (resolution, format, features enabled)?
38. **Batch export:** Can you export multiple assets/variants at once?
39. **File naming:** For variant exports (angles, assemblies), how are files named? Manual? Auto-generated pattern?
40. **Export preview:** Is there a preview dialog before export, or direct export?

#### Performance & Scalability
41. **Editor performance target:** Should the editor handle 100 shapes? 1000? 10,000?
42. **Rendering optimization:** Is dirty-region rendering needed, or just redraw everything each frame?
43. **Component caching:** Are component renders cached, or re-rendered every frame?
44. **Large palettes:** If supporting many palette slots, is there a performance concern with palette lookups?

#### Platform & Distribution
45. **Platform:** Desktop app (Electron, Tauri), web app, or both?
46. **Offline support:** Must it work offline, or is internet connectivity required?
47. **Installation:** Installed application, or browser-based?
48. **Save location:** Local filesystem, cloud storage, browser localStorage?
49. **Collaboration:** Is multi-user editing in scope? Or strictly single-user?

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
6. **Negative Space Paths** (#10)
7. **Full Palette Inheritance** (advanced version of #6)
8. **Left/Right Palette Variation** (#7)
9. **Sprite Sheet Metadata Export** (#11)

### Nice-to-Have
10. **Faux 3D Lighting** (#4)
11. **Automatic Palette Generation** (#12)
12. **Undo/Redo System** (#13)

### Infrastructure (Will Get Done)
13. **TypeScript Rendering Library** (#8)
14. **Client-Side Web Application** (#9)

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
- 6d. Stroke and fill color assignment (paths reference palette slots separately for stroke/fill)
- 6e. Palette editor UI
- 6f. Inheritance visualization in editor

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

### Feature 10: Negative Space Paths
**Sub-features:**
- 10a. Data model for negative vs. positive paths (path type flag)
- 10b. Rendering engine path subtraction (boolean difference operation)
- 10c. Editor UI for toggling path type (positive/negative)
- 10d. Visual differentiation in editor (show negative paths distinctly)
- 10e. Symmetry application to negative paths
- 10f. Multiple negative path composition within single shape

**Dependencies:**
- Requires: Basic vector path rendering (#8)
- Requires: Symmetry system (#1) for consistent mirroring
- Works with: All shape-based features
- Independent of: Grid system, palette system, component system

**Priority:** MEDIUM (enhances visual possibilities without adding complexity)

### Feature 11: Sprite Sheet Metadata Export
**Sub-features:**
- 11a. JSON schema design for metadata format
- 11b. Frame location and dimension tracking (x, y, width, height)
- 11c. Rotation angle metadata for each frame
- 11d. Component hierarchy and composition data export
- 11e. Palette information and color references
- 11f. Animation sequence grouping (rotation sets, frame collections)
- 11g. Source asset linking (sprite → vector definition)
- 11h. Game engine compatibility layer (Excalibur.js format)

**Dependencies:**
- Requires: TypeScript Rendering Library (#8) for sprite sheet generation
- Requires: Component System (#5) for composition data
- Works with: Palette system (#6) for color metadata
- Works with: Rotation generation (#4 lighting) if implemented
- Independent of: Editor UI features

**Priority:** MEDIUM (essential for game engine integration and workflow efficiency)

### Feature 12: Automatic Palette Generation
**Sub-features:**
- 12a. Color theory algorithm implementation (hexadic harmony calculation)
- 12b. Shade/tint generation (lighter and darker variants)
- 12c. Single color picker UI for base color input
- 12d. Palette population logic (auto-fill slots with generated colors)
- 12e. Customization controls (saturation, brightness adjustments)
- 12f. Preview of generated palette before applying
- 12g. Integration with palette editor

**Dependencies:**
- Requires: Basic palette system (#6) to populate
- Works with: Palette editor UI from Feature #6
- Independent of: All other features (standalone tool)

**Priority:** NICE-TO-HAVE (workflow enhancement, not essential)

### Feature 13: Undo/Redo System
**Sub-features:**
- 13a. Action history data structure (stack of reversible operations)
- 13b. Command pattern implementation for all editable operations
- 13c. Undo operation (revert last action, move to redo stack)
- 13d. Redo operation (re-apply undone action)
- 13e. History limit configuration (memory management)
- 13f. UI indicators (undo/redo button states, keyboard shortcuts)
- 13g. Clear history on file save/load

**Dependencies:**
- Requires: Editor UI framework
- Works with: All editing operations (paths, palettes, components)
- Independent of: Rendering system

**Priority:** NICE-TO-HAVE (quality of life, not essential for MVP)

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
       ├─→ [3. Tri Pixels] ← rendering technique, independent
       └─→ [10. Negative Space] ← path subtraction, works with symmetry

Component System:
  [5. Component Assembly]
       ↓
  [6. Palette Inheritance]
       ├─→ [12. Auto Palette Generation] ← workflow tool for palette creation
       └─→ [7. L/R Palette Variation] ← requires both symmetry (#1) and palettes (#6)

Export & Integration:
  [11. Sprite Sheet Metadata] ← depends on rendering (#8) + component system (#5)
       └─→ JSON export with frame locations, rotations, composition data

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

## Project Roadmap

1. **Complete Feature Document** - Finish defining all features, priorities, and architectural decisions
2. **Produce Specification** - Create detailed technical specification from feature document
3. **Produce Todo Document** - Break down specification into actionable development tasks

## Notes

- Target: 2D top-down games
- Philosophy: Drop features that exceed "simple project" threshold
- All features are candidates, not commitments
- Compatibility issues are opportunities to simplify or clarify, not necessarily blockers

---

## Implementation Readiness Assessment

This brainstorming document has identified **8 feature ideas**, but many critical details remain unspecified. Before implementation can begin, the following areas **must** be clarified:

### CRITICAL PATH DECISIONS (Must resolve before ANY coding)

1. **Platform choice** (Q45-49): Web vs desktop fundamentally affects architecture, rendering approach, and file I/O
2. **Native file format** (Q9, 16-19): Defines data structures throughout the entire application
3. **Drawing model** (Q26-27): Bezier curves vs polygons affects rendering pipeline complexity
4. **Canvas & coordinate system** (Q21-23): Affects all geometric calculations and transformations

### HIGH PRIORITY SPECIFICATIONS (Must resolve early)

#### Symmetry System
- Data model: mirrored storage vs computed (Q: Data model)
- Breaking mechanism and re-enabling behavior (Q: Breaking symmetry, Undo/redo)
- Selection and manipulation UI (Q: Selection model)
- **Blocker:** Without these specs, can't implement core editing

#### Component Assembly
- Storage format: separate files vs embedded (Q: Component storage)
- Update propagation: live vs baked (Q: Component updates)
- Transform capabilities (Q: Transform constraints)
- **Blocker:** Affects file format and data model fundamentally

#### Palette System
- Slot count and semantics (Q: Palette slot count, Slot semantics)
- Fallback behavior (Q: Default/fallback colors)
- UI for slot assignment (Q: Color assignment UI)
- **Blocker:** Affects data model and export format

### MEDIUM PRIORITY (Can defer to later milestones)

#### Triangular Grid (#2)
- Grid topology and orientation (Q: Grid topology, Grid orientation)
- Coordinate system relationship (Q: Coordinate system)
- Most questions can be deferred if this feature is implemented later

#### Triangular Pixel Rendering (#3)
- Rasterization algorithm (Q: Rasterization algorithm)
- Size specification (Q: Size specification)
- Can be implemented as export-only feature, doesn't affect editing

#### Faux 3D Lighting (#4)
- Lighting model complexity (Q: Lighting model)
- Angle generation count (Q: Angle specification)
- Can be entirely export-time feature

#### Left/Right Palette (#7)
- Region determination (Q: Region determination)
- Extends palette system, so depends on base palette being specified first

### LOW PRIORITY (Can be decided during implementation)

#### TypeScript Export (#8)
- Code structure and API (Q: Code structure)
- Feature preservation decisions (Q: Feature preservation)
- Export format can evolve independently of editor

#### UI/UX Details
- Tool selection, keyboard shortcuts, layout
- Can be prototyped and refined iteratively

### RECOMMENDED NEXT STEPS FOR SPECIFICATION

1. **Answer Critical Path questions** - Make platform and file format decisions
2. **Create minimal data model** - Define core types (Point, Shape, Asset, Component, Palette)
3. **Specify MVP feature set** - Choose subset of 8 features for first version
4. **Document core workflows** - Write step-by-step user scenarios for key tasks
5. **Create wire frames** - Sketch basic UI layout showing tools and panels
6. **Define export specifications** - Pick specific export formats and parameters
7. **Write technical architecture doc** - Rendering pipeline, state management, file I/O
8. **Prototype risky features** - Test tri-pixel rendering, lighting, or other complex features before committing

### KEY AMBIGUITIES TO RESOLVE

**Symmetry + Components interaction:**
- Do components have inherent symmetry, or is symmetry only at assembly level?
- Can you break symmetry on a component instance in an assembly?
- If a symmetric component is placed asymmetrically in an assembly, what happens?

**Palette inheritance depth:**
- Single level (component → assembly) or unlimited nesting?
- How are conflicts resolved if component A and B both want to provide palette to C?

**Export feature coverage:**
- Which features are "edit-time only" vs "preserved in export"?
- TypeScript export in particular: does it replicate ALL features, or just geometry?

**Grid alignment enforcement:**
- Is the grid optional or mandatory?
- Can geometry exist off-grid, or is everything snapped?
- How do components from different grid scales interact?

### SIMPLIFICATION OPPORTUNITIES

To keep this project "super simple," consider these potential cuts:

1. **Feature #4 (Lighting)** - Most complex feature, requires angle variants, gradient computation
2. **Feature #7 (L/R Palette)** - Adds significant complexity to already-complex palette system
3. **Nested components** - Limit to single-level assembly (parent + children, no grandchildren)
4. **Bezier curves** - Restrict to straight-edged polygons only
5. **Advanced transforms** - Limit to translation only, no rotation/scale/skew
6. **Multi-axis symmetry** - Support only bilateral (left/right), not radial or quadrant

Cutting these would significantly reduce implementation complexity while preserving core vision of "symmetrical vector sprite editor with component system."
