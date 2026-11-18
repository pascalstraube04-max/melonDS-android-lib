# Vulkan DS Edge-Coverage Investigation

Branch: `coverage-fix-engine-investigation`

## Goal

Reproduce the Nintendo DS polygon edge-inclusion rules in the Vulkan raster path, especially the remaining one-pixel gaps reported along the lower edge of the lower DS screen, without reducing the requested 6x render scale or introducing per-fragment work that is unnecessarily expensive.

## Confirmed architecture

- The accelerated frontend builds `SharedGraphicsScene.Vertices`, `Triangles`, `Draws`, and `EdgeIndices`.
- `GraphicsSceneVertexBuffer` is already uploaded and used as a vertex-input buffer.
- `GraphicsEdgeIndexBuffer` is already uploaded and used as an index buffer for edge rendering.
- `GraphicsPolygonDraw` retains per-draw ranges: `firstVertex`, `vertexCount`, `firstEdgeIndex`, and `edgeIndexCount`.
- The graphics descriptor-set layout currently exposes storage buffers at bindings 0, 2, and 5. Binding 0 is triangle data; binding 2 is toon data; binding 5 is clear data.
- `updateGraphicsDescriptorSet()` writes those descriptors and caches their buffer handles.
- The raster fragment shader receives triangle flags, but does not currently apply DS scanline edge-inclusion rules from the boundary flags.
- The current scene vertex buffer is created with `VK_BUFFER_USAGE_VERTEX_BUFFER_BIT` only. It is not currently available as an SSBO.
- The scene vertex data is built through `resolveFixedCoords()`. When coverage expansion is enabled, those positions are the expanded positions, not a preserved copy of the original polygon edge coordinates. Therefore the existing scene vertex buffer alone is not sufficient to reconstruct the original DS edge rules.
- The edge-index data is `u16` and currently has index-buffer usage. It cannot safely be declared as a normal GLSL `uint[]` SSBO without changing its representation or adding a compatible packed decoding scheme.

## DS reference behavior

The software renderer's `RenderPolygonScanline` uses slope/direction-dependent inclusion rules, including special handling for vertical edges, negative X-major edges, flat edges, and the final scanline. Vulkan's standard triangle top-left fill convention is not a direct equivalent. Coverage expansion is therefore an approximation rather than a faithful implementation.

## Important depth caveat

The Vulkan depth allowance constants were corrected to the DS equal-depth tolerance magnitudes (W: 255, Z: 512) in the existing branch. This is not an exact reproduction of the DS symmetric absolute-difference test: the current Vulkan depth comparison remains one-sided. The change is committed but has not been built or device-tested.

## Follow-up findings (2026-11-18)

- The BetterPolygons path inserts a synthetic center vertex at offset 0, then appends the original polygon perimeter vertices. The old edge-loop call closed over the center and perimeter, incorrectly generating center-to-perimeter diagonals as if they were source polygon edges.
- Fixed on this branch: appendEdges now accepts a starting vertex offset. Polygon perimeter edges skip the synthetic center (offset 1 when the center was successfully emitted); line primitives still start at offset 0. This corrects edge-index topology but does not itself remove raster cracks.
- Confirmed the more important limitation: resolveFixedCoords() selects expandedX/Y whenever coverage correction is active. Consequently, even perimeter vertices stored in SharedGraphicsScene.Vertices carry expanded positions. The edge-index list identifies perimeter topology, but not original unexpanded edge coordinates.
- GraphicsSceneVertexBuffer currently stores 64-byte GraphicsVertexGpu records and is vertex-input-only. A reuse route is to extend each record with original source X/Y, enable storage-buffer usage on the same allocation, and expose it to the fragment stage. This avoids a second full scene-vertex buffer, but increases the existing vertex record size and requires descriptor/stride updates.
- The fragment shader also needs a stable per-triangle mapping to its polygon's original perimeter range. GraphicsPolygonDraw has these ranges on the CPU, but they are not copied into TriangleGpu. A compact pair of per-triangle range values is a candidate; indexing must be verified against merged draws and every render pass.
- Binding 0 triangle SSBO is already bound for vertex stage. Making it fragment-visible is comparatively small, but adding polygon range fields changes the 120-byte TriangleGpu std430 contract and embedded SPIR-V. Do not change the C++ layout without regenerating and validating every affected shader binary.
- The edge index stream is u16. Reusing it in GLSL requires an explicit 16-bit storage feature/extension or a packed-u32 representation; do not reinterpret it as uint[] with the current byte layout.

## Current safe next implementation sequence

1. Preserve original source positions separately from expanded raster positions in CPU scene vertices.
2. Keep the corrected edge-index topology and add per-triangle polygon range metadata only after confirming all triangle creation paths.
3. Decide whether the existing scene vertex allocation can carry both vertex and storage usage across all render contexts.
4. Add a fragment-stage edge test that reproduces DS scanline inclusion at the DS pixel grid, using original perimeter coordinates and explicit boundary-edge selection.
5. Regenerate all embedded SPIR-V variants from the project's actual shader build pipeline; do not hand-edit binary layouts.
6. Build only after explicit user approval, then test 6x on Adreno 750 with repeatable lower-screen crack and snow/shadow flicker scenes.

## Implementation options to evaluate

1. Preserve original, unexpanded polygon vertex positions alongside expanded raster positions during accelerated-scene construction.
2. Preserve per-polygon source-edge metadata and a stable mapping from emitted triangles/fragments back to the polygon's original edge segments.
3. Decide whether to expose this metadata through:
   - an extended existing scene-vertex buffer with storage-buffer usage, or
   - a compact dedicated edge metadata buffer.
4. If using the existing scene vertex buffer as an SSBO, update its usage flags, descriptor layout, descriptor pool sizing, descriptor writes/cache invalidation, and host-write-to-fragment-read synchronization. Keep vertex-input usage because the same buffer remains a vertex source.
5. Avoid directly binding the current `u16` edge-index stream as `uint[]` without a deliberate layout-compatible representation.
6. Derive the fragment-to-DS-pixel/edge test at the requested render scale, including the DS bottom-scanline and slope rules. Do not assume the existing `dsPixelCenterDelta()` helper solves edge coverage.
7. Keep the new path gated behind the existing coverage-fix enable state during development, so it can be compared against the current path.

## Constraints and validation

- Preserve 6x rendering.
- Avoid broad depth shifts: the current coverage depth bias affects all emitted vertices of an expanded polygon, not just its boundary.
- Do not claim the depth change or a future edge fix works until it has been built and tested on the target Adreno device.
- Do not trigger GitHub Actions, CI, or APK builds until the user explicitly authorizes a build.

## Shader-access validation (2026-11-18)

- The raster vertex shader forwards only flat per-triangle info already packed into vertex attributes; it does not forward a stable triangle index to the fragment shader.
- The raster fragment shader currently declares the toon SSBO at binding 2, but does not declare or read the triangle SSBO at binding 0.
- Descriptor binding 0 is currently vertex-stage-visible. Fragment visibility is enabled only for the normalized-texture-descriptor configuration, not as a general guarantee. Any fragment-side triangle metadata path must update this stage visibility deliberately and remain valid for both texture-sampling configurations.
- GraphicsSceneVertexBuffer allocation is shared by ordinary and retained render contexts and is double-sized for ping-pong rendering when context == nullptr. Its allocation currently uses vertex-buffer usage only and its 64-byte stride is asserted in ensureGraphicsSceneVertexBuffer().
- TriangleGpu is populated in the per-polygon emission path. GraphicsPolygonDraw is queued separately and records per-draw vertex/edge ranges, but that draw record is not directly present in the fragment shader interface.
- Therefore, adding an SSBO read in the fragment shader is not a local GLSL-only change: it requires a stable fragment-to-triangle association, descriptor stage/layout updates, matching C++/std430 layout, and regenerated embedded SPIR-V for the project's actual shader variants.
- The least invasive next experiment is to determine whether the existing per-vertex flat metadata can encode a triangle-local boundary reference without expanding global buffer interfaces. If not, compare a compact per-triangle metadata extension against a dedicated packed edge buffer before implementing either.

## Revised low-overhead shader path (2026-11-18)

A fragment-stage SSBO lookup is not the preferred first implementation. The existing raster vertex shader already reads the per-triangle SSBO at binding 0, and `TriangleGpu` already carries the three raster-space vertex positions plus the three boundary-edge bits in `flags`. This gives a potentially smaller route:

1. Extend each `TriangleGpu` with the three original, unexpanded screen-space vertex positions (six floats). Keep the existing x/y triplets as the expanded raster positions used by Vulkan coverage.
2. In the vertex shader, use the existing triangle index / SSBO lookup path to fetch the original perimeter coordinates and forward them as flat varyings. Verify the exact triangle-index calculation against every draw mode before relying on it.
3. In the fragment shader, use `gl_FragCoord` and the flat original perimeter coordinates to evaluate only edges marked by `kTriangleFlagBoundaryEdge0/1/2`; apply DS edge-inclusion rules only to original polygon boundaries, leaving internal fan diagonals to Vulkan's normal top-left rule.
4. Keep the correction gated by the existing coverage-fix flag so default rendering remains unchanged. Avoid any new fragment SSBO descriptor or per-fragment buffer lookup if the vertex-stage forwarding works.

This path still needs careful handling of clipping, perspective interpolation, viewport scaling, and the DS scanline edge cases documented above. It is an implementation hypothesis, not yet a verified fix. Before editing shader interfaces, validate `gl_VertexIndex`, `pc.triangleBase`, triangle-buffer indexing, and all shader variants in the actual pipeline.


## Draw-index mapping verification (2026-11-18)

The proposed vertex-stage SSBO lookup is not necessary for associating raster vertices with their triangle:

- `appendTriangle()` pushes exactly one `TriangleGpu` and then appends exactly three corresponding `GraphicsVertexGpu` records in vertex order (0, 1, 2).
- The raster draw uses `vkCmdDraw(mergedTriangleCount * 3u, 1u, firstDraw.firstTriangle * 3u, 0u)`. Thus the first vertex for triangle N is exactly vertex index `N * 3`, and each triangle's three emitted vertex records are contiguous.
- Each emitted vertex already carries the same flat `triangle.flags`, including the three boundary-edge bits and coverage-fix bit.
- Consequently, if original boundary-edge coordinates can be resolved reliably on the CPU, they can be copied into the three per-triangle `GraphicsVertexGpu` records and forwarded as flat vertex outputs. This avoids a triangle-SSBO lookup, descriptor changes, and fragment-stage storage-buffer access.
- `TriangleGpu` and `GraphicsVertexGpu` are separate upload streams. The existing raster vertex attributes are sourced from `GraphicsVertices`; the triangle SSBO is not required to recover the triangle number for this draw.

Remaining blocker: `appendTriangle()` receives positions from `SharedGraphicsScene.Vertices`, which have already passed through `resolveFixedCoords()` and can be coverage-expanded. The source `Polygon` pointer and its original `Vertices[]` are available during polygon emission, but a correct mapping must account for clipping, generated vertices, triangulation, and which emitted boundary segments still correspond to original DS edges. Do not simply copy the first three source vertices: that is incorrect for polygons with more than three vertices and for clipped polygons.

Implementation direction revised: first trace the frontend's clipping and triangulation provenance so each emitted boundary edge can carry its true unexpanded endpoints. Then add compact per-triangle boundary endpoint attributes to `GraphicsVertexGpu` (or an equivalent CPU-side record), forward them flat, and implement the DS edge test. The vertex-to-triangle draw mapping itself is now confirmed; source-edge provenance remains unverified.


## Source polygon provenance trace (2026-11-18)

The accelerated frontend does not triangulate from an opaque clipped mesh: for each non-line render polygon it iterates `polygon->Vertices[0..NumVertices)`, creates one scene perimeter vertex for each non-null source entry, and stores those indices in `polygonVertexIndices`. The optional BetterPolygons center is inserted separately at offset 0; the perimeter remains in source order after it.

Triangulation preserves this perimeter order:
- BetterPolygons fan triangles are `center, perimeter[i-1], perimeter[i]`; the perimeter side is marked BoundaryEdge0.
- The ordinary fan uses the same perimeter order; the first and last fan triangles additionally mark the two polygon closing sides as BoundaryEdge2 and BoundaryEdge1.
- Therefore, for these frontend-generated triangle fans, the boundary-edge bit identifies which pair of triangle corners is a source polygon perimeter segment. Internal fan diagonals are not marked as boundary edges.

The original (unexpanded) screen coordinates can be captured at the same time each scene perimeter vertex is created: `resolveFixedCoords()` currently chooses expandedX/Y for render coordinates when correction is active, but the underlying `Vertex*` is still available and its `HiresPosition[]` is read in the non-expanded branch. The coordinate provenance should be stored explicitly before `resolveFixedCoords()` discards that distinction. The source vertex list is already the polygon presented to this accelerated frontend; this trace found no separate clipping/triangulation stage inside `BuildAcceleratedScene()` between the polygon list and the emitted fan.

This substantially narrows the implementation: preserve a second pair of unexpanded screen coordinates per `AcceleratedSceneVertex`, copy the three source-corner coordinate pairs into each emitted raster triangle's three vertex records, and forward them as flat varyings. The boundary flags already distinguish the polygon perimeter from fan diagonals. Remaining checks before coding are the exact `HiresPosition` scale/viewport convention for all `UseHiresCoordinates` modes, whether any null source vertices can shift the compacted perimeter indices, and the complete vertex-input attribute/stride contract. No shader or runtime behavior has been changed by this investigation.
