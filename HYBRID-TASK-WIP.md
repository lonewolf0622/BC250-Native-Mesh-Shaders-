# Hybrid TASK + Native MESH — Work in Progress

> **Experimental / not GPU validated.**
>
> This work is separate from `bc250-native-mesh-v1-known-good.patch`. Do not fold it into the known-good V1 baseline until the hybrid path has independently passed materialization and runtime validation.

## Why hybrid TASK

RADV's normal GFX10.3 TASK architecture is a two-engine design:

```text
TASK on ACE/MEC
    |
    v
task draw/payload rings
    |
    v
gang synchronization
    |
    v
MESH on graphics
```

That path depends on GFX10.3+ TASK packets, task rings, and a usable compute/ACE path.

On BC-250:

- the physical graphics device is GFX10/GFX1013;
- the compute queue is intentionally unavailable/broken;
- native TASK packet/ring behavior is unvalidated and high risk.

This project therefore does **not** try to force native GFX10.3 TASK onto BC-250.

## Chosen architecture

The current design emulates only the unsupported TASK scheduling layer while preserving native MESH:

```text
Application MESA_SHADER_TASK
        |
        | clone + lower
        v
internal MESA_SHADER_COMPUTE producer
on graphics/general cmd_buffer->cs
        |
        +--------------------------+
        |                          |
        v                          v
12-byte XYZ indirect buffer    payload buffer
        |                          |
        +-------------+------------+
                      |
                      v
native indirect MESH multi-draw
DISPATCH_MESH_INDIRECT_MULTI 0x4C
                      |
                      v
existing native BC-250 MESH shader
```

No native TASK hardware path is required.

## Hard architectural rules

The hybrid path must not enable:

- AMD_IP_COMPUTE submission
- ACE/MEC TASK execution
- native TASK packets
- native task control/draw/payload rings
- gang submission
- GFX10.3 device spoofing

The application MESH shader must remain:

```text
MESA_SHADER_MESH
```

and continue through the existing validated BC-250 native-MESH path.

## Buffer ABI

The design uses **separate XYZ and payload buffers**.

### XYZ indirect buffer

For TASK workgroup `N`:

```text
+0  uint32_t mesh_x
+4  uint32_t mesh_y
+8  uint32_t mesh_z
```

Stride:

```text
12 bytes
```

This preserves the already validated native indirect-MESH command layout.

### Payload buffer

For TASK workgroup `N`:

```text
payload_address =
    payload_base
    + N * task_payload_stride
    + payload_offset
```

where:

```text
task_payload_stride = align_up(task_payload_size, 16)
```

## TASK producer lowering

The logical TASK shader is cloned and converted into an internal COMPUTE producer.

Current intended formulas:

### XYZ

```text
xyz_base + task_linear_id * 12
```

`EmitMeshTasksEXT(x, y, z)` becomes stores of `x`, `y`, and `z` to that record.

### Payload stores

```text
payload_base
+ task_linear_id * task_payload_stride
+ payload_offset
```

The producer must contain no native TASK execution intrinsics by the time it reaches COMPUTE compilation.

## Native MESH payload loads

The MESH stage remains native.

Hybrid task-payload reads are lowered to:

```text
task_payload_base
+ DrawID * task_payload_stride
+ payload_offset
```

The WIP currently reserves internal MESH user-data slots for:

```text
task_payload_base
task_payload_stride
```

and requests the existing native DrawID path.

The numeric SGPR layout must be proven collision-free before GPU execution.

## Why DrawID is central

The intended mapping is:

```text
TASK workgroup 0 -> MESH draw 0 -> DrawID 0 -> payload record 0
TASK workgroup 1 -> MESH draw 1 -> DrawID 1 -> payload record 1
...
```

One TASK workgroup's `EmitMeshTasksEXT(x,y,z)` becomes one native MESH indirect draw with dimensions `{x,y,z}`.

All child MESH workgroups for that draw share the same DrawID/payload record while receiving normal MESH WorkGroupID coordinates and NumWorkGroups dimensions.

This mapping is architecturally accepted but **not yet GPU validated**.

## Current implementation status

The WIP has progressed beyond a design-only prototype.

Implemented or partially implemented:

- `radv_graphics_pipeline::bc250_hybrid_task`
- TASK+MESH structural pipeline acceptance
- logical TASK NIR cloning
- TASK -> COMPUTE producer lowering
- `EmitMeshTasksEXT` lowering
- task-payload store lowering
- internal producer ownership
- internal COMPUTE producer creation path
- hidden native-MESH payload ABI
- DrawID request for hybrid payload indexing
- native MESH task-payload load lowering
- initial runtime bridge scaffolding
- graphics/general `cmd_buffer->cs` producer model
- reuse of the 12-byte native MESH indirect path

No hybrid TASK GPU execution has occurred yet.

## Current blocker: malformed producer NIR during ABI lowering

The current blocker is entirely CPU/compiler-side.

During TASK+MESH pipeline materialization, the internal COMPUTE producer reaches NIR validation/serialization with malformed `nir_src` bookkeeping.

Observed validator errors:

```text
%29 = iadd %11, %28
error: src->_parent & SRC_TAG_SEEN

%34 = iadd %9, %33
error: src->_parent & SRC_TAG_SEEN

error: state->nr_tagged_srcs == 0
```

The original serializer failure was:

```text
nir_serialize.c:108
write_lookup_object()
Assertion `entry' failed
```

The backtrace confirmed the malformed shader is the hybrid COMPUTE producer:

```text
radv_bc250_compile_hybrid_task_producer
 -> vk_meta_create_compute_pipeline
 -> radv_CreateComputePipelines
 -> radv_compute_pipeline_hash
 -> vk_pipeline_hash_shader_stage_blake3
 -> nir_serialize
 -> write_alu
 -> write_lookup_object
```

The failed lookup is a `nir_def *` referenced by an ALU source.

## Isolation so far

The producer is valid before the ABI-lowering section.

The current pass sequence under investigation is:

1. producer before ABI lowering
2. `ac_nir_lower_intrinsics_to_args`
3. `radv_nir_lower_abi`
4. `ac_nir_lower_global_access`
5. `nir_lower_int64`
6. final producer validation

Pass-by-pass validation is gated by:

```text
BC250_HYBRID_NIR_VALIDATE_PASSES=1
```

for the COMPUTE NIR named:

```text
bc250_hybrid_task_producer
```

The immediate next task is to identify the **first individual ABI sub-pass** after which the producer becomes invalid.

## Host / sandbox note

Codex development runs inside a bubblewrap sandbox with a private `/dev`, so host Vulkan/GPU tests must run outside that sandbox.

The host BC-250 is healthy:

```text
PCI:       0000:01:00.0
PCI ID:    1002:13fe
driver:    amdgpu
DRM:       /dev/dri/card1
render:    /dev/dri/renderD128
Vulkan:    AMD BC-250 (RADV GFX1013)
```

Known-good direct native-MESH control on the host:

```text
center_rgba=255,0,0,255
NON_BLACK_PIXELS=968
BOUNDING_BOX=10,10,53,52
PASS
```

## Next diagnostic

Run on the normal host:

```bash
cd /home/deck/mesh-test/mesa-bc250-lavapipe-gpu

BC250_HYBRID_NIR_VALIDATE_PASSES=1 \
./tests/bc250-native-mesh/run-hybrid-materialize-gdb.sh
```

Then inspect:

```bash
grep -E \
"NIR validation failed BC250 hybrid|BC250 hybrid after|src->_parent|Assertion" \
/tmp/bc250-hybrid-materialize-gdb.log | head -n 100
```

Do not execute hybrid TASK GPU work until the compiler/materialization problem is resolved.

## Debugging rules

The investigation should stay narrow:

1. identify the first ABI sub-pass that introduces invalid NIR;
2. decode `%9`, `%11`, `%28`, `%33`, `%29`, `%34`;
3. identify the malformed `nir_src` relationship;
4. repair it using normal NIR builder/rewrite APIs;
5. require `nir_validate_shader()` to pass at every instrumented boundary;
6. require zero serializer-order bad-ALU findings;
7. successfully materialize the producer and native MESH shaders;
8. perform the numeric SGPR collision audit;
9. only then attempt the first GPU execution.

Do **not** "fix" the problem by:

- modifying `nir_validate.c`
- modifying `nir_serialize.c`
- clearing `SRC_TAG_SEEN`
- manually patching `_parent`
- bypassing pipeline hashing
- bypassing serialization
- disabling assertions

The producer NIR itself must be valid.

## First intended runtime slice

After materialization is clean:

```text
TASK grid: 1x1x1
TASK payload: one small sentinel
EmitMeshTasksEXT: 1x1x1

MESH:
3 vertices
1 triangle
payload controls deterministic output
```

Expected path:

```text
graphics-queue internal COMPUTE producer
 -> XYZ = 1,1,1
 -> payload sentinel
 -> synchronization
 -> DISPATCH_MESH_INDIRECT_MULTI 0x4C
 -> DrawID 0
 -> native MESH reads payload record 0
 -> red triangle
```

The first runtime test must not include:

- multiple TASK workgroups
- TASK indirect
- TASK indirect-count
- zero-group compaction
- DGC + TASK
- CullPrimitive
- MESH output scratch ring
- stress/repeated execution

## Next milestone after 1x1x1

If the first hybrid test passes, the next critical test is:

```text
TASK grid = 2x1x1
```

with different payload values per TASK workgroup.

That validates:

```text
record 0 -> DrawID 0 -> payload 0
record 1 -> DrawID 1 -> payload 1
```

Only after that should work expand toward general task grids, zero-emission handling, indirect TASK, indirect-count TASK, DGC+TASK, or shader-object coverage.

## Relationship to native MESH V1

Hybrid TASK work must remain separate from:

```text
bc250-native-mesh-v1-known-good.patch
```

until independently validated.

The stable V1 native-MESH patch must remain usable even if the hybrid branch is broken.
