# Asset processing and validation tools

## Full asset validation

```bash
npm run verify:assets
```

This command only reads files. It checks the syntax of all asset JSON files,
consistency between regular/compressed/master cars and their manifests,
content-based conversion signatures, size limits and wheel patterns.
For active tracks, it validates the model, eight spawn points, start and
checkpoint lines, optional pit configuration and zone map, and the
`collision.bin` format required for multiplayer. Environments must include
an HDR/EXR file. Errors produce a nonzero exit code, so the command can be
used before deployment or in CI with the full asset library available.

Large GLB/BIN/HDR files are excluded from Git. A fresh clone without those
assets is therefore expected to report them as missing.

## Optimizing large player models

```bash
npm run cars:optimize
```

Cars larger than 20 MB in `web/assets/cars/*.glb` are preserved at full
quality in `masters/cars/`, outside the webroot. The original public path
is then replaced with a player model capped at 15 MB. Cars below 20 MB are
not duplicated: their existing GLB serves as both source and player model.

The 15 MB search first tries the original textures and full geometry.
Gentle geometry simplification and then WebP are used only when needed.
The tool chooses the option that sacrifices the least visible detail for
each car. Masters are not public assets and are never downloaded during play.

## Generating multiplayer and ghost models

```bash
npm run cars:compress
```

For each car, the tool uses its `masters/cars/` source when available;
otherwise, it reads `web/assets/cars/*.glb`. It produces a model capped at
5 MB in `web/assets/cars/compressed/`, avoiding another lossy conversion
from a previously optimized 15 MB model. Up-to-date outputs are skipped.

The official `gltfpack` binary is versioned and committed under
`tools/vendor/gltfpack/<version>/`, allowing offline conversion. If the
binary for your platform is missing, it is downloaded on first use and
verified with SHA-256 before installation there. The repository includes
the Windows binary.

Regenerate one or more specific cars:

```bash
npm run cars:compress -- 2008_bmw_sauber_f1.08 --force
```

Try a different target size:

```bash
npm run cars:compress -- --target-mb=4
```

The game uses the model capped at 15 MB for the local car and the compressed
version for multiplayer opponents and Hot Lap ghosts. Both use the same
JSON configuration, preserving `wheelPattern` and wheel animation.

### How settings are selected

Geometry and textures have separate quality scales rather than a combined
profile ladder. Different cars are limited by different data: the BMW 320i
uses 4.17 MB of images and 2.19 MB of geometry, while the Porsche 911 GT1
has the opposite balance at 2.20/5.13 MB. A combined ladder would reduce
quality even on the side that does not need shrinking.

The axes are independent: geometry ratio does not change texture size, and
vice versa. The script measures each axis once, calculates the size of every
combination by addition, and selects the combination with the lowest quality
cost that fits. Each scale's `cost` field expresses the tradeoff: halving a
4096-pixel texture on an opponent 20 meters away is barely visible (1),
while reducing triangle count to one third affects the silhouette (8).

The manifest records the limiting side as `limitedBy` for each car, making
remaining optimization opportunities visible.

The gltfpack call deliberately omits `-sp` (permissive simplification),
which can merge vertices across UV seams and visibly distort textures.
Measurements on four cars showed only a 0.2–1.2% size increase without it,
while still reaching the target ratio. `-vt 16` protects texture coordinates
during quantization for the same reason.

## Splitting models containing multiple cars

> See **[PACKS.md](PACKS.md)** for the full workflow: splitting, identifying
> cars from liveries, wheel patterns for models without useful names,
> deployment and known pitfalls. This section covers only the node-level splitter.

```bash
node tools/split-car-pack.mjs <pack.glb>            # inspect only
node tools/split-car-pack.mjs <pack.glb> --write    # write individual cars
```

Car packs contain a grid of cars arranged side by side. The script finds
gaps between nodes' world-space bounding boxes rather than relying on names,
which are often generic values such as `Object_47`. For a hierarchy with one
root child, as in Sketchfab exports, it descends until the tree branches.

It does not cut geometry. Each car gets a copy where the branching node's
children are restricted to that car, then `gltfpack` rebuilds the buffer and
removes unreferenced data. Selected nodes are not promoted to scene roots:
a glTF node can have at most one parent, and gltfpack rejects invalid hierarchies.

Single-car models were checked to produce exactly one group, with the same
part count, size and material set as the source.

Process the extracted cars with `cars:wheels`, `cars:compress`, then
`tools/audit/wheel-orbit.mjs`. Before splitting a pack, check whether its
liveries share one texture atlas: every extracted car will retain the
entire atlas, since gltfpack cannot partially trim it.

## Generating wheel configurations

```bash
npm run cars:wheels
```

Scans all cars and writes missing `wheelPattern` values. Existing
configurations are preserved because they contain manual corrections,
such as missing tread materials or steering-wheel materials deliberately
excluded from animation. Blind regeneration would discard those fixes.

Force regeneration for one car; existing notes remain in the file for reference:

```bash
npm run cars:wheels -- 2024_ford_mustang_gt3 --force
```

`yawDegrees` defaults to 0, assuming a forward-facing model. If a car faces
backward, adjust its JSON manually; regeneration preserves that value.
The tool lists low-confidence proposals at the end. Inspect those visually
in the car tester.

The offline GLB analyzers run in Node.js without dependencies. They help
find a new car's `wheelPattern` without manually hunting for mesh names
in the browser.

## Inspecting one car

```bash
node tools/propose.mjs 2024_ford_mustang_gt3
```

Prints scored candidate patterns, a proposed `wheelPattern` and measurements:
track width, wheelbase, part count and orbital motion. It also flags patterns
that would match parts located away from the wheels.

Always verify proposals visually in the car tester. The analyzer provides
a starting point, but the rendered result determines whether it is correct.

## Files

- **`glb.mjs`**: a minimal GLB/glTF reader. Extracts JSON and BIN chunks,
  traverses the node hierarchy, and reports each primitive's name chain,
  material name and world-space AABB. It can also be used independently
  for other model analysis.
- **`analyze.mjs`**: an offline replica of `main.js` logic in `setCar`,
  `buildWheelPivots` and `splitMergedWheelMesh`. Its `evaluate()` function
  reports the wheel groups the game would build for a given pattern.
  Results match the game for every manually verified car configuration
  in the repository.
- **`propose.mjs`**: the proposal generator. Tokenizes mesh and material
  names, runs `evaluate` for each candidate, scores symmetry, wheel
  proportions, orbital motion and coverage, then gathers additional
  wheel parts geometrically around the best seed.

## Parts deliberately excluded

These parts are near wheels but do not roll with them. Including them in
a wheel pivot would make them spin with the tire:

- **Suspension** (`susp`, `sospensioni`, `wishbone`, `lengőkar`, `damper`):
  follows steering only.
- **Cross-axles and bracing plates**: sit at wheel height and span the left
  and right wheels, but are flat.
- **Running gear and differentials**: their bounding boxes can accidentally
  resemble wheel proportions, so the analyzer splits them and checks
  whether their components actually coincide with the wheels.

One case required manual handling: the **Brabham BT46B** has inboard brakes
next to the gearbox, so those brake discs must not rotate with the wheels.
