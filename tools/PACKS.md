# Importing car packs

A car pack is a single GLB containing multiple cars arranged side by side.
This guide covers splitting, identification, wheel configuration and
deployment, including pitfalls discovered while processing packs.

The 2010, 2013 and 2014 F1 packs were imported through this workflow,
adding 34 cars in total.

## 0. Identify the pack structure

Start by inspecting the pack:

```bash
node tools/split-car-pack.mjs <pack.glb>
```

**A) Separate nodes for each car.** The output contains one group per car,
often with useful names such as `ferrari_2014_1`. Use `split-car-pack.mjs`.

**B) One node with meshes spanning every car.** The output contains one
suspiciously wide group: the 2010 pack was 117 units wide. Its hierarchy
looks like this:

```text
GLTF_SceneRootNode
  bodywithwheels_0        <- a single node
    Object_4 [mesh]       <- each mesh spans the entire field
    Object_5 [mesh]
    ...
```

Regrouping nodes cannot separate cars when one mesh contains geometry
from several cars. Use `tools/slice-car-pack.mjs` to split at triangle level.

A quick distinction: if some primitives span the full pack width, use case B.

## 1. Split the pack

### Case A: node-level splitting

```bash
node tools/split-car-pack.mjs <pack.glb> --write
```

Creates a copy for each car, restricting the branching node's children to
that car. `gltfpack` then rebuilds the buffer; no geometry is cut.

### Case B: triangle-level splitting

```bash
node tools/slice-car-pack.mjs <pack.glb> --write
```

Assigns each triangle to a car by its centroid and builds a new buffer per
car. This requires indexed triangles with `POSITION`/`NORMAL`/`TEXCOORD`
attributes, without compression or a skeleton. All three packs processed
so far met those requirements. Inspect them with:

```bash
node --input-type=module -e "
import { loadGlb } from './tools/glb.mjs';
const { json: g } = loadGlb('<pack.glb>');
console.log(g.extensionsUsed, [...new Set(g.meshes.flatMap(m => m.primitives.flatMap(p => Object.keys(p.attributes))))]);
"
```

## 2. Identify the cars

Pack names such as `Body27Mtl`, `Object_4` and `MIRRORS.013_18` often reveal
nothing. Livery textures contain driver names and racing numbers, which
can identify the season and team.

In these packs, each car has exactly one unique body material; the other
materials are shared across the field. Export the unique texture for each
car and inspect it directly. A custom 3D viewer is unnecessary.

**Do not identify cars by color alone.** In the 2010 pack, `car_0` had a
livery that was 71% red and appeared to be a Ferrari. It was actually the
**Virgin VR-01**, identified by the numbers: #24 Glock and #25 di Grassi.

## 3. Find wheel patterns

The name-based `tools/propose.mjs` cannot find useful patterns in these
packs because the materials have generic names. Use geometry instead:

```bash
node tools/audit/wheels-by-geometry.mjs <car.glb-or-directory>
```

A wheel material should form four corner groups with equal diameters,
round shapes in the plane of rotation and centers concentric with the tires.
The pattern lists exact material names anchored at the name boundary
(`name(?![^\s])`). Without this boundary, `rim` would also match `blur_rim`.

## 4. Import and deploy

```bash
# 1. Copy the extracted GLB to its final name.
cp <sliced>.glb web/assets/cars/2013_ferrari_f138.glb

# 2. Create the wheel configuration using the geometry-based pattern above.
# 3. Generate the compressed multiplayer model.
npm run cars:compress -- 2013_ferrari_f138

# 4. Validate the result.
node tools/audit/wheel-orbit.mjs --scan
node tools/audit/missing-wheel-parts.mjs
```

**Upload GLBs separately** when deploying: they are excluded from Git.
The commit transfers only the JSON configuration and `compressed/manifest.json`.

```bash
scp web/assets/cars/<id>.glb apex-overdrive-server:/opt/racing/web/assets/cars/
scp web/assets/cars/compressed/<id>.glb apex-overdrive-server:/opt/racing/web/assets/cars/compressed/
```

`apex-overdrive-server` is your own local SSH alias, not a real server name.
See the [deployment guide](../DEPLOY.md#ssh-connection) to configure it.

Check the public HTTP response as well as the file on disk:

```bash
curl -s -o /dev/null -w "%{http_code}\n" https://racing.levente.net/assets/cars/<id>.glb
```

### Name collisions

If a car ID already exists, give the old model the `_2` suffix and use the
plain name for the newly imported car. Rename on the server with `mv`
instead of uploading the file again, and update the keys in
`compressed/manifest.json`.

Lap-time records reference car IDs. After a rename, existing records under
the plain ID refer to the new model.

## Known pitfalls

**`-kn -km` prevents cleanup.** These flags retain named nodes and materials.
Everything in a pack has a name, so gltfpack retained unused content and all
11 outputs were 45.7 MB. Wheel detection needs names, so remove names from
the other cars rather than discarding all names.

**gltfpack retains nodes unreachable from the scene.** Their mesh references
must also be removed.

**Unreferenced images are retained too.** In the 2014 pack, all 102 images
(23.6 MB) remained even after unused materials were removed. Filter materials,
textures and images explicitly, rebuilding their indices.

**Triangle-level splitting must bake transforms.** Exporting local vertex
positions lost the Sketchfab rotation and left cars on their sides:
8.7 × **23.9** × 5.2 instead of 8.7 × 5.2 × 23.9. Apply the full transform
to positions and its rotation component to normals.

**Gaps are unreliable separators.** Cars were nearly touching: 1.8-meter-wide
cars spaced 1.96 meters apart. A gap threshold merged them. Test overlap
instead: components belonging to one car overlap, while separate cars do not.

**Wheel diameter needs a relative minimum.** Rims are naturally smaller than
tires: 0.31 versus 0.56 on the F14 T. An absolute threshold excluded inner
wheel parts, leaving rims stationary. Set limits relative to the tire.

**Measure concentricity in the plane perpendicular to the axle.** Wheels
rotate around the X axis, so offsets along it are irrelevant. Inner and
outer wheel faces can be 0.13 units apart along X, and a 3D-distance test
would incorrectly exclude them. The same error also affected orbital-motion
measurements.

## Related tools

- [tools/README.md](README.md): everyday tools such as `cars:wheels`
  and `cars:compress`.
- `tools/audit/`: `wheel-orbit.mjs` checks orbital motion,
  `missing-wheel-parts.mjs` finds omitted wheel components;
  also see `fix-wheel-orbit.mjs`, `caliper-exact.mjs` and `drop-nonround.mjs`.
