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
**Description:** Construction aid using triangular/hexagonal grid for control point snapping.

**Key aspects:**
- Control points and vertices snap to triangular lattice
- Facilitates easy drawing of triangles and hexagons
- Grid-aligned geometry construction

**Open questions:**
- What is the grid scale/size?
- Is grid snapping always on, or toggleable?
- How does grid relate to final output resolution?

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
**Description:** Aesthetic style using large visible triangular "pixels" instead of square pixels.

**Key aspects:**
- Rasterize with chunky triangular pixel look
- Output to standard image formats (PNG, etc.) but with tri-pixel aesthetic
- Each "tri-pixel" rendered as multiple regular pixels forming triangle

**Open questions:**
- What is the tri-pixel size relative to output resolution?
- Do tri-pixels align with construction grid (#2) if both features are used?
- Are tri-pixels anti-aliased or hard-edged?

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
**Description:** Shared palette slots can have different values for left vs right side.

**Key aspects:**
- Enables asymmetric coloring on symmetric geometry
- Child components use left/right palette slots
- Example: red left glove, blue right glove on symmetric character

**Open questions:**
- How are left/right regions defined? (automatic from symmetry axis? manual painting?)
- Does this apply only to shared palette or also local colors?
- Can this support more than bilateral symmetry (e.g., radial with 4+ segments)?

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
