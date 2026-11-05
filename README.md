# pixal3d.cpp on an AMD GPU (Linux)

Turn images into textured 3D models (GLB files) on your own GPU, with no Python at runtime.
This is a fork of [raven38/pixal3d.cpp](https://github.com/raven38/pixal3d.cpp), which builds on
[pwilkin/trellis.cpp](https://github.com/pwilkin/trellis.cpp) (MIT). It runs two model families:

- **TRELLIS.2** (Microsoft): one photo or picture in, 3D model out. Background removal is built in.
- **Pixal3D** (TencentARC): camera-aware reconstruction from either one cut-out image
  (single view) or four views of the same object (multiview).

This README is a short, tested path from a fresh clone to a GLB on an AMD card. The original
project README is kept as [`README.upstream.md`](README.upstream.md). It covers every flag, the
HTTP server, the desktop app, the model details and benchmarks.

**Tested on:** Fedora 43, AMD Radeon RX 7900 XT (20 GB, `gfx1100`), ROCm 6.4 from the Fedora
repos, Mesa 25.3.6 (RADV), November 2026.

> **Build with ROCm, not Vulkan.** On this card, Vulkan gives broken Pixal3D results while ROCm
> matches the reference. See [Vulkan problems](#vulkan-problems-on-the-rx-7900-xt).

## 1. Install the build tools

```sh
# needed for every build
sudo dnf install git cmake ninja-build gcc-c++
# ROCm build (recommended)
sudo dnf install rocm-hip-devel hipblas-devel rocblas-devel rocminfo
# Vulkan build (optional)
sudo dnf install vulkan-headers vulkan-loader-devel glslc glslang spirv-headers-devel
# Hugging Face CLI ("hf"), for the model downloads
pip install --user -U huggingface_hub
```

ROCm needs your user in the `render` and `video` groups. Check with `id -nG`. If either is
missing, run `sudo usermod -aG render,video $USER` and log in again.

## 2. Get the source

```sh
git clone <this repo> pixal3d.cpp
cd pixal3d.cpp
git submodule update --init --recursive
```

The first configure step downloads libwebp, so it needs network access.

## 3. Build

### ROCm (recommended)

```sh
ROCM_LLVM=$(hipconfig -l)          # /usr/lib64/rocm/llvm/bin on Fedora
HIPCXX=$ROCM_LLVM/clang++ HIP_PATH=$(hipconfig -R) \
cmake -S . -B build-rocm -G Ninja -DCMAKE_BUILD_TYPE=Release \
    -DGGML_HIP=ON -DGGML_HIP_GRAPHS=OFF -DGPU_TARGETS=gfx1100 \
    -DCMAKE_C_COMPILER=$ROCM_LLVM/clang \
    -DCMAKE_CXX_COMPILER=$ROCM_LLVM/clang++ \
    -DCMAKE_HIP_COMPILER=$ROCM_LLVM/clang++ \
    -DCMAKE_PREFIX_PATH=$(hipconfig -R)
cmake --build build-rocm -j --target trellis-cli
```

- `gfx1100` is the RX 7900 XTX / XT / GRE. For another card, `rocminfo | grep gfx` shows the
  right target.
- `-DGGML_HIP_GRAPHS=OFF` matches the project's own release build. HIP graph capture stalls on
  these graphs.
- `--target trellis-cli` builds only the command-line tool. Leave it out to also build
  `trellis-server` and the tests.

### Vulkan (optional)

```sh
cmake -B build -G Ninja -DCMAKE_BUILD_TYPE=Release -DGGML_VULKAN=ON
cmake --build build -j --target trellis-cli
```

On the RX 7900 XT this build is fine for TRELLIS.2 single-image runs but not for Pixal3D. See
[Vulkan problems](#vulkan-problems-on-the-rx-7900-xt).

## 4. Download the models

All three model sets go into one folder, `models/q8`, which git ignores. `q8` means 8-bit
quantized weights, which the project describes as near-lossless. They take about 22 GB in total.

```sh
# TRELLIS.2, plus the files Pixal3D shares with it (DINOv3, decoders, BiRefNet): ~10 GB
hf download ilintar/trellis2-gguf --include "q8/*" --local-dir models

# Pixal3D single view: ~5.9 GB
hf download raven38/pixal3d-sv-q8_0-v1 --include "pixal3d_*" --local-dir models/q8

# Pixal3D multiview: ~5.9 GB
hf download raven38/pixal3d-q8_0-v1 --include "pixal3d_*" --local-dir models/q8
```

You only need the Pixal3D sets for the modes you use. With `--include "pixal3d_*"`, only the
Pixal3D-specific files are downloaded, next to the shared files from the first set. That shared
folder is fine for the command-line tool. (The desktop app and `trellis-server` expect a separate
folder per Pixal3D set; see `README.upstream.md`.)

## 5. Generate models

All commands use the ROCm binary and are run from the repo root. Notes:

- `--require-gpu` stops with an error instead of quietly falling back to the very slow CPU path.
- The default seed is 42. Change it with `--seed N` to get a different result from the same
  input.
- Besides the `.glb`, the tool also writes `<name>.ply` (the mesh) and `<name>_base.png` (the
  colour texture) next to it.
- To view a result, import it in Blender (File → Import → glTF 2.0) or drop it on
  <https://gltf-viewer.donmccurdy.com/>.

```sh
mkdir -p out
```

### 5.1 TRELLIS.2: one image (no Pixal3D)

```sh
./build-rocm/trellis-cli assets/goblin.png out/goblin.glb --models models/q8 --require-gpu
```

- Any PNG or JPG works. The background is removed automatically. One object on a plain
  background works best.
- The default resolution is 1024. `--res 512` is faster and lighter, and `--res 1536` is
  sharper and heavier.
- The goblin takes about 4 minutes on the RX 7900 XT.

**TRELLIS.2 with several images** (2 to 8 pictures of one object, no camera poses needed):

```sh
./build-rocm/trellis-cli --trellis2-mv my_photos/ out/obj.glb --models models/q8 --require-gpu
```

The images are read in natural filename order. This mode uses the normal TRELLIS.2 weights. It
is separate from Pixal3D multiview: it ignores `transforms.json` and camera data.

### 5.2 Make a cut-out (images without transparency)

Pixal3D does no background removal of its own, so it needs PNGs with a real transparent
background.

**For single view**, use the built-in BiRefNet matting:

```sh
./build-rocm/trellis-cli assets/goblin.png out/goblin.glb --models models/q8 --bg-only
```

This writes `out/goblin_cutout.png` and nothing else. The name comes from the output path you
give, and no GLB is made. The cut-out is cropped square around the object, which is what
`--sv-image` wants anyway.

**For multiview, don't use `--bg-only`.** It re-crops every view, so the views no longer match
their cameras. Use a matting tool that keeps the original canvas instead, for example
[rembg](https://github.com/danielgatis/rembg):

```sh
uvx --python 3.12 --from "rembg[cpu,cli]" rembg i front.jpg front.png
```

The output has the same size as the input, with an alpha channel. On first use, rembg downloads
its default model, BRIA RMBG-2.0 (about 1 GB, stored in `~/.rembg/`).

### 5.3 Pixal3D: single view

```sh
./build-rocm/trellis-cli --sv-image out/goblin_cutout.png --models models/q8 --require-gpu out/goblin_sv.glb
```

- The input must be an RGBA cut-out (see 5.2).
- The tool crops it like the reference preprocess and picks the single-view weights by itself.
  It saves the staged `input.png` and `transforms.json` in `out/goblin_sv.svviews/`.
- If the picture was taken with a wide lens, pass its horizontal field of view in radians with
  `--fov` (the default is 0.349, i.e. 20°).
- Pixal3D always runs at 1024 or 1536 (`--res 512` is refused). By default it makes a mesh of
  about 1M faces with a 4096² texture, so GLBs are 30 to 70 MB. Expect about 2 minutes per
  model on the RX 7900 XT.

### 5.4 Pixal3D: multiview

Try Pixal3D's own four-view sample first:

```sh
mkdir -p pixal3d-example
for f in transforms.json view00_azim000.png view01_azim090.png view02_azim180.png view03_azim270.png; do
  curl -fL -o pixal3d-example/$f \
    https://raw.githubusercontent.com/TencentARC/Pixal3D/master/assets/mv_images/example/$f
done
./build-rocm/trellis-cli --views pixal3d-example --models models/q8 --require-gpu out/example_mv.glb
```

On the RX 7900 XT this takes about 5.5 minutes. The output matches the project's reference run;
see the Vulkan section for the numbers.

**Your own four views** must show one object from the canonical turntable rig: elevation 0°,
azimuth 0° / 90° / 180° / 270°, 20° field of view, camera distance 3.119. Each view must be:

- an RGBA PNG with a transparent background, matted without re-cropping (see 5.2);
- at the same scale as the others, with the object centred and about 85 % of the frame height.

The 90° camera sits to the right of the front camera, so it sees the **left** side of a subject
that faces the front camera. The upstream docs call this view "right".

Then do one of these:

- **With `transforms.json` (recommended):** copy `pixal3d-example/transforms.json` into your
  folder and change each `file_path` to your own image, keeping the order front, 90°, back,
  270°.
- **Without `transforms.json`:** put exactly four images in the folder and pass
  `--mesh-scale 1.0`. They are taken in natural filename order as front, 90°, back, 270°, so
  name them to sort that way, for example `0_front.png`, `1_left.png`, `2_back.png`,
  `3_right.png`.

```sh
./build-rocm/trellis-cli --views my_views/ --models models/q8 --require-gpu out/my_mv.glb
```

**Check the result against the inputs** instead of eyeballing it. This script projects the mesh
through the input cameras and compares silhouettes. It needs numpy and pillow, and part of its
output text is in Japanese.

```sh
python3 tools/silhouette_iou.py out/my_mv.glb my_views/ --res=256
```

Write `--res=256` with `=`; `--res 256` crashes. A mean IoU around 0.98 is what the sample
gets, and a broken run scores around 0.2. Character turnarounds often score lower on the two
side views; `README.upstream.md` ("Known multiview failure modes") explains why.

### Useful flags

| flag | effect |
|---|---|
| `--seed N` | different random seed, so a different result (default 42) |
| `--res 512\|1024\|1536` | geometry resolution (512 works only for TRELLIS.2) |
| `--no-texture` | geometry only |
| `--atlas PX` | texture size |
| `--webp off` | PNG textures instead of WebP (MeshLab can't read WebP GLBs) |
| `--bg-removal threshold\|birefnet` | force a background-removal method (TRELLIS.2 image input) |
| `--gpu N` | pick a GPU index (`-1` = CPU) |

Run `./build-rocm/trellis-cli --help` for the full list.

## Vulkan problems on the RX 7900 XT

**Summary:** on this card and driver, the Vulkan build produces wrong Pixal3D output. The same
models are correct on ROCm on the same GPU, and on the CPU backend. Use the ROCm build.

### What goes wrong

| mode | Vulkan, default | Vulkan, `--no-fa` | ROCm, default |
|---|---|---|---|
| TRELLIS.2, one image | works | not needed | works |
| Pixal3D single view | fails: "no voxels produced" | works | works |
| Pixal3D multiview | fails: "no voxels produced" | runs, but the mesh is unrecognizable | works |

Measured on the Pixal3D multiview sample (seed 1), against the project's reference run on Apple
Metal:

| stage | reference | ROCm | Vulkan `--no-fa` |
|---|---|---|---|
| structure voxels | 4438 | 4437 | 4338 |
| HR shape latent, std | 5.25 | 5.30 | **3077** |
| decoded voxels | 4.71 M | 4.73 M | **1.20 M** |
| silhouette IoU vs inputs | 0.979 | 0.981 | **0.20** |

### Why

Attention, the core operation of these transformer models, has two code paths in this program:

1. **FlashAttention (the default).** With a padding mask, it gives wrong results on Vulkan in
   one specific place: Pixal3D's cross-attention over its five "global" image tokens. The first
   stage then finds no voxels, and the run stops with "no voxels produced". TRELLIS.2 doesn't
   use that attention shape, so it is unaffected.
2. **Plain attention (`--no-fa`).** This one breaks at large token counts. In the multiview
   high-resolution stage (about 17,500 tokens) the shape latent blows up and the mesh comes out
   as garbage. Single view uses far fewer tokens and survives.

### Is it the driver or the program?

Neither the model code nor the pipeline is at fault. The identical computation graph gives
correct results on the CPU backend and on ROCm on the same GPU. The project also records it as
correct on CUDA, Metal and NVIDIA Vulkan. The fault sits in the Vulkan layer underneath, either
in ggml's Vulkan kernels (the copy in `thirdparty/ggml`, commit `737e88f2` from 2026-08-19) or
in how the RADV driver compiles and runs them. These tests couldn't tell which:

- **Kernel switches change the outcome.** `GGML_VK_DISABLE_COOPMAT=1` (no matrix cores) or
  `GGML_VK_DISABLE_INTEGER_DOT_PRODUCT=1` fixes the "no voxels" bug in the first stage.
  `GGML_VK_DISABLE_F16`, `_FUSION` and `_GRAPH_OPTIMIZE` don't. So one specific kernel path
  misbehaves on this card.
- **No single Vulkan setting fixes everything.** With `GGML_VK_DISABLE_COOPMAT=1`, multiview
  gets past the first stage but then aborts later with non-finite values.
- **No driver update was available.** In November 2026, Fedora 43 had no Mesa newer than 25.3.6,
  not even in `updates-testing`. A newer Mesa, or a newer ggml, may fix it. If you try one, use
  the checks below to see whether it did.

### If you must use Vulkan

- **TRELLIS.2:** works as is.
- **Pixal3D single view:** add `--no-fa`.
- **Pixal3D multiview:** `TRELLIS_FA_NOMASK=1 ./build/trellis-cli --views …` keeps
  FlashAttention but drops its padding mask. That is a debug switch, not an official option. On
  the sample it scored IoU 0.978 (reference 0.979), and it took 8:46 against 5:39 on ROCm.

### Checking whether a run is healthy

- **The log:** the line starting `[stats] HR slat` should show `std=` around 5. Broken runs
  show thousands.
- **The finished mesh:** run `tools/silhouette_iou.py` on it (see 5.4).

## The "inner shell" in the meshes

Output meshes have a second, inward-facing wall just inside the outer surface. It looks as if
Blender's Solidify modifier had been applied. This is not a bug. The final remesh step builds
the surface from an *unsigned* distance field, the same way the reference TRELLIS.2
postprocess does. It places a surface a small distance on both sides of the model's thin shell,
which gives an outer skin plus an inner skin about one voxel inside it. Hunyuan3D 2.0 doesn't
have this because it uses a signed field with marching cubes.

To strip the inner wall in **MeshLab** (generate with `--webp off`, since MeshLab can't read
WebP textures):

1. **File → Import Mesh**, pick the `.glb`.
2. **Filters → Quality Measure and Computations → Compute Ambient occlusion**, Rays = 128.
3. **Filters → Selection → Select by Face Quality**: Min 0, Max 0, Inclusive on. This selects
   the faces no ray can escape from.
4. **Filters → Selection → Delete Selected Faces and Vertices**.
5. **Filters → Cleaning and Repairing → Remove Isolated pieces (wrt Face Num.)**, size 100.
6. **File → Export Mesh As → OBJ**, with texture saving off. Copy `<name>_base.png` next to
   the OBJ and change `map_Kd texture_0` in the `.mtl` to that file name.

On the goblin this kept the whole outer surface and dropped about two-thirds of the faces.
Limitations:

- **No GLB export:** MeshLab can't write GLB. For a GLB, import the OBJ into Blender and export
  from there.
- **Open edges:** the mesh is no longer closed where the inner wall met openings.
- **Partly visible wall stays:** inner wall that can be seen through an opening gets some light,
  so it isn't removed.

## Credits and license

- **Code:** MIT. © 2026 Piotr Wilkin (trellis.cpp) and raven (pixal3d.cpp changes). See
  [`LICENSE`](LICENSE) and [`NOTICE`](NOTICE) for all third-party attributions.
- **Models:** Microsoft [TRELLIS.2](https://github.com/microsoft/TRELLIS.2) and TencentARC
  [Pixal3D](https://github.com/TencentARC/Pixal3D), each under its own license.
