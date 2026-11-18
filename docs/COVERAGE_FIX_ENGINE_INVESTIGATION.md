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
