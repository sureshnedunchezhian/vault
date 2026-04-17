# Scope: Extract XRDMA ULP from FunOS into Storage-xDPU-XStore

## Summary
The XRDMA ULP (Upper Layer Protocol) currently lives inside the FunOS source
tree. We want to relocate it into the **Storage-xDPU-XStore** repo
(`msazure.visualstudio.com/One/_git/Storage-xDPU-XStore`), which already
hosts XStore/csnetlib storage code and is the primary consumer of the XRDMA
API. Storage-xDPU-XStore builds `libfunstorage.a` and delivers it to FunOS
via FunSDK.

After the move, XRDMA source will be compiled as part of `libfunstorage`
alongside the existing storage code that already consumes it. FunOS will
continue to consume XRDMA through the public SDK header
(`sdk_include/FunOS/networking/rdma/ulp/xrdma/xrdma.h`) only.

## Why
- Decouple XRDMA lifecycle (development, review, release) from FunOS.
- Enforce a stable public API: today several FunOS files reach directly into
  XRDMA internals, which blocks independent evolution of the ULP.
- Co-locate XRDMA with its primary consumer (XStore/csnetlib), which already
  lives in Storage-xDPU-XStore and uses XRDMA exclusively through the SDK
  header.

## Current footprint in FunOS
| Area | Location | Size |
|---|---|---|
| XRDMA core | `networking/rdma/ulp/xrdma/` | 27 files, ~9.3K LOC |
| XRDMA CM glue | `services/rdma_cm/ulp/xrdma/xrdma_cm.c` | ~740 LOC |
| XRDMA apps | `apps/rdma/xrdma_ping.c`, `apps/rdma/xrdma_test_utils.{c,h}` | ~2.8K LOC |
| XRDMA tests | `tests/xrdma_test.c` + xrdma branches in `tests/rdma_test.c`, `tests/rdma_cpi_test.c` | ~6K LOC |
| Public API (stays in FunOS) | `sdk_include/FunOS/networking/rdma/ulp/xrdma/xrdma.h` | — |

## Storage-xDPU-XStore repo (destination)
The repo already:
- Builds `libfunstorage-$(MACHINE).a` and installs into FunSDK via
  `install-libfunstorage` target.
- Has `src/build.mk` with `SUB_DIRS := storage storage_apps kv`.
- Uses `<FunOS/networking/rdma/ulp/xrdma/xrdma.h>` (SDK header only) in
  `src/storage/xstore/csnetlib/csnetlib_xrl*.{c,h}` — no internal headers.
- Has its own CI pipeline (`OneBranch.Official.FunStorage.yml`), test
  apps (`src/storage_apps/`), and FoD automation (`src/fod_automation/`).

The XRDMA code will land as a new `SUB_DIRS` entry (e.g. `src/xrdma/`).

## What moves out of FunOS (into Storage-xDPU-XStore)
1. All 27 files under `networking/rdma/ulp/xrdma/`.
2. `services/rdma_cm/ulp/xrdma/xrdma_cm.c` (XRDMA-specific CM dispatch).
3. `apps/rdma/xrdma_ping.c`, `apps/rdma/xrdma_test_utils.{c,h}`.
4. `tests/xrdma_test.c`, plus the xrdma-only branches split out of
   `tests/rdma_test.c` and `tests/rdma_cpi_test.c`.
5. XRDMA-specific JSON dump helpers currently hosted in
   `props_bridges/dpsock_bridge.c` and `props_bridges/cpsock_bridge.c`
   (see coupling #3 below).

## Proposed layout in Storage-xDPU-XStore
```
src/
  xrdma/                        # NEW — added to SUB_DIRS
    build.mk                    # SRC_FILES for XRDMA core
    core/                       # ex-networking/rdma/ulp/xrdma/*
    cm/                         # ex-services/rdma_cm/ulp/xrdma/xrdma_cm.c
    bridges/                    # ex-props_bridges/* XRDMA JSON generators
  storage_apps/
    xrdma_ping.c                # ex-apps/rdma/xrdma_ping.c
    xrdma_test_utils.{c,h}      # ex-apps/rdma/xrdma_test_utils.*
    xrdma_test.c                # ex-tests/xrdma_test.c
```
XRDMA objects will be compiled into `libfunstorage.a` alongside existing
storage code, and delivered to FunOS through FunSDK the same way storage
already is.

## Current in-repo couplings that must be cleaned up (FunOS side)
Today, multiple FunOS files reach past the public SDK header and pull
**internal** XRDMA headers. These need to be reworked before XRDMA can
leave the tree.

### 1. PCB embeds an XRDMA struct directly
- `networking/rdma/rdma_internal.h` holds `struct xrdma_dp_socket *` with
  full struct visibility.
- `networking/rdma/rdma.c` calls `xrdma_init_state_push` /
  `xrdma_fini_state_push` directly and does `fun_calloc(sizeof(struct
  xrdma_dp_socket), ...)`.

**Fix**: Make `struct xrdma_dp_socket` opaque in FunOS. XRDMA registers its
ULP ops (init/fini push, dp-socket alloc/free, size) at module init through
a new SDK registration API.

### 2. Common RDMA CPI code uses XRDMA internal types
- `networking/rdma/ulp/common/rdma_cpi_dpi_internal.h` embeds `struct
  xrdma_conn_params` and includes the SDK xrdma.h.
- `networking/rdma/ulp/common/rdma_cpi_dpi_wuh.h` declares XRDMA
  channel/WU handlers.
- `services/rdma_cm/ulp/common/rdma_cpi.c` includes 5 XRDMA internal
  headers and reads `xrdma_dp_socket` / `socket_config->u.xrdma` fields.

**Fix**: Promote the minimum required types (`xrdma_conn_params`, a small
set of accessors for `sq_depth`/`rq_depth`/dp-socket response flow,
channel/WU handler declarations) into new SDK headers. Drop the internal
includes.

### 3. Props bridges dump XRDMA internals as JSON
- `props_bridges/sock_bridge.h`, `dpsock_bridge.c`, `cpsock_bridge.c`
  reach into `xrdma_ring`, `xrdma_credits`, `xrdma_operation`,
  `xrdma_dp_socket`, `xrdma_conn_params`, etc. to produce JSON.

**Fix**: Move the XRDMA-specific JSON generators into Storage-xDPU-XStore.
Props-bridges keeps the framework wiring (directory registration, context
plumbing) and invokes XRDMA-owned generators through function pointers
registered at init.

### 4. roce_json references XRDMA query tables and sizes
- `networking/rdma/roce/roce_json.c` references
  `xrdma_query_ops_table`, `xrdma_query_recv_cmpls_table`,
  `xrdma_query_recv_rsps_table`, `xrdma_query_frmr_ddr_table`,
  `xrdma_query_frmr_hsm_table`, `xrdma_query_rqbufs_ddr_table`,
  `xrdma_query_rqbufs_hsm_table`, `xrdma_query_ring_bufs_ddr_table`,
  `xrdma_query_ring_bufs_hsm_table`, and `sizeof(struct xrdma_operation)`.

**Fix**: XRDMA registers these query tables via a new SDK registration API;
`roce_json.c` invokes them through the registered pointers.

### 5. Global stats and FTR
- `utils/common/pervp_stats.h` hosts ~20 `xrdma_*` per-VP counter entries
  in a global X-macro.
- `networking/common/ftr_users.h` has an `"xrdma"` FTR tag.

**Fix**: Move XRDMA-specific stats and FTR registration into XRDMA,
registered dynamically at init. Requires a small extension to the per-VP
stats and FTR frameworks to accept dynamic registrations; fallback is to
leave these thin static entries in FunOS (minimal coupling).

### 6. Build system (FunOS)
- `networking/rdma/build.mk` compiles 14 XRDMA sources (`ulp/xrdma/*.c`).
- `services/rdma_cm/build.mk` compiles `ulp/xrdma/xrdma_cm.c`.
- `apps/build.mk` compiles XRDMA apps.
- `tests/build.mk` compiles `xrdma_test.c`.

**Fix**: Remove all of the above SRC_FILES entries. XRDMA is now compiled
into `libfunstorage.a` in Storage-xDPU-XStore and linked into FunOS via
FunSDK — the same way existing storage code is already linked.

## New SDK surface (proposed)
Added under `sdk_include/FunOS/networking/rdma/ulp/xrdma/`:
- `xrdma.h` *(exists)* — primary public API.
- `xrdma_ulp_ops.h` — ULP ops registration (init/fini push, DP socket
  alloc/free).
- `xrdma_cpi.h` — `struct xrdma_conn_params` + the accessors used by
  `rdma_cpi.c`.
- `xrdma_bridge.h` — registration hooks for props-bridge JSON generators.
- `xrdma_query.h` — registration hooks for `roce_json` query tables.
- `xrdma_wuh.h` — channel/WU handler prototypes referenced from
  ULP-common code.

## Work breakdown (execution order)

### Phase 1: Decouple in FunOS (no code moves yet)
1. **Define SDK surface.** Draft the new SDK registration headers listed
   above.
2. **Opaque PCB.** Make `xrdma_dp_socket` opaque in FunOS, move
   alloc/free and ULP ops to registration.
3. **Rework RDMA CPI.** Drop internal includes in `rdma_cpi.c` and
   `rdma_cpi_dpi_*.h`; use the new SDK accessors.
4. **Bridges.** Relocate XRDMA JSON generators behind function-pointer
   registration.
5. **roce_json.** Switch to registered query tables.
6. **Stats & FTR.** Move or dynamically register.

### Phase 2: Move code to Storage-xDPU-XStore
7. **Add `src/xrdma/` directory** with `build.mk`; add `xrdma` to
   `SUB_DIRS` in `src/build.mk`.
8. **Copy XRDMA core** (27 files) into `src/xrdma/core/`.
9. **Copy xrdma_cm.c** into `src/xrdma/cm/`.
10. **Copy XRDMA JSON generators** into `src/xrdma/bridges/`.
11. **Copy XRDMA apps & tests** into `src/storage_apps/`.

### Phase 3: Remove from FunOS and validate
12. **Remove XRDMA source** from FunOS tree.
13. **Update 4 build.mk files** in FunOS (remove SRC_FILES).
14. **Validate FunOS build** (POSIX, QEMU/s2) with XRDMA coming from
    `libfunstorage.a`.
15. **Validate Storage-xDPU-XStore build** compiles cleanly with XRDMA.
16. **Run full RDMA test suite** and smoke-test XRDMA on QEMU / FoD.

## Risks and open questions
- **Per-VP stats / FTR dynamic registration** — current frameworks use
  fixed X-macros. Either extend the frameworks or keep these small static
  entries in FunOS as an accepted residual coupling.
- **WU / channel handler linkage** — handler names are referenced in
  FunOS common code but defined in `libfunstorage.a`; verify linker
  resolution or switch to dynamic handler registration with the WU
  framework.
- **rdma_cm ULP dispatch** — confirm `rdma_cm` supports runtime ULP
  registration; if not, add a small hook.
- **Test ownership** — generic RDMA tests stay in FunOS; XRDMA-only tests
  move. Split points in `rdma_test.c` / `rdma_cpi_test.c` need review
  with the test owner.
- **FunSDK version gating** — Storage-xDPU-XStore already gates features
  on `FUNSDK_VERSION` / `FUNSDK_VERSION_MINOR`; new XRDMA SDK headers
  will need the same gating if they add new APIs.
- **Include paths** — XRDMA source currently uses `#include
  <networking/rdma/ulp/xrdma/...>` (FunOS-internal paths). After the
  move, these become local includes within Storage-xDPU-XStore; update
  include paths or add `-I` flags in the new `build.mk`.

## Out of scope
- Refactoring XRDMA internals beyond what's needed for the split.
- Changes to other RDMA ULPs (SMBD, native) except where they share
  `rdma_cpi_dpi_*` headers affected by the split.
- Performance or functional changes to XRDMA behavior.
- Changes to the existing XStore/csnetlib code that already consumes
  XRDMA through the SDK header.
