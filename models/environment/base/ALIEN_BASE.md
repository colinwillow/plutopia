# Alien Base — three.js scene

The player's home base: a domed circular atrium with a mezzanine, three corridors, and a domed water-garden wing. Walkable. Built in Blender, exported as a single GLB with no textures — all materials are flat colours plus emission, so the file is tiny and everything is tuned in code.

## Asset

| File | Size | What it is |
|---|---|---|
| `alien_base.glb` | 0.82 MB | Whole environment: 292 nodes, 42 meshes, 45 materials, 1 camera. Draco-compressed geometry, zero image textures |

Copy into `public/base/`. Draco decoder required: copy `node_modules/three/examples/jsm/libs/draco/gltf/` into `public/draco/` and register it with `GLTFLoader.setDRACOLoader()`.

No lightmaps, no HDR probe, no texture folder. That is intentional — see Lighting below.

## Layout and coordinates

Units are metres. Blender is Z-up, glTF/three.js is Y-up, so a Blender point `(x, y, z)` becomes three.js `(x, z, -y)`. All figures below are already in three.js terms.

- **Atrium**: circular, radius 11.5, centred on the origin. Floor at `y = 0`. Glass dome from `y = 6.6` to `y = 10.2`, with a lit oculus ring at the top.
- **Central platform**: radius 4.4, top surface at `y = 0.75`. Four ramps up at roughly 45°, 135°, 225°, 315°. Four consoles face outward around a hologram column.
- **Mezzanine ring**: walkable surface at `y = 4.0`, inner edge at radius 8.9, outer at 11.5. Reached by two curved wall ramps: one climbing anticlockwise between 60° and 130°, one between 240° and 180°.
- **Three corridor arches** at 30°, 150°, 270°, each 2.4 m tall at the crown, opening into a 12.5 m ribbed corridor.
- **Water garden**: at the end of the 150° corridor, centred near `(-27.7, 0, -16)` in three.js terms, radius 8.5, own glass dome. Sunken pond at `y = -0.6`, central rock island at `y = 0.55` with the giant mushroom tree.
- **Hatch door** with the glowing planet emblem on the atrium's back wall, at Blender `+Y`.

Player spawn suggestion: just inside the 270° arch, three.js `(0, 0, 10.4)`, facing the origin.

## Instancing — do not break it

264 of the 292 nodes share 13 meshes. The biggest wins: 100 railing segments share one mesh, 40 planters share one, 29 orb posts, 23 small tanks, 21 small mushrooms.

- `GLTFLoader` gives these as separate `Mesh` objects that share one `BufferGeometry`. That already saves memory.
- For draw calls, convert each reused geometry to a single `InstancedMesh`: group `scene` children by `geometry.uuid`, and where the count is above roughly 8, build an `InstancedMesh` from that geometry and material, write each original's `matrixWorld` into an instance matrix, then remove the originals. That takes about 290 draw calls down to about 40.
- Do this after load, before the first frame. Keep a map from instance index back to the original node name if anything needs to be interactive later.

## Lighting — real-time, not baked

Nothing is baked, and no lights were exported. This style is driven by flat bright surfaces and glowing trim, not by subtle bounce light, so it is cheaper and better-looking to light in code. Recipe:

- `HemisphereLight`, sky a pale blue-violet, ground a warm cream, intensity around 1.5. This does most of the work.
- One `DirectionalLight` from above, cool white, intensity around 1.5, positioned over the oculus at three.js `(0, 12, 0)` aimed at the origin. Enable shadows on this one only, with a tight shadow camera around the atrium.
- `PointLight`s, cool cyan, at the oculus `(0, 9, 0)`, the hologram `(0, 4.3, 0)`, and the garden oculus. Intensity around 30 to 60 with `distance` set so they fall off inside their room.
- A warm `PointLight` ring: four to six lights at radius 7, `y = 3.3`, colour around `0xffd9a0`, low intensity, to mimic bounce off the cream floor.
- **Bloom is what sells it.** `UnrealBloomPass` or the newer `SelectiveBloom`, threshold around 0.8, strength 0.4 to 0.6, radius 0.4. All the cyan slots, orb posts, screens, and the hologram are emissive and will halo correctly.
- Tone mapping: `ACESFilmicToneMapping` or `AgXToneMapping`, exposure around 1.0, `outputColorSpace = SRGBColorSpace`.

Emission strengths in the GLB are deliberately modest (1.4 to 2.5 via `KHR_materials_emissive_strength`) on the assumption bloom adds the glow. If it looks dull without bloom, raise `material.emissiveIntensity` rather than editing the GLB.

## Materials worth knowing about

The GLB uses `KHR_materials_transmission`, `clearcoat`, `sheen`, and `ior`. three.js maps these to `MeshPhysicalMaterial`, which is correct but expensive — transmission in particular forces extra render passes.

- `Glass_Clear`, `Fluid_Cyan`, `Water` are the transmissive ones, on the specimen tanks, egg domes, and pond.
- On mobile or if the frame rate suffers, swap those to `MeshStandardMaterial` with `transparent = true` and an opacity around 0.15 to 0.35. Visually almost identical at this art style, far cheaper.
- Set `depthWrite = false` on the transparent ones if sorting artefacts show up through stacked tank glass.

## Collision

No collision mesh is exported. Build it from these primitives rather than the visual geometry:

- Atrium floor: disc, radius 11.5 at `y = 0`
- Platform: disc, radius 4.4 at `y = 0.75`, plus four ramp wedges
- Mezzanine: annulus, inner 8.9 and outer 11.5 at `y = 4.0`, plus the two curved wall ramps
- Perimeter wall: cylinder shell at radius 11.5, with three gaps at 30°, 150°, 270°
- Corridors: boxes 3.4 wide running out from each arch
- Garden: disc radius 8.5, pond depression at `y = -0.6`, island disc radius 2.6 at `y = 0.55`
- Railings: low walls along the platform rim, mezzanine edge, ramps, and garden walkway

The railings are visual only and about 1.05 m tall, so the player will walk through them without collision volumes.

## Interactive candidates

Nothing is rigged yet, but these are separate nodes with sensible pivots: the four platform consoles, the eight wall consoles, the specimen tanks, the six egg carts, and the hatch doors. The hatch door mesh has its emblem as part of the same mesh, so opening it means rotating or sliding the whole node.

## Re-exporting (Colin's machine)

`C:\Users\colin\Desktop\3D STUFF\5_ENVIRONMENTS\alien_base\`

- `alien_base.blend` — the editable master. 1,005 objects, kit pieces live in a hidden `Kit` collection off at `x = 60`; every placed copy is a linked duplicate sharing the kit mesh.
- `alien_base_export.blend` — the flattened export copy. Modifiers baked, curves converted, one-off geometry merged by material down to 27 meshes, instancing preserved. The GLB came from this file.
- `FOR_REPO/base/alien_base.glb` — the deliverable. Drag the `base` folder into the repo's `public`.
- `base_helpers.py` in the master's Text Editor holds the geometry builders, including `instance()`, which is what preserves mesh sharing.

To change the design, edit the master, then redo the flatten steps in a fresh export copy. Editing a kit object updates every copy of it.

## Not built yet

- Side rooms behind the twelve mezzanine window ports — currently just glowing discs
- Stairs between the mezzanine and any upper level; the references hint at a third tier
- Signage with readable text
- Anything animated: the hologram planet spinning, water movement, tank bubbles rising
