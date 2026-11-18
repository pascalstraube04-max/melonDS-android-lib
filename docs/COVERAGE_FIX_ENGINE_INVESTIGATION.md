# Coverage Fix / Vulkan Engine Investigation

Branch: `coverage-fix-engine-investigation`

## Scope

This branch is an isolated investigation branch. It now contains a candidate convex-polygon coverage-expansion prototype and an exact-viewport-boundary guard adjustment. These changes are uncompiled and unvalidated; they are not yet a confirmed fix.

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


## Candidate change: allow the exact mathematical right/bottom boundary

Commit `cb19a69d2aec49cc91e0d4e4aa6f7672750cbcac` changes the accelerated frontend guard to compare against `MaxFixedX + 1` and `MaxFixedY + 1` using 64-bit temporaries. This admits the exact mathematical right/bottom viewport plane while continuing to reject coordinates strictly beyond it. The coverage-expansion helper still clamps generated coordinates to the last addressable fixed-point coordinate (`MaxFixed*`), so this change does not intentionally enlarge the render target.

The source trace supports the four-fractional-bit interpretation: `SubmitPolygon()` performs homogeneous clipping before calculating `HiresPosition`, and the viewport transform multiplies by 16 before integer division. At 6x, the exact lower plane is 18432 while the last addressable coordinate is 18431.

Review caveats before treating this as validated:
- The guard currently uses the `+1` allowance for both coordinate modes. The Vulkan scene builder currently selects `UseHiresCoordinates = true`; the non-hires branch should still be checked against its integer-pixel convention before this code is reused elsewhere.
- The guard is still all-or-nothing for strict overflow. A polygon with one genuinely out-of-range vertex still loses coverage-fix eligibility as before.
- The candidate has not been compiled or tested on a device. It is not yet established that the game emits exact-boundary vertices frequently, or that this change removes the visible lower-screen cracks.
- No evidence yet that the patch changes depth ordering or flicker; depth bias behavior remains a separate issue and must be tested with snow and building shadows.

Build verification status: the repository's Ubuntu workflow is present on the default `DevelopSub` branch and its push filter includes `ci/*`. A dedicated `ci/coverage-boundary-check` branch and a workflow-file commit were created, but the Actions API currently reports zero workflow runs and zero status checks. Therefore there is no successful build result to report. The workflow has `continue-on-error: true`, so any eventual run must be checked at the individual job/log level rather than relying only on an overall workflow conclusion.


## Prototype: edge-normal offset for convex polygons

Commit `77378f2de915b4ccdb53c750857bff4dde7b71cf` replaces the radial-only expansion for convex polygons with an edge-normal offset: adjacent outward edge normals are combined at each corner and intersected as a miter. The miter length is capped at four times the requested expansion to avoid unbounded spikes near acute/degenerate corners. A subsequent correction in `31d83719a62d03885fa4464709ea738f7adbf4a1` preserves the previous radial behavior as a fallback for non-convex or near-zero-area polygons, rather than leaving zero-filled coordinates.

This is a geometry prototype, not a validated fix. Static review points:
- The method assumes the polygon boundary is consistently ordered; winding is derived from signed area.
- Convexity is checked before applying the miter offset. Non-convex inputs use the prior radial method.
- Consecutive duplicate/near-zero edges keep the affected source vertex rather than normalizing a zero-length edge.
- The final expanded coordinates are still clamped to the render target, so viewport-edge corners can have a shorter effective offset.
- The current all-or-nothing strict-overflow guard remains unchanged; the exact-boundary allowance is still only the earlier candidate.
- No unit-test harness for this frontend helper was found in the repository tree. No compiler, CI, or device test has been run; no build was triggered.

Required next checks before treating this as an engine-complete change:
1. Compile the engine after explicit user build authorization.
2. Add or run deterministic geometry checks for clockwise/counter-clockwise triangles and quads, near-collinear corners, duplicate vertices, and non-convex fallback.
3. Verify attribute interpolation and fan triangulation remain visually stable after moving only outer vertices.
4. Test lower-screen clipping, snow, building shadows, repeated/clamped textures, and long-session 6x performance on the S24 Ultra.
5. Revisit depth bias separately; this prototype does not change the polygon-wide Z shift.


The convex-offset prototype was subsequently hardened on the same investigation branch:
- Missing source vertices now preserve the available source coordinates rather than collapsing those vertices to the zero-initialized output.
- Near-zero-area and non-convex polygons fall back to the prior radial expansion.
- Edge normals are precomputed once per polygon edge and shared by adjacent vertices. The ordinary corner intersection uses an algebraic miter expression; an extra square root is needed only when enforcing the acute-corner miter cap. This keeps the normal-construction cost near one normalization per vertex rather than recalculating both adjacent edges at every corner.
- The Vulkan scene builder passes the user expansion value directly as target-screen pixels and clamps it to 0..2.0. At 6x this remains a high-resolution-pixel distance; the code does not multiply the expansion a second time by the 6x scale.


### Deterministic geometry review vectors (pre-build)

These are hand-checkable expectations for the new convex offset, not substitutes for compiled tests:

- CCW square `(0,0), (10,0), (10,10), (0,10)` with distance 2 should produce `(-2,-2), (12,-2), (12,12), (-2,12)` before viewport clamping.
- The same square with reversed winding should produce the same expanded outline in the corresponding reversed vertex order.
- A triangle should offset each straight side by the requested perpendicular distance; corner miters follow the adjacent-normal intersection, subject to the 4x cap.
- A concave or near-zero-area polygon should take the radial fallback, not the convex miter path.
- Duplicate consecutive vertices should not generate a division by zero; the entire polygon takes the radial fallback so the outline does not mix miter-expanded and unexpanded corners.
- At 6x, exact bottom-boundary Y=18432 is admitted by the guard and expansion output clamps to 18431; Y=18433 remains strict overflow and disables the fix for that polygon.

These expectations have not been run against compiled C++ code.


A further static-review correction (dc0620915686d32e3d987ed9e5e5ce58eeb0f4b7) makes degenerate-edge handling consistent: if any polygon edge has effectively zero length, or adjacent outward normals have no stable miter intersection, the entire polygon uses the radial fallback. This avoids mixing unexpanded source vertices with miter-expanded neighbors along one outline. This remains uncompiled and untested.


### Follow-up source audit: fan center and depth scope

The accelerated scene builder computes the optional polygon-fan center after `resolveFixedCoords()` has applied the expanded outer coordinates. Thus the fan center follows the expanded contour rather than remaining at the original center; this is internally consistent for the position geometry. Its color, texture coordinates, W and Z are still reconstructed from the original polygon attributes using the existing perspective-weighted averaging. The same `makeTriangleVertex()` path applies the configured user depth bias to every emitted vertex for that polygon, including the fan center and boundary-line quads. No selective edge-only depth correction exists in this path. Changing that now would require a separate depth model and risks altering DS depth ordering, so the bias remains untouched pending a controlled test.


### Configuration-scope audit

The Vulkan path under investigation sets `UseHiresCoordinates = true`, so the coverage helper's high-resolution coordinate resolution matches the scene builder's normal vertex path. The legacy/non-hires branch uses a different coordinate conversion in `resolveFixedCoords()`; coverage-enabled builds with `UseHiresCoordinates = false` and `Scale > 1` need a separate consistency check before this helper is generalized to that configuration. This does not affect the current Vulkan configuration, but it remains an explicit scope limitation. The project is C++17 and this translation unit includes `<cmath>` for the geometry math.


### Follow-up: convex-offset numerical and interpolation review

A line-by-line review of the current convex offset helper confirms the corner construction is mathematically consistent for a consistently wound convex polygon:

- The signed area selects the winding.
- Each edge gets one outward unit normal.
- At each vertex, the adjacent normals are intersected using the miter factor `distance / (1 + dot(n0,n1))`.
- For very acute corners the miter is capped at four times the requested distance; nearly opposite normals or zero-length edges send the whole polygon through the radial fallback.
- The convexity test rejects a turn opposite to the polygon winding, while collinear turns remain eligible.

The cap is intentionally conservative: at sufficiently acute corners the resulting outline is no longer a mathematically exact constant-distance offset, but it avoids extreme spikes. This may leave a small residual corner gap in pathological geometry; raising the cap globally would trade that for larger overdraw and unstable geometry, so it should only be changed in response to a reproducible test case.

The output is rounded to 1/16-pixel fixed units and clamped independently to the last addressable viewport coordinate. Consequently, edges that reach the viewport boundary cannot retain the full requested offset in the outward direction. This is expected target clipping, not a guarantee of full-distance expansion at the attachment edge.

The fan center is recomputed from the expanded outer positions, but its perspective-weighted color, texture coordinates, W and Z are derived from the original source vertices. That preserves the existing attribute construction and keeps the center inside the expanded contour, but it does slightly change the geometric interpolation domain. Textured polygons with strong perspective gradients therefore need explicit visual regression testing; source-level review alone cannot establish that the texture remains visually unchanged.

No further renderer change is justified by this review. In particular, do not raise the miter cap or alter depth bias speculatively. The required deterministic cases remain: both windings, right/obtuse/acute corners, nearly collinear edges, duplicate vertices, concave fallback, exact viewport-boundary contact, and perspective-textured quads. All findings remain static-only; no compile, automated test, Android integration, or device validation has been performed.


### Follow-up: source clipper limits and pathological-W caveat

The DS homogeneous clipper was re-opened at the implementation level rather than relying only on the call-site trace:

- `ClipAgainstPlane` uses a fixed `Vertex temp[10]` buffer, matching the `Polygon::Vertices[10]` storage inspected in `GPU3D.h`.
- The clip sequence is Z, then Y, then X. Intersections are generated into the polygon vertex sequence before the viewport transform.
- The clipper source itself retains explicit TODOs about hardware differences for vertices with W=0 and unusual Y results. This means ordinary convex clipping behavior is well understood, but pathological zero-W cases are not proven equivalent to physical DS hardware.
- The accelerated frontend rejects degenerate polygons and polygons with fewer than three vertices before the coverage helper runs; the helper also bounds its local arrays to ten entries.

Engineering consequence: the convex miter path is a reasonable candidate for normal clipped convex polygons, but the clipper's documented W=0 uncertainty prevents claiming that every possible post-clip polygon is a well-formed convex contour. The current convexity / area / zero-edge checks provide fallback protection for several malformed shapes, but they are not a substitute for a deterministic test of zero-W and near-zero-W clipping.

No code change was made: changing the DS clipper or its fixed-size buffering without reproducing a failure would broaden risk beyond the Vulkan coverage issue. Add W=0 and near-zero-W cases to the eventual geometry regression set. Static review only; no build, CI, or device test was run.


### Vulkan raster and depth-pipeline audit

Static inspection of the Vulkan graphics path confirms the following data flow:

- The CPU scene builder converts the configured coverage depth-bias value with a factor of 16,777,215. The graphics triangle builder subtracts that value from each affected vertex Z, clamps at zero, and only does so when the polygon's user coverage fix is active. The adjustment is therefore applied before interpolation, not as a Vulkan fixed-function depth-bias state.
- The raster vertex shader normalizes the incoming 24-bit-style Z value with 1/16,777,216, applies the DS depth-equal window offset where required, and places the normalized depth in clip-space Z multiplied by clip W. The fragment shader then selects linear or perspective depth according to its specialization and writes that value to `gl_FragDepth` in the fragment-depth variant.
- Opaque pipeline creation selects `LESS` or `LESS_OR_EQUAL` according to the emulated depth-compare mode. Depth testing and writes are enabled on the relevant opaque pipelines; other specialized passes deliberately differ.
- The selected depth/stencil attachment format is device-dependent: the renderer tries D24_UNORM_S8 first, then D32_SFLOAT_S8, then D16_UNORM_S8. The same nominal 24-bit DS depth therefore encounters different attachment quantization on devices that fall back to D32 or D16.
- The graphics vertex structure is statically asserted to 64 bytes, and its C++ offsets match the Vulkan vertex attribute descriptions (position/W, UV, packed color, flags/metadata, and signed texture-coordinate bounds).

Engineering implications:

1. The user's depth-bias value changes the interpolated depth of the whole polygon, not just the expanded edge. This explains why increasing it can change overlap and depth-test outcomes beyond the visible seam; it must not be increased blindly.
2. The normalized shader divisor is 2^24 while the D24 attachment has 2^24-1 representable intervals. The CPU bias conversion uses the latter scale. This is a small, deliberate-or-incidental scale difference worth retaining in the precision review; it is not by itself proof of a defect.
3. Because the attachment format is selected per device, depth-quantization behavior must be recorded on the Galaxy S24 Ultra during later device validation. Static source review cannot establish which candidate format that phone actually selects.
4. The shader path has both linear and perspective depth interpolation variants. Their selection and the W-buffer/Z-buffer combinations must be covered by regression tests, especially at overlapping coplanar edges and with the user's snow/shadow flicker scenes.

No Vulkan source change is justified by this audit alone. The findings are static-only: no shader compilation, C++ build, automated test, APK, or device validation was performed.

### Corrective source change: complete depth-image usage validation

Commit `762577831d92ac01ba4f8ab7ff5e9853cf7f2e0e` updates `selectGraphicsDepthStencilFormat()` to retain the existing D24, D32, D16 preference order but reject a candidate unless the physical device accepts the exact image configuration used by the renderer: 2D image, optimal tiling, and the combined depth/stencil-attachment, sampled-image, and transfer-source usage flags. The previous check established only depth/stencil attachment support and did not validate the combined image usage.

This is a narrowly scoped capability-validation correction. It adds no render pass, GPU pass, allocation, per-frame work, or synchronization and therefore has no expected frame-time cost. It may select a later supported depth format on a device where the first candidate cannot satisfy the full usage combination.

Validation status: source change committed; not compiled, not run on Android, and not tested with the S24 Ultra / Mr Purple T30. Runtime format selection and image creation still require device validation. No APK or CI build was started.
