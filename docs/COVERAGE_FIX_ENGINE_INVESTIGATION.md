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
