# Coverage Fix / Vulkan Engine Investigation

Branch: `coverage-fix-engine-investigation`

## Scope

This branch is an isolated investigation branch. No renderer behavior has been changed yet.

## Confirmed code paths

### Coverage is disabled for an entire polygon when any vertex lies outside the viewport

In `src/GPU3D_AcceleratedFrontend.cpp`, inside `BuildAcceleratedScene`, the current implementation checks each polygon vertex against `MaxFixedX` / `MaxFixedY`. If any vertex is outside the viewport, it clears `draw.CoverageFixState`.

This is conservative: it avoids expanding geometry that crosses the viewport boundary, but it also means a polygon with only a small portion outside loses coverage expansion everywhere. This is a plausible source of residual edge gaps, but it is not yet proven to explain the user's observed lower-screen artifacts.

### Expansion geometry

`ComputeAcceleratedCoverageExpandedVerticesFixed` moves each vertex radially away from the polygon's average center, then clamps the resulting coordinates to the viewport bounds.

This is simple and inexpensive, but is not equivalent to a true offset of the polygon boundary. For elongated, concave, thin, or clipped polygons, radial expansion can produce uneven coverage and may not close every gap consistently.

### Depth adjustment

In `src/GPU3D_Vulkan.cpp`, `CoverageFixDepthBias` is converted to a normalized-depth scale using the 24-bit depth range. Later, the value is subtracted from the Z values of the expanded polygon vertices.

This is a polygon-wide depth modification, not a depth-only adjustment of newly added edge fragments. That can change depth ordering against nearby geometry and is consistent with the reported shadow/snow flicker at high values. Exact units and observed thresholds depend on the UI-to-renderer conversion and must be traced before changing the algorithm.

### Scene-build allocation

The Vulkan scene build reuses capacity for several vectors, but creates a local `std::unordered_map` for texture lookup on each scene build. This is a candidate for profiling, not proof of a performance bottleneck. Any optimization must be driven by measurements and preserve cache invalidation correctness.

## Next engineering steps

1. Trace the full clipping pipeline and determine whether the frontend receives original or already-clipped vertices in each render path.
2. Add focused tests or diagnostic counters for polygons rejected from coverage because of viewport bounds.
3. Prototype a clip-aware coverage expansion on a separate branch or behind a diagnostic toggle. Do not simply remove the current safety check.
4. Compare radial expansion with a robust polygon-offset method, including degenerate, concave, thin, and viewport-crossing polygons.
5. Trace the UI value conversion and all depth paths before changing depth semantics.
6. Profile scene build and Vulkan submission separately before attempting allocation or GPU optimizations.
7. Add German localization only after identifying the actual localization resource and preserving existing translations.

## Validation requirements

A change is not considered complete until it passes build checks and visual tests on-device. Required visual cases include Pokémon Platinum's lower-screen edge, snow, building shadows, repeated/clamped textures, and scenes with polygons crossing viewport edges. Performance must be measured against the same baseline scene and device.

## Important caution

The observed relationship between Coverage Expand / Repeat / Clamp and the user's artifacts is strong evidence that the coverage path matters. It does not by itself prove which internal branch is responsible. Avoid broad changes to clipping or depth until the path is traced and test coverage exists.


## Follow-up: clipping semantics need care

The frontend's viewport-boundary guard checks projected high-resolution screen coordinates. The underlying `Vertex::Clipped` flag is set by the 3D clipping pipeline (`ClipSegment`) when a vertex is generated at a clip-plane intersection; it is not a direct replacement for the frontend's viewport-boundary test. Do not replace the viewport test with a simple `Clipped` check.

The GPU pipeline also wraps the DS final screen coordinates (`FinalPosition`) to hardware-sized fields, while the accelerated frontend uses signed high-resolution coordinates. This distinction needs to be preserved when tracing edge behavior.

The current radial expansion clamps each expanded vertex individually to the viewport. That can change polygon shape at the boundary. A safer prototype should clip the polygon in screen space first, then offset the resulting boundary, or selectively expand only edges whose offset remains inside the viewport. This requires handling winding, concavity, zero-length edges, and perspective-correct interpolation of all attributes; a naive per-vertex replacement is not safe.


## Follow-up: verified clipping order and 6x Android target

The core DS polygon clipper in `src/GPU3D.cpp` runs the generated polygon through Z, then Y, then X clipping (`ClipPolygon`). `ClipSegment` interpolates position and, when requested, color and texture coordinates, and marks generated intersection vertices with `Clipped = true`.

This confirms that the accelerated frontend normally receives geometry after the DS homogeneous clip stage. However, that stage is not the same as the later high-resolution viewport guard: the frontend still performs a separate bounds check against its configured target dimensions. The two stages must remain distinct.

The frontend guard converts coordinates using either:
- `HiresPosition * Scale` when high-resolution coordinates are enabled, or
- `FinalPosition << 4` otherwise.

It then disables coverage for the whole polygon if any converted coordinate is outside `[0, MaxFixedX] × [0, MaxFixedY]`. This is the precise condition to instrument next. Do not infer from `Vertex::Clipped` alone that a polygon touches the high-resolution viewport edge.

The accelerated scene builder also stores signed floating-point screen coordinates only on the explicit preserve-signed path; its normal scene vertex path clamps coordinates to the target. Any clip-aware fix must preserve the existing distinction between signed screen-space geometry and packed/clamped fixed coordinates.

### Target hardware and performance envelope

The primary validation target is the user's Samsung Galaxy S24 Ultra (Snapdragon 8 Gen 3 for Galaxy), maximum display resolution, WatermelonDS at 6x internal DS resolution, with the device's Game Booster and in-game performance profiles selected. This is a sustained-load target, not a short peak benchmark.

At 6x, the internal pixel count is 36x the native DS pixel count. Renderer changes therefore need to avoid unnecessary per-frame allocations, extra full-screen passes, and avoidable overdraw. The thermal objective is stable long-session performance before thermal throttling, not maximum momentary throughput. Lower scales remain regression checks.

No device profiling or Android build has been performed yet. The 6x profile is a validation requirement, not a claim that the current branch has passed it.

### Next diagnostic step

Add a low-overhead, opt-in diagnostic that counts polygons rejected by the viewport guard, with separate counters for left/right/top/bottom overflow and a bounded sample of polygon/vertex coordinates. Keep it disabled by default and avoid per-frame log spam. Use this to determine whether the user's lower-screen artifacts correlate with polygons rejected solely because one or more vertices cross a viewport edge. Only then prototype selective edge-aware expansion.


### Android repository integration and CI build path

The Android repository `pascalstraube04-max/WatermelonDS` currently pins its `melonDS-android-lib` submodule to commit `65f4b25ef463b7ac4d9c8741f1a35e92af50e504`. This is exactly the current `DevelopSub` base commit in the user's engine fork, so the investigation branch starts from the Android app's currently pinned engine revision and does not introduce an unrelated engine baseline.

The Android repository's `.gitmodules` currently points the submodule URL at `https://github.com/SapphireRhodonite/melonDS-android-lib.git`, with branch metadata `GBARumble_PR`. To build a future engine change from the user's fork, the Android integration branch must update the submodule URL to the user's fork and update the gitlink to the exact engine commit. Merely creating a branch in the engine repository does not make the Android app build that branch.

A GitHub Actions workflow exists at `.github/workflows/main.yaml`. It triggers on pushes to any branch and supports manual dispatch. It checks out recursive submodules, sets up Java 21 and Android SDK, runs `./gradlew :app:regenerateVulkanSpirv :app:assembleGitHubNightlyRelease`, then uploads the APK and attempts to update the nightly release. The workflow references signing secrets (`MELONDS_KEYSTORE`, `MELONDS_KEYSTORE_PASSWORD`), so availability of those secrets in the user's fork must be confirmed before assuming the release build will complete. The workflow's artifact upload may still be usable if the build step is adjusted to avoid unavailable signing secrets, but no workflow change has been made.

This establishes a viable cloud build route for future test APKs, subject to submodule integration and signing configuration. No workflow has been run for the investigation branch and no APK has been produced.
## User test priority: lower edge of the lower display

The user reports that black coverage cracks are most noticeable along the bottom edge of the lower DS screen. Prioritize this boundary in the next diagnostic pass:

- Count coverage-fix rejections caused by yFixed > MaxFixedY separately from top, left, and right overflow.
- Record whether a polygon crosses only the lower boundary or multiple boundaries; keep samples bounded and diagnostics opt-in.
- Correlate lower-boundary rejections with the visible crack locations before changing the geometry path.
- Do not assume every lower-edge crack is caused by this guard: also inspect raster edge ownership / triangle boundary handling at the target's final scanlines if the guard counts do not correlate.

For the current Vulkan target, the viewport is 256 * scale by 192 * scale, with fixed-point maxima set to dimension * 16 minus one. The lower-edge test therefore concerns projected Y values beyond MaxFixedY, not merely vertices marked Clipped by the earlier DS homogeneous clipper.


## Follow-up: Vulkan fragment-depth and depth-test path verified

The Vulkan raster vertex shader (`GPU3D_Vulkan_GraphicsRasterShader.vert`) converts the polygon vertex Z value into normalized depth using the 24-bit DS depth range and forwards both linear and perspective depth variants. The fragment shader (`GPU3D_Vulkan_GraphicsRasterShader.frag`) writes the selected interpolated value directly to `gl_FragDepth` in the fragment-depth variants. There is no later shader stage that restores the original depth or limits the coverage bias.

The pipeline creation path in `GPU3D_Vulkan.cpp` confirms that opaque raster pipelines use depth testing and depth writes, with depth comparison selected between `VK_COMPARE_OP_LESS` and `VK_COMPARE_OP_LESS_OR_EQUAL` according to the emulated depth-compare mode. The selected depth attachment format is the first supported of D24_UNORM_S8, D32_SFLOAT_S8, or D16_UNORM_S8. Consequently, a polygon-wide Z shift can affect both depth comparison and the value stored in the depth attachment; the exact quantization depends on the selected device format.

This reinforces the requirement that coverage bias must not be used as an unrestricted global polygon-depth nudge. A safer design should separate geometric edge coverage from depth handling, or constrain any depth adjustment to a mathematically justified boundary-only operation that does not reorder unrelated surfaces. The current code inspection does not yet establish a safe replacement algorithm; device-format selection and both depth-interpolation modes must be included in subsequent validation.


## Follow-up: exact point of coverage depth-bias application

The current scene-emission path computes `coverageDepthBias = CoverageFixDepthBias * 16777215.0f`. For each draw, `effectiveCoverageDepthBias` is non-zero only when `sceneDraw.CoverageFixState.ApplyUserFix` is true. In `makeTriangleVertex`, the code subtracts that value from `vertex.Z` and clamps at zero. This helper is used for the generated triangle vertices of the polygon draw, including the regular polygon triangles and generated boundary/line triangles; it is not restricted to newly expanded outer-edge vertices.

The raster vertex shader then normalizes Z with `(z - ventanaIgual) / 16777216.0`, clamps it to [0,1], and forwards it as both linear and perspective depth. The fragment shader writes the selected interpolated depth to `gl_FragDepth`. Therefore the user bias is a pre-rasterization Z shift, not a small post-depth-test coverage correction. It can influence depth testing and depth writes across the whole affected polygon. It also means that a polygon's bias may be quantized differently depending on the physical depth attachment selected at device initialization (D24S8 first, then D32S8, then D16S8).

Engineering implication: do not increase the bias as the primary crack fix. A future solution should first make coverage geometry correct at the viewport boundary, then evaluate whether a depth adjustment is still necessary. If depth adjustment remains necessary, its scope and magnitude must be bounded and validated against overlapping opaque/translucent geometry, depth-write modes, both W/depth interpolation modes, snow, and building shadows. No code behavior was changed in this documentation-only follow-up.


## Follow-up: mathematical limitation of the current radial expansion

The implementation of `ComputeAcceleratedCoverageExpandedVerticesFixed` confirms that each vertex is displaced along the normalized vector from the arithmetic mean of all polygon vertices to that vertex. The displacement length is the configured pixel amount, independent of the adjacent edge directions. The expanded coordinates are then rounded to 1/16-pixel fixed units and independently clamped to the viewport rectangle.

This is not a constant-distance offset of the polygon boundary: the movement is radial rather than along each vertex's angle bisector / the adjacent edge normals. Consequently the actual increase in coverage perpendicular to an edge depends on that edge's orientation and the vertex's location relative to the centroid. Independent coordinate clamping further changes the displacement direction and can collapse multiple expanded vertices onto the same viewport boundary coordinate. That can leave some edges under-expanded while over-expanding others, especially around corners and clipped boundaries.

The polygon source is already passed through the DS homogeneous clipper, but the accelerated frontend applies a second high-resolution viewport guard. These are distinct clipping domains. The next prototype should therefore be based on an explicit screen-space polygon boundary representation and should preserve the original vertex attributes/interpolation; replacing the current radial vector with a simple larger scalar or merely removing the viewport guard would not be a sound fix.


## Follow-up: triangulation and raster shader boundary semantics

A further pass traced the accelerated scene's polygon triangulation and the Vulkan graphics emission path.

For polygons with more than three vertices, when `BetterPolygons` is enabled, `BuildAcceleratedScene` creates an additional center vertex from averaged screen position and perspective-weighted attributes, then emits a triangle fan from that center to consecutive outer vertices. Coverage expansion changes the outer source vertices, while the generated center remains the average of those expanded coordinates. This means the fan's internal diagonals are also recomputed as the outer contour moves. The fan is a triangulation strategy, not a separate polygon-boundary offset.

The boundary flags on generated triangles identify which triangle sides belong to the polygon perimeter. The Vulkan graphics emission path transfers those flags into triangle flags. In the inspected fragment shader, however, the ordinary color/depth path does not use those boundary flags to expand or repair raster coverage; the shader's visible edge-mark pass is a separate purpose. Therefore the existence of boundary flags alone is not evidence that the renderer performs a coverage correction along the outer perimeter.

The Vulkan emission helper `makeTriangleVertex` applies the configured coverage depth bias to each emitted vertex Z before rasterization whenever the polygon's user coverage fix is active. The same adjusted vertex data is then used to build the emitted triangles. This confirms the bias affects the complete triangulated polygon, including its fan center and all fan triangles, rather than only perimeter fragments.

The graphics emission path also recomputes each emitted triangle's Y bounds from that triangle's actual three screen-space vertices. The earlier scene-level packed polygon Y bounds are not used by this emission helper (`packedYBounds` is explicitly unused there). Thus the lower-screen artifact investigation should distinguish (a) the frontend polygon-wide viewport guard, (b) the final per-triangle bounds/raster dispatch, and (c) Vulkan's triangle edge ownership. A lower-edge crack cannot yet be attributed to the polygon Y bounds alone.

No renderer code was changed in this follow-up. The next source-level target is the triangle raster/dispatch implementation that consumes the emitted triangle flags and Y bounds, including its pixel-center and edge-equation rules. In particular, verify whether shared fan edges and final target scanlines are covered consistently before designing any geometry or depth modification.


## Follow-up: explicit top-left edge ownership metadata

The Vulkan triangle emission helper computes a signed triangle area and classifies each non-boundary side with an `isTopLeftEdge` test. It sets one of three `kTriangleFlagTopLeftEdge*` bits for qualifying internal sides, while explicitly skipping sides marked as polygon boundary edges.

This is an important distinction: the renderer has a custom top-left ownership convention for internal triangle edges, but polygon perimeter edges are deliberately excluded from that ownership-bit assignment. The flags are packed into each emitted `TriangleGpu`. The actual pixel inclusion test that consumes these flags still needs to be traced before deciding whether this is correct or a source of edge cracks.

The current evidence narrows the next investigation to the consumer of `TriangleGpu::flags` and the raster coverage equations. In particular, verify the sign convention, pixel sample location, epsilon use, and treatment of boundary edges at y = targetHeight - 1. Do not alter the ownership flags until their consumer is understood. No renderer code was changed.


## Follow-up: Vulkan raster ownership and lower-edge scissor audit

The actual graphics raster pipeline uses Vulkan triangle-list rasterization with fill mode, no face culling, and single-sample rasterization (`VK_SAMPLE_COUNT_1_BIT`). It does not implement a custom software edge-function pixel test in the inspected graphics path. Pixel coverage of emitted triangles is therefore ultimately governed by Vulkan's fixed-function triangle rasterization rules.

The CPU-side `kTriangleFlagTopLeftEdge0/1/2` metadata is written into the per-triangle flags, but the inspected raster vertex/fragment shader source defines and consumes texture/depth-related flags only; it contains no top-left-edge flag constants or custom edge inclusion/discard test. Consequently the CPU top-left metadata is not currently evidence of a custom shader-side pixel ownership rule. Do not modify those bits as a proposed seam fix without locating another consumer.

The Vulkan viewport is configured with origin (0,0), positive width and height equal to the render target dimensions. The per-draw scissor derives its Y interval from emitted triangle bounds, adds padding of at least two pixels (or the effective raster scale), and clamps the exclusive bottom coordinate to `ColorImageHeight`. This does not reveal an obvious one-row omission at the lower edge: the scissor can include the final row, and the padding is clipped only at the attachment boundary.

The triangle Y bounds are computed as floor(clamped minimum Y) and ceil(clamped maximum Y), with the bottom represented as an exclusive coordinate. This is consistent with a half-open scissor interval in principle. The remaining risk is not a proven off-by-one in this code, but a mismatch between the coordinate convention used when projected vertices are generated and Vulkan's pixel-center/fixed-function coverage convention, or an earlier loss of coverage when the frontend disables the fix for a polygon crossing the viewport.

Updated investigation direction:
1. Trace the exact screen-space coordinate generation and any half-pixel / fixed-point conversion before the Vulkan vertex buffer.
2. Inspect all paths that choose the graphics viewport/scissor, including final and auxiliary passes, to rule out a different lower-edge rectangle.
3. Compare polygons ending exactly at Y = targetHeight, just below it, and just above it using a deterministic off-device coverage test if the scene builder can be isolated.
4. Keep the lower-edge viewport-guard counters as the first on-device correlation test. Do not change Vulkan fill rules, scissor padding, or depth bias speculatively.

No renderer behavior was changed in this follow-up.


## Follow-up: exact viewport-edge coordinate mismatch identified

The frontend viewport guard uses inclusive maxima `MaxFixedX = targetWidth * 16 - 1` and `MaxFixedY = targetHeight * 16 - 1`, then rejects coverage for the entire polygon when a projected vertex is greater than either maximum.

A vertex exactly on the mathematical viewport boundary has coordinate `targetHeight * 16` (Y = targetHeight), which is one fixed-point unit greater than `MaxFixedY`. Such a vertex is a normal possible result of clipping a polygon against the viewport boundary: the polygon can legitimately terminate exactly at the lower edge, while rasterization/scissoring clips fragments to the valid rows [0, targetHeight). The current guard nevertheless classifies that exact-boundary vertex as outside and disables the coverage fix for the entire polygon.

This is a concrete lower-edge false-rejection candidate, not merely a general suspicion about clipping. It is especially relevant because the user's most visible cracks occur at the lower edge. It can also occur at the top/left/right edges with the analogous exact-boundary coordinate.

The safe correction is not to increase the maximum indiscriminately or remove the guard. The guard needs a boundary-aware classification that distinguishes:
- strictly outside coordinates (< 0 or > targetWidth/Height), from
- coordinates exactly on the mathematical viewport boundary (== targetWidth/Height), which are valid clipping boundaries but cannot be expanded beyond the render target.

A polygon touching the boundary should retain coverage eligibility for its interior edges, while expansion of the boundary-facing edge itself must be clipped or selectively suppressed. The current all-or-nothing polygon rejection loses coverage on every edge and is therefore overly conservative for this case.

Before implementing, add bounded diagnostics that separately count exact-boundary contact and strict overflow on each side. Then validate with a synthetic polygon whose lower edge is exactly Y = targetHeight, plus a polygon with a small strict overflow. The first should remain eligible for safe interior-edge coverage; the second must not produce out-of-target geometry. No renderer behavior was changed in this follow-up.


### Exact 6x coordinate values on the user's target

The Vulkan scene builder explicitly sets `UseHiresCoordinates = true`, and `Vertex::HiresPosition` is documented in `GPU3D.h` as having four fractional bits. At 6x, a lower viewport boundary at 192 * 6 = 1152 high-resolution pixels is therefore represented in the frontend's fixed-point units as:

- exact mathematical bottom boundary: `1152 * 16 = 18432`
- current accepted maximum: `192 * 6 * 16 - 1 = 18431`

So the exact-boundary rejection is not an ambiguity about the UI slider or the depth-bias scale. It follows directly from the configured 6x viewport and the four-fractional-bit coordinate representation. A vertex at Y = 18432 triggers `yFixed > MaxFixedY`, even though it lies exactly on the viewport's lower clipping plane.

The same one-unit gap exists on X and at the top/left/right boundaries. This confirms the arithmetic behind the candidate; it still does not establish how often the user's game scene produces such vertices. Instrument exact-boundary contacts separately from strict overflow before changing behavior.


## Follow-up: DS clipping and high-resolution coordinate generation traced

The coordinate source was traced in `src/GPU3D.cpp`:

- `SubmitPolygon()` calls `ClipPolygon<true>()` before the viewport transform.
- `ClipPolygon()` clips in Z, then Y, then X using the homogeneous clip-space planes (`ClipAgainstPlane<2>`, `<1>`, `<0>`). Thus the polygon is clipped before screen-space coordinates are calculated.
- After clipping, the viewport transform computes the ordinary integer screen coordinates in `FinalPosition`.
- The high-resolution path then independently computes `HiresPosition` from the same clipped homogeneous position and viewport registers, multiplying the viewport result by 16 before integer division. This is the source of the documented four fractional bits; it is not a later Vulkan half-pixel offset.
- `HiresPosition[1]` is stored as `posY & 0xFFF`. The 12-bit mask is part of the legacy coordinate representation and must be considered when testing extreme/out-of-range values; it must not be mistaken for an explicit clamp to the valid 0..192 viewport.

This confirms the ordering relevant to the hypothesis: homogeneous clipping occurs first, and the exact screen-space viewport boundary can then be represented by the high-resolution viewport transform. A vertex on the lower viewport plane can therefore legitimately map to the exact bottom coordinate before the accelerated frontend applies its stricter `targetHeight * 16 - 1` maximum. At 6x that exact coordinate becomes 18432 after scaling, versus the current maximum 18431.

Important qualification: this source trace confirms the arithmetic path and makes exact-boundary contact a valid case to test; it does not yet prove that a particular game scene emits such a vertex after the viewport transform, nor that it is the only cause of the user's remaining cracks. The next step remains bounded per-side diagnostics, followed by a deterministic exact-boundary/strict-overflow test. No renderer behavior was changed in this follow-up.
