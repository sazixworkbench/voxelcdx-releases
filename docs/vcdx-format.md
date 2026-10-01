# .vcdx file format

This is the project file of [VoxelCdx](https://voxelcdx.pages.dev), format version 11 (app 1.1.5+). Wrote this up for people making importers / converters. If the doc and the app disagree, the app wins and the doc has a bug, so let us know.

## Basics

- everything is little endian
- `bool` is one byte (0 / 1)
- strings are .NET `BinaryWriter` strings: byte length as a 7-bit varint (low bits first, high bit = more bytes follow), then UTF-8. So `""` is just `00`, and a 300 byte string starts with `AC 02`
- `vec3` = 3 floats (x, y, z)
- right handed, **Y up** (same as glTF). Voxel `(x, y, z)` is the cube from `(x, y, z)` to `(x+1, y+1, z+1)`, coords can be negative
- a voxel is a palette index: `0` = empty, `1..1023` = colour

To and from MagicaVoxel (Z up):

```
vox = (x, -z - 1, y)
xyz = (vox.x, vox.z, -vox.y - 1)
```

When we import a .vox, colour index N stays N, so a straight index copy works both ways (for the first 256).

## File

```
char[4]  magic = "VCDX"
i32      version                  // 11 right now, refuse anything newer than you know
string   info                     // v4+, small json, see below
i32      thumbnailLength          // v4+, can be 0
u8[]     thumbnail                // png, 192x192, preview of the active model
...      body                     // brotli compressed, runs to the end of the file
```

The header is uncompressed on purpose so file pickers can grab the preview and model names cheaply.

`info` is only for display, you can ignore it when reading. When writing, fill it in like this:

```json
{"models": 2, "voxels": 1308, "size": [64, 64, 64], "names": ["Hero", "Hero2"]}
```

`voxels` = voxel count over all layers of all models. `size` = canvas size of the active model (or its bounding box if it has no canvas).

## Body

```
i32      paletteSize              // 1024
Palette  palette                  // the palette the voxels use
string   renderSettings           // json, lights/camera. skip it; "{}" is fine when writing
i32      pageCount                // extra palettes the user keeps around ("pages" in the UI)
i32      activePage
Page     pages[pageCount]
i32      modelCount
i32      activeModel
Model    models[modelCount]
```

```
Palette {
    Slot   slots[paletteSize]     // slot 0 is written but never used
    Row    rows[paletteSize / 16] // 64 rows of 16
}

Slot {
    u8  r, g, b, a                // a = 0 on slots 1+ means "not added yet" (an empty, growing page); draw it opaque if you meet it
    u8  material                  // 0 diffuse, 1 metal, 2 emissive, 3 glass
    f32 roughness                 // 0..1
    f32 metallic                  // 0..1, only used by metal
    f32 emission                  // 0 = off
    f32 ior                       // glass only
    f32 transparency              // glass only, 0..1
}

Row {
    string label                  // row name in the palette panel, "" is fine
    bool   groupStart             // draws a divider above the row
}

Page {
    string  name
    Palette palette
}
```

The render settings json is optional data for the app's own renderer: lights, camera, background (`"Sky"`, `"SolidColor"`, `"Transparent"`, `"Gradient"`, `"Image"`), quality, and the reference pictures the user models over (`References`, with the picture file bytes in base64). Readers can skip the whole string; none of it changes the model.

### Models and layers

```
Model {
    string name
    bool   hasCanvas
    i32    canvasMin[3], canvasSize[3]   // only if hasCanvas
    i32    layerCount
    i32    activeLayer
    Layer  layers[layerCount]
    Rig    rig                           // v5+, always present even without bones
}

Layer {
    string name
    bool   visible
    bool   locked
    i32    chunkCount
    Chunk  chunks[chunkCount]
}

Chunk {
    i32 cx, cy, cz                // chunk coordinate, one chunk = 32x32x32
    u16 voxels[32768]
}
```

The voxel at local `(lx, ly, lz)` sits at `(cx*32 + lx, cy*32 + ly, cz*32 + lz)` and is stored at index `lx | ly << 5 | lz << 10` (x fastest). Empty chunks are not written. We write chunks sorted by z, y, x but don't rely on that. Values >= 1024 are garbage, the app clamps them to 1023.

The canvas is just the editing box from the editor, voxels are allowed outside of it.

## Rig (v5+)

Parts marked vN only exist from that version on.

```
Rig {
    i32          boneCount
    Bone         bones[boneCount]
    LayerBinding bindings[layerCount]    // one per layer, same order
    i32          clipCount
    Clip         clips[clipCount]

    // v7+
    u8           boneShape               // 0 octahedral, 1 stick. display only
    string       armatureName            // root node name in fbx/gltf exports, usually "Armature"
    BoneColor    colors[boneCount]
    bool         isAttachment[layerCount]

    // v8+
    Attachment   attachments[layerCount]

    // v9+
    bool         isFavorite              // pinned in the model list
    string       animationSource         // "" or name of the model this one borrows clips from
    ClipLink     links[clipCount]
}

Bone {
    string name                   // unique per model
    string parent                 // "" = root. parents can come after children, match by name
    vec3   head                   // joint, voxel units, model space
    vec3   tail
    i32    id                     // v6+, unique, > 0
}

LayerBinding {
    string boneName               // "" = layer isn't bound
    i32    chunkCount             // v6+
    Chunk  paint[chunkCount]      // v6+, same layout as voxels but holds bone ids, 0 = not painted
}
```

Which bone moves a voxel:

1. painted bone id, if non-zero and that bone exists
2. else the layer's `boneName`
3. else nothing, the voxel is static

### Animation

```
Clip {
    string name
    f32    fps
    i32    length                 // frames
    bool   loop
    i32    trackCount
    Track  tracks[trackCount]
}

Track {
    string bone
    i32    keyCount
    Key    keys[keyCount]         // sorted by frame
    bool   scaleChildren          // v11+, see posing below. false = scale stays on this bone
}

Key {
    i32  frame
    f32  rotation[4]              // quaternion x y z w, local, around the bone head
    vec3 translation              // local offset, voxel units
    u8   interpolation            // easing towards the NEXT key, see below
    vec3 scale                    // v11+, around the bone head, (1, 1, 1) = none
}
```

Interpolation: `0` linear, `1` smooth (smoothstep), `2` step, `3` ease in (cubic), `4` ease out (cubic), `5` back, `6` elastic, `7` bounce.

Between two keys the eased factor comes from the first key, rotation is slerped, translation and scale lerped. Before the first key / after the last one the pose just holds. Bones without a track stay in rest pose. Files older than v11 have no scale, treat it as (1, 1, 1).

Posing, row vectors (`point * matrix`, like System.Numerics):

```
M(bone) = T(-head) * S(scale) * R(rotation) * T(head + translation) * B(parent)
P(bone) = T(-head) *            R(rotation) * T(head + translation) * B(parent)

B(parent) = M(parent)   if the parent's scale is (1, 1, 1) or its track has scaleChildren
            P(parent)   otherwise
```

Root bones use identity for `B(parent)`. A vertex `v` bound to the bone ends up at `v * M(bone)`. `P` is the same thing without the bone's own scale, it's what children follow when the scale is meant for that one bone only (a fat pelvis with normal legs). Without any scale `M = P` and it's the old v10 formula.

Our glTF / FBX exports bake this into plain TRS nodes per frame. A bone that is scaled on its own and has children gets an extra child `<name>_scale` that carries the scale and the bone's voxels, so engines where scale always goes to the children still show the same thing.

### Colours, attachments, links

```
BoneColor {                       // v7+, bone colour in the viewport
    bool hasColor
    u8   r, g, b                  // only if hasColor
}

Attachment {                      // v8+
    vec3 grip                     // point of the prop that sits on the socket
    vec3 offset
    vec3 rotation                 // euler degrees, q = qZ * qY * qX (x applied first)
    f32  scale                    // 1 = none
}

ClipLink {                        // v9+
    string linkedFrom             // "" = own clip, else source model name
    bool   customized             // v9 only: whole clip edited locally
    bool   overridesTiming        // v10+: fps/length/loop changed locally
    i32    overriddenCount        // v10+
    string overriddenBones[]      // v10+: tracks edited locally
}
```

An attachment layer is a prop (usually a weapon) held by a socket bone, the socket is the layer's `boneName`. It gets exported as its own object with the origin at `grip`. Normal layers still write an attachment record (zeros, scale 1).

Links are how character variants share the base model's animations. The clip data in the file is always complete, so if you don't care about links just read the clips and ignore `ClipLink`.

## Writing a file from scratch

Minimum the app will open:

- `VCDX`, version 11, the info json, thumbnail length 0
- then brotli of:
  - `1024`, 1024 slots (unused ones can be `0,0,0,255`, diffuse, roughness 0.8, metallic 1, emission 0, ior 1.5, transparency 0.9, which are the app's defaults), 64 rows of `""` / `false`
  - `"{}"` for render settings, `0` pages, active page `0`
  - your models

A model without a rig still needs the whole rig block: 0 bones, per layer an empty binding (`""`, 0 chunks), 0 clips, shape 0, `"Armature"`, per layer `false`, per layer an attachment of zeros with scale 1, `false`, `""`.

## Version history

- 4: uncompressed header with json + thumbnail, multiple models, 1024 colour palette with materials, 16-bit voxels
- 5: rig (bones, layer bones, clips)
- 6: bone ids, per-voxel bone painting
- 7: bone shape, armature name, bone colours, attachment layers
- 8: attachment pose
- 9: favourites, animation source, linked clips
- 10: per-bone / timing overrides on linked clips
- 11: bone scale in keys, scaleChildren per track

1 to 3 were pre-release (single model, 256 colours, 8-bit voxels, no header) and aren't documented. You won't run into them.

Questions: [Discord](https://discord.gg/kAAgz4x2z2) (sazixt / mercenarytf) or [Telegram](https://t.me/szxqpi).
