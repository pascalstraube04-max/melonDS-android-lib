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
