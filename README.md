# BC250 RADV GFX10 Mesh

Created by LoneWolf

Project: AMD BC-250 / Cyan Skillfish physical-GFX10 RADV Mesh Shader
support.

## Hardware

- PCI ID: `1002:13fe`
- ASIC family: `CHIP_GFX1013`
- Physical graphics level: `GFX10`
- Wave size: wave64

## Release identity

Suggested release name: **BC250 RADV GFX10 Mesh by LoneWolf —
Pre-Task/Cull Preview**

```text
UPSTREAM_BASE_COMMIT=6dfbc555b4128ee51139c5f78c5aba2594c9701b
UPSTREAM_BASE_DESCRIBE=mesa-26.1.4
RELEASE_COMMIT=3b42bd3590a3a5a350b10029a81df55cafbccfb9
RADV_SHA256=18fe7bf6b39c40d1b6b4f32edd942c071918f53b8ee9f8551d4c0547a65ac1b1
```

This project does **not** globally spoof GFX10.3. The physical family stays
`CHIP_GFX1013` and the physical `gfx_level` remains `GFX10`. Mesh targets the
physical-GFX10 NGG path, whose final hardware stage is
`AC_HW_NEXT_GEN_GEOMETRY_SHADER`. The BC250 project path disables
`GS_FAST_LAUNCH` for GFX1013.

The implementation uses direct/hybrid routing: proven direct-safe work stays
on the direct path, while overflow and conservative unknown cases use the
ordered graphics-queue record-once/raster-only fallback. Replay is not the
default. `FALSE_DIRECT=0` at this known-good checkpoint.

Final Fantasy VII Rebirth has been demonstrated working on BC250 with this
backend. The retained FF7 Mesh corpus contains 224 linked Mesh PSOs, all
classified `DIRECT` offline (`224/224`, with zero overflow, unknown,
rejected, or false-direct cases). This is an experimental preview and is not
a claim of full `VK_EXT_mesh_shader` compliance.

## Applying the patch

From a clean Mesa checkout at the exact base:

```sh
git clone https://gitlab.freedesktop.org/mesa/mesa.git
cd mesa
git checkout 6dfbc555b4128ee51139c5f78c5aba2594c9701b
git apply /path/to/bc250-radv-mesh-gfx10.patch
```

Build RADV in a fresh build directory. The following is one example; adjust
drivers and build options to the target distribution:

```sh
meson setup build \
  -Dvulkan-drivers=amd \
  -Dgallium-drivers=zink \
  -Dglx=disabled -Degl=disabled -Dgles2=disabled \
  -Dshared-llvm=disabled -Dllvm=disabled \
  -Dxmlconfig=disabled -Dlmsensors=disabled -Dvalgrind=disabled \
  -Dbuildtype=debugoptimized -Dbuild-tests=true \
  -Dbuild-aco-tests=false -Dbuild-radv-tests=false
ninja -C build src/amd/vulkan/libvulkan_radeon.so
```

Use an ICD manifest pointing to the resulting library. Do not overwrite a
system RADV installation.

## Experimental flags

The normal FF7-tested configuration is:

```sh
RADV_EXPERIMENTAL=bc250_mesh
```

The source defines these BC250-related `RADV_EXPERIMENTAL` names:

- `bc250_mesh` — opts in to the experimental BC250 Mesh exposure and
  graphics-queue path. This is required for the project Mesh path.
- `bc250_mesh_direct_prim` — enables the bounded research transform for
  generic Mesh per-primitive outputs and `PrimitiveId` through the direct
  physical-GFX10 representation. It is not required by the retained FF7
  corpus and does not provide full generic per-primitive support.
- `bc250_mesh_hybrid` — enables the ordered record-once/raster-only fallback
  for direct-representation capacity cases when used together with
  `bc250_mesh_direct_prim`. It is not required by the retained FF7 corpus.

Only enable the additional flags when testing the corresponding research
features:

```sh
RADV_EXPERIMENTAL=bc250_mesh,bc250_mesh_direct_prim
RADV_EXPERIMENTAL=bc250_mesh,bc250_mesh_direct_prim,bc250_mesh_hybrid
```

## FF7 capability switches

These optional switches are capability-query compatibility controls for the
historical FF7/DX12 negotiation surface:

- `RADV_BC250_ADVERTISE_TASK=1` may advertise the Task feature for the FF7
  capability check. It does **not** implement or make native Task execution
  safe.
- `RADV_BC250_EXPOSE_FSR=1` exposes the historically required fragment
  shading-rate/VRS capability surface used by the DX12 Ultimate negotiation.
  “FSR” here means fragment shading rate, not FidelityFX Super Resolution.
  It does not globally spoof GFX10.3.

Example environment (replace all paths with local paths):

```sh
PROTON_LOG=1 \
PROTON_LOG_DIR=/path/to/logs \
RADV_EXPERIMENTAL=bc250_mesh \
RADV_BC250_ADVERTISE_TASK=1 \
RADV_BC250_EXPOSE_FSR=1 \
VK_ICD_FILENAMES=/path/to/bc250_icd.json \
%command%
```

Read `KNOWN_LIMITATIONS.md` before enabling optional research or capability
switches.
