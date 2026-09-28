# VoxelCdx project format (`.vcdx`)

This document describes the project file written by [VoxelCdx](https://voxelcdx.pages.dev), format **version 10** (VoxelCdx 1.1.x). It is meant for tools that want to read or write VoxelCdx projects. Feel free to use it in any project, open-source or commercial.

A `.vcdx` file holds one or more **models**. Each model has **layers** of voxels and, optionally, a **rig** (bones, per-voxel bone painting, animation clips and weapon attachments). All models share one **palette** of 1024 colours with PBR-like materials.

## Conventions

| Type | Encoding |
|---|---|
| `u8` | 1 byte |
| `bool` | 1 byte, `0` = false, `1` = true |
| `u16` | 2 bytes, little-endian |
| `i32` | 4 bytes, little-endian, two's complement |
| `i64` | 8 bytes, little-endian |
| `f32` | 4 bytes, IEEE 754 single, little-endian |
| `vec3` | `f32 x, f32 y, f32 z` |
| `string` | length in **bytes** as a 7-bit encoded unsigned integer, then that many UTF-8 bytes (the .NET `BinaryWriter.Write(string)` format, see below) |

**7-bit encoded length:** the length is written in groups of 7 bits, least significant group first. Every byte except the last has its high bit (`0x80`) set. `5` → `05`, `300` → `AC 02`. An empty string is a single `00` byte.

**Coordinates:** right-handed, **Y is up**, X points right, Z points towards the viewer (the same as glTF / OpenGL). Voxel `(x, y, z)` fills the unit cube from `(x, y, z)` to `(x + 1, y + 1, z + 1)`. Coordinates can be negative.

Conversion to and from MagicaVoxel (`.vox`, Z up):

```
vox  = (x, -z - 1, y)
cell = (vox.x, vox.z, -vox.y - 1)
```

**Palette indices:** voxels store a palette index from `1` to `1023`. Index `0` means "no voxel". When VoxelCdx imports a `.vox`, MagicaVoxel colour index *i* becomes palette index *i*.

## File layout

| Field | Type | Notes |
|---|---|---|
| magic | 4 bytes | ASCII `VCDX` |
| version | `i32` | `10` today. A reader should refuse versions newer than it knows. |
| info | `string` | JSON summary, see below. Present from version 4. |
| thumbnail length | `i32` | Present from version 4. May be `0`. |
| thumbnail | bytes | PNG, 192 × 192, preview of the active model. |
| body | rest of file | **Brotli** compressed ([RFC 7932](https://www.rfc-editor.org/rfc/rfc7932)). Decompress it and read the body below from the result. |

The header (magic, version, info, thumbnail) is **not** compressed, so file browsers can show the preview and names without decompressing anything.

### Info JSON

Informational only, readers can ignore it. Writers should fill it in:

```json
{ "models": 2, "voxels": 1308, "size": [64, 64, 64], "names": ["Hero", "Hero2"] }
```

`voxels` is the total voxel count of all layers of all models. `size` is the canvas size of the active model, or the size of its bounding box if it has no canvas.

## Body

```
i32      paletteSize            // 1024
Palette  palette                // the active palette
string   renderSettings         // JSON with lighting and camera settings, see below
i32      pageCount
i32      activePage
Page[pageCount]
i32      modelCount
i32      activeModel
Model[modelCount]
```

`renderSettings` is VoxelCdx-specific. Readers can skip it. Writers can store `{}`, and VoxelCdx then uses its defaults.

### Palette

```
Slot[paletteSize]
Row[paletteSize / 16]           // 64 rows of 16 colours
```

**Slot:**

| Field | Type | Notes |
|---|---|---|
| r, g, b, a | 4 × `u8` | sRGB colour and alpha |
| material | `u8` | `0` diffuse, `1` metal, `2` emissive, `3` glass |
| roughness | `f32` | 0 – 1 |
| metallic | `f32` | 0 – 1 |
| emission | `f32` | strength, 0 = none |
| ior | `f32` | index of refraction (glass) |
| transparency | `f32` | 0 – 1 |

Slot `0` is written but ignored.

**Row:** `string label`, `bool groupStart`. The label is the row name shown in the palette panel (for example "Reds"). `groupStart` marks a row that starts a new visual group. Writers can use `""` and `false`.

### Page

Extra palettes the user keeps next to the active one (the palette "pages" in the UI):

```
string   name
Palette  palette
```

Writers can store `pageCount = 0` and `activePage = 0`.

### Model

```
string   name
bool     hasCanvas
  (if hasCanvas) i32 minX, minY, minZ, i32 sizeX, sizeY, sizeZ
i32      layerCount
i32      activeLayer
Layer[layerCount]
Rig      rig                    // version 5+
```

The **canvas** is the editing box shown in the editor. Voxels may lie outside it. `hasCanvas = false` means an unbounded model.

### Layer

```
string   name
bool     visible
bool     locked
i32      chunkCount
Chunk[chunkCount]
```

### Chunk

Voxels are stored in dense chunks of 32 × 32 × 32:

```
i32      cx, cy, cz             // chunk coordinate
u16[32768] voxels               // palette index per voxel, 0 = empty
```

The voxel at chunk-local `(lx, ly, lz)` (each `0..31`) has world position `(cx * 32 + lx, cy * 32 + ly, cz * 32 + lz)` and is stored at index `lx | (ly << 5) | (lz << 10)`, so X changes fastest. VoxelCdx writes chunks sorted by z, then y, then x, and only writes chunks that contain at least one voxel. Readers must not depend on either.

Values of `1024` or more are invalid, and VoxelCdx reads them as `1023`.

## Rig (version 5+)

Every model has a rig block, even if it has no bones. Fields marked *vN* only exist from that version on.

```
i32      boneCount
Bone[boneCount]
LayerBinding[layerCount]        // one per layer, same order as the layers
i32      clipCount
Clip[clipCount]
// v7+
u8       boneShape              // 0 octahedral, 1 stick (display only)
string   armatureName           // name of the armature root node in FBX / glTF exports
BoneColor[boneCount]
bool     isAttachment[layerCount]
// v8+
Attachment[layerCount]
// v9+
bool     isFavorite             // model is pinned in the model list
string   animationSource        // "" or the name of the model this one takes its animations from
ClipLink[clipCount]
```

### Bone

```
string   name                   // unique within the model
string   parent                 // "" for a root bone
vec3     head                   // joint position, voxel units, model space
vec3     tail                   // end of the bone
i32      id                     // v6+, unique and > 0, referenced by bone painting
```

Bones are listed so that the order is stable, but a parent may appear **after** its child. Resolve parents by name.

### LayerBinding

Says which bone moves each voxel of the layer:

```
string   boneName               // "" = the layer is not bound
i32      chunkCount             // v6+
Chunk[chunkCount]               // v6+, bone painting
```

The painting chunks use the chunk encoding above, but each `u16` holds a **bone id** instead of a colour, and `0` means "not painted". For each voxel:

1. If the painting has a non-zero value and a bone with that id exists, the voxel follows that bone.
2. Otherwise it follows the layer's `boneName`, if set.
3. Otherwise the voxel is static.

### Clip

```
string   name
f32      fps
i32      length                 // in frames
bool     loop
i32      trackCount
Track[trackCount]
```

**Track:** `string bone`, `i32 keyCount`, then `Key[keyCount]` sorted by frame.

**Key:**

| Field | Type | Notes |
|---|---|---|
| frame | `i32` | |
| rotation | 4 × `f32` | quaternion `x, y, z, w`, local rotation around the bone head |
| translation | `vec3` | local offset in voxel units |
| interpolation | `u8` | easing from **this** key to the next one: `0` linear, `1` smooth (smoothstep), `2` step, `3` ease-in (cubic), `4` ease-out (cubic), `5` back, `6` elastic, `7` bounce |

Between two keys, rotation uses spherical interpolation and translation uses linear interpolation, with the eased factor of the first key. Before the first key and after the last key the pose holds. A bone without a track stays in its rest pose.

### Posing

With row vectors (point × matrix, the `System.Numerics` convention), the world matrix of a bone is:

```
M(bone) = T(-head) · R(rotation) · T(head + translation) · M(parent)
```

`T` is a translation and `R` a rotation. A root bone uses the identity as `M(parent)`. A voxel vertex `v` bound to a bone moves to `v · M(bone)`.

### BoneColor (v7+)

`bool hasColor`, then `u8 r, g, b` if `hasColor`. This is the colour of the bone in the viewport.

### Attachments (v7+ / v8+)

An **attachment** layer is a prop such as a weapon held by a socket bone. The layer's `boneName` is the socket. It is exported as a separate object with its origin at the grip.

**Attachment (v8+):**

| Field | Type | Notes |
|---|---|---|
| grip | `vec3` | point of the prop that sits on the socket, voxel units |
| offset | `vec3` | extra translation |
| rotation | `vec3` | Euler degrees, combined as `q = qZ · qY · qX` (`System.Numerics` quaternion multiplication, so X is applied first) |
| scale | `f32` | uniform scale, `1` = none |

Layers that are not attachments still store an attachment record (zeros and scale `1`).

### ClipLink (v9+)

Variants of a character can share the animations of a base model (`animationSource`). Each clip of the variant records where it comes from:

```
string   linkedFrom             // "" = the model's own clip, else the name of the source model
// v9 only:
bool     customized             // true = the whole clip was edited locally
// v10+:
bool     overridesTiming        // fps / length / loop were changed locally
i32      overriddenCount
string   overriddenBones[overriddenCount]   // tracks edited locally, the rest follow the source
```

The clip data stored in the file is always complete, so a reader that ignores links still gets correct animations.

## Writing a minimal file

To write a project that VoxelCdx opens:

1. `VCDX`, version `10`, the info JSON, thumbnail length `0`.
2. Brotli-compress a body with:
   - `paletteSize = 1024` and 1024 slots, where unused slots can be `0,0,0,255` with VoxelCdx's default material: diffuse, roughness `0.8`, metallic `1`, emission `0`, ior `1.5`, transparency `0.9` (metallic only matters for metal, and ior and transparency only for glass);
   - 64 rows `("", false)`, `renderSettings = "{}"`, `pageCount = 0`, `activePage = 0`;
   - your models.
3. For a model without a rig, the rig block is:
   - `boneCount = 0`, then per layer `boneName = ""` and `chunkCount = 0`;
   - `clipCount = 0`, `boneShape = 0`, `armatureName = "Armature"`;
   - per layer `isAttachment = false`, then per layer an attachment record of zeros with scale `1`;
   - `isFavorite = false`, `animationSource = ""`.

## Version history

| Version | Changes |
|---|---|
| 4 | Uncompressed header with info JSON and PNG thumbnail. Several models per file. 1024-colour palette with materials. 16-bit voxels. |
| 5 | Rig block: bones, layer bones, animation clips. |
| 6 | Bone ids and per-voxel bone painting. |
| 7 | Bone shape, armature name, bone colours, attachment layers. |
| 8 | Attachment pose (grip, offset, rotation, scale). |
| 9 | Favourite flag, animation source and linked clips. |
| 10 | Per-bone and timing overrides for linked clips. |

Versions 1 – 3 were used only before the public release: one model, 256 colours, 8-bit voxels, no header. They are not described here.

## Contact

Questions or corrections: Discord [discord.gg/kAAgz4x2z2](https://discord.gg/kAAgz4x2z2) (sazixt, mercenarytf) or Telegram [@szxqpi](https://t.me/szxqpi).
