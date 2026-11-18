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
