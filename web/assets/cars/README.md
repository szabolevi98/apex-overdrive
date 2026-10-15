# Car models (`web/assets/cars/`)

The Node.js server automatically includes `.glb` files in this directory's
root in the asset manifest and in-game car selector. No car list needs to
be maintained manually.

## Files

A car can have the following files sharing the same ID:

```text
web/assets/cars/<id>.glb             local/player model
web/assets/cars/<id>.json            optional configuration
web/assets/cars/compressed/<id>.glb  multiplayer and Hot Lap model
masters/cars/<id>.glb                original master, when available
```

- Player models have a target size of at most 15 MB.
- `compressed/` models have a target size of at most 5 MB and are downloaded
  for opponents and Hot Lap ghosts.
- `masters/cars/` is outside the webroot and cannot be downloaded during
  play. When a master exists, compression starts from it rather than
  applying another lossy conversion to an already optimized model.
- Do not edit generated manifests manually; model-processing commands keep
  them up to date.

See the [model-processing tools](../../../tools/README.md) for current
commands to optimize player models, generate compressed versions and
create wheel configurations.

## Car configuration

An optional `<id>.json` sits beside the model with the same base filename:

```json
{
  "yawDegrees": 0,
  "wheelPattern": "tyre|rim|disc|caliper"
}
```

- `yawDegrees`: additional rotation around the Y axis, in degrees.
  Usually `0`; a backward-facing model typically needs `180`.
- `wheelPattern`: a case-insensitive regular expression matched against
  mesh, parent-node and material names. Matches are used to build the
  visual wheel groups for rolling and steering.
- Fields beginning with an underscore, such as `_wheelPattern` and
  `_yawDegrees`, are comments only. The server does not send them to clients.

Both functional fields are optional. A car without `wheelPattern` remains
usable, but its wheels do not visibly roll or steer. An invalid regular
expression produces a console warning, and loading continues without
wheel animation.

## Processing a new car

1. Copy the source `.glb` into this directory's root.
2. Optimize the player model: `npm run cars:optimize`.
3. Generate or verify wheel configuration: `npm run cars:wheels`.
4. Generate the multiplayer/ghost model: `npm run cars:compress`.
5. Check orientation and wheel animation locally in the developer wheel tester.

Inspect wheel candidates for one car:

```bash
node tools/propose.mjs <id>
```

Batch wheel processing preserves manually corrected configurations.
Before forcing regeneration, review the existing `_wheelPattern` note,
which may document model-specific exceptions.

## How wheel detection works

The game groups `wheelPattern` matches into four corners by position.
Merged wheel geometry is split when necessary, then the visible wheel
centers are used to center the entire model along X/Z. A pattern should
therefore include only parts that actually roll together. Exclude
suspension, wishbones and other nearby running-gear components.

Always finish with a visual check. Automatic analysis provides a useful
proposal, but unusually exported models may need manual corrections.
