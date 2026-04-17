# XRDMA Extraction: Scope & SDK Promotion Plan

## Summary
The XRDMA ULP (Upper Layer Protocol) currently lives inside the FunOS source
tree. We want to relocate it into the **Storage-xDPU-XStore** repo
(`msazure.visualstudio.com/One/_git/Storage-xDPU-XStore`), which already
hosts XStore/csnetlib storage code and is the primary consumer of the XRDMA
API.

### Build & dependency flow
```
FunOS ──builds──► FunSDK (ships RDMA SDK headers: rdma.h, rdma_ulp.h, rdma_cm.h, …)
                      │
                      ▼
           Storage-xDPU-XStore (consumes FunSDK)
             ├── xrdma code + xrdma.h (moved here)
             ├── xstore / csnetlib (existing consumer)
             └── builds libfunstorage.a / funos-storage image
```

After the move:
- **FunOS has zero XRDMA headers or source.** The `sdk_include/.../xrdma/xrdma.h`
  header also moves out.
- XRDMA source compiles in Storage-xDPU-XStore, consuming the RDMA SDK
  headers delivered via FunSDK.
- XStore/csnetlib uses XRDMA via repo-local includes (no SDK round-trip).

## Why
- **Ownership transfer**: The XRDMA module ownership is moving to the
  Storage/XStore team. The code should live in the repo they own and
  develop in (Storage-xDPU-XStore).
- Decouple XRDMA lifecycle (development, review, release) from FunOS.
- Enforce a stable API boundary: today several FunOS files reach directly
  into XRDMA internals, which blocks independent evolution of the ULP.
- Co-locate XRDMA with its primary consumer (XStore/csnetlib), which already
  lives in Storage-xDPU-XStore and uses XRDMA exclusively through its public
  header.

## Current footprint in FunOS
| Area | Location | Size |
|---|---|---|
| XRDMA core | `networking/rdma/ulp/xrdma/` | 27 files, ~9.3K LOC |
| XRDMA CM glue | `services/rdma_cm/ulp/xrdma/xrdma_cm.c` | ~740 LOC |
| XRDMA apps | `apps/rdma/xrdma_ping.c`, `apps/rdma/xrdma_test_utils.{c,h}` | ~2.8K LOC |
| XRDMA tests | `tests/xrdma_test.c` + xrdma branches in `tests/rdma_test.c`, `tests/rdma_cpi_test.c` | ~6K LOC |
| Public API (also moves out) | `sdk_include/FunOS/networking/rdma/ulp/xrdma/xrdma.h` | — |

## Storage-xDPU-XStore repo (destination)
The repo already:
- Consumes FunSDK (RDMA SDK headers, funosrt, funcrypto, etc.) to build
  `libfunstorage-$(MACHINE).a` and the `funos-storage` image.
- Has `src/build.mk` with `SUB_DIRS := storage storage_apps kv`.
- Uses `<FunOS/networking/rdma/ulp/xrdma/xrdma.h>` (SDK header only) in
  `src/storage/xstore/csnetlib/csnetlib_xrl*.{c,h}` — no internal headers.
- Has its own CI pipeline (`OneBranch.Official.FunStorage.yml`), test
  apps (`src/storage_apps/`), and FoD automation (`src/fod_automation/`).

After the move, XRDMA code lands as a new `SUB_DIRS` entry (`src/xrdma/`),
and `xrdma.h` becomes a repo-local header. XStore/csnetlib switches from
the FunSDK-delivered xrdma.h to the local copy.

## What moves out of FunOS
1. All 27 files under `networking/rdma/ulp/xrdma/`.
2. `sdk_include/FunOS/networking/rdma/ulp/xrdma/xrdma.h` (public API header).
3. `services/rdma_cm/ulp/xrdma/xrdma_cm.c` (XRDMA-specific CM dispatch).
4. `apps/rdma/xrdma_ping.c`, `apps/rdma/xrdma_test_utils.{c,h}`.
5. `tests/xrdma_test.c`, plus the xrdma-only branches split out of
   `tests/rdma_test.c` and `tests/rdma_cpi_test.c`.
6. XRDMA-specific JSON dump helpers currently hosted in
   `props_bridges/dpsock_bridge.c` and `props_bridges/cpsock_bridge.c`.

## Proposed layout in Storage-xDPU-XStore
```
src/
  xrdma/                        # NEW — added to SUB_DIRS
    build.mk                    # SRC_FILES for XRDMA core
    include/xrdma.h             # ex-sdk_include/.../xrdma/xrdma.h (public API)
    core/                       # ex-networking/rdma/ulp/xrdma/*
    cm/                         # ex-services/rdma_cm/ulp/xrdma/xrdma_cm.c
    bridges/                    # ex-props_bridges/* XRDMA JSON generators
  storage_apps/
    xrdma_ping.c                # ex-apps/rdma/xrdma_ping.c
    xrdma_test_utils.{c,h}      # ex-apps/rdma/xrdma_test_utils.*
    xrdma_test.c                # ex-tests/xrdma_test.c
```
XRDMA objects compile into `libfunstorage.a` alongside existing storage code.
XRDMA consumes RDMA SDK headers (`rdma.h`, `rdma_ulp.h`, `rdma_cm.h`) from
FunSDK. XStore/csnetlib switches to the repo-local xrdma.h include.

---

## FunOS-side couplings to clean up

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

**Fix**: Promote the minimum required types into new SDK headers (see SDK
promotion section below). Drop the internal includes.

### 3. Props bridges dump XRDMA internals as JSON
- `props_bridges/sock_bridge.h`, `dpsock_bridge.c`, `cpsock_bridge.c`
  reach into `xrdma_ring`, `xrdma_credits`, `xrdma_operation`,
  `xrdma_dp_socket`, `xrdma_conn_params`, etc. to produce JSON.

**Fix**: Move the XRDMA-specific JSON generators into Storage-xDPU-XStore.
Props-bridges keeps the framework wiring and invokes XRDMA-owned generators
through function pointers registered at init.

### 4. roce_json references XRDMA query tables and sizes
- `networking/rdma/roce/roce_json.c` references `xrdma_query_*` tables
  and `sizeof(struct xrdma_operation)`.

**Fix**: XRDMA registers these query tables via a new SDK registration API;
`roce_json.c` invokes them through registered pointers.

### 5. Global stats and FTR
- `utils/common/pervp_stats.h` hosts ~20 `xrdma_*` per-VP counter entries.
- `networking/common/ftr_users.h` has an `"xrdma"` FTR tag.

**Fix**: Handled in Storage-xDPU-XStore side. `pervp_stats.h` and
`ftr_users.h` are not promoted — xrdma manages its own stats/FTR after
the move.

### 6. Build system (FunOS)
- `networking/rdma/build.mk` compiles 14 XRDMA sources.
- `services/rdma_cm/build.mk` compiles `xrdma_cm.c`.
- `apps/build.mk` compiles XRDMA apps.
- `tests/build.mk` compiles `xrdma_test.c`.

**Fix**: Remove all SRC_FILES entries and `sdk_include/.../xrdma/xrdma.h`.

---

## SDK promotion: what XRDMA needs from FunOS

All `<FunOS/...>` SDK headers (flows, nucleus, platform, utils, services,
FunHCI) are already in FunSDK — no work needed for those.

**FTR**: `ftr.h` is already in FunSDK. ~24 call sites. No promotion needed.

**pervp_stats / props bridges**: handled in Storage-xDPU-XStore. Not
promoted.

The remaining FunOS-internal headers used by XRDMA are resolved as follows.
**No `*_internal.h` file goes into SDK.**

### New SDK header: `sdk_include/FunOS/networking/rdma/ulp/rdma_dpi.h`

Consolidates the 3 ULP-common data-plane infrastructure APIs used by both
XRDMA and SMBD. Clean abstractions, not internals.

**Replaces**: `rdma_dpi_buf.h`, `rdma_dpi_frmr.h`, `rdma_dpi_res_mgr.h`.

| What | Why |
|------|-----|
| `struct rdma_dpi_buf`, `struct rdma_dpi_bufs_table` | Buffer alloc/free for ULP data plane |
| `rdma_dpi_get_buf_item()`, `rdma_dpi_put_buf_item()`, `rdma_dpi_init_buf_free_list()`, `rdma_dpi_get_buf_noblock()`, `rdma_dpi_free_buf()`, `rdma_dpi_free_bufs_table_push()`, `rdma_dpi_query_bufs_table()` | Buffer management API |
| `struct rdma_dpi_frmr_table`, `struct rdma_dpi_create_frmrs_state` | FRMR table management |
| `rdma_dpi_get_frmr_item()`, `rdma_dpi_put_frmr_item()`, `rdma_dpi_frmr_tab_from_restab()`, `rdma_dpi_init_frmr_free_list()`, `rdma_dpi_init_frmr_res_items_push()`, `rdma_dpi_init_frmr_res_items_ext_mem_push()` | FRMR alloc/free API |
| `enum rdma_dpi_res_types` (7 values), `enum rdma_dpi_res_mem_types` (4 values) | Resource type enums |
| `struct rdma_dpi_res_table`, `struct rdma_dpi_res_table_counters`, `struct rdma_dpi_res_table_query_ovw` | Resource table types |
| `rdma_dpi_alloc_res_table_push()`, `rdma_dpi_free_res_table_push()`, `rdma_dpi_get_res_item()`, `rdma_dpi_get_res_item_nf()`, `rdma_dpi_free_res_item()`, `rdma_dpi_res_fid()`, `rdma_dpi_query_restab()`, `rdma_dpi_inc_res_in_use()` | Resource management API |

### New SDK header: `sdk_include/FunOS/networking/rdma/ulp/rdma_cpi.h`

Consolidates ULP-common control-plane types and socket state machine.

**Replaces**: `rdma_cpi_dpi_internal.h`, `rdma_cpi_states.h`,
`rdma_cm_internal.h` (xrdma only used socket state enums from it).

| What | Why |
|------|-----|
| `enum rdma_cpi_socket_states` — 7 values | Socket state machine — 59 usages across 4 xrdma files, also used by SMBD |
| `struct rdma_cpi_socket` | Control-plane socket — fields: `app_cmf`, `client`, `dp_sock`, `funxrdma_params`, `pd`, `socket_config`, `socket_id`, `state`, `vp` |
| `struct rdma_cpi_listen_socket` | Listen socket type |
| `struct rdma_cpi_mem_descriptor` | Memory descriptor for CPI |
| `struct xrdma_conn_params` | Connection params (xrdma-specific but embedded in CPI socket) |
| `struct rdma_dpi_frmr` | FRMR descriptor |
| `struct rdma_dpi_buf_registration`, `struct rdma_dpi_frmr_bufreg` | Buffer registration types |
| `typedef enum socket_type_e` | Socket type enum |
| Trace/log macros: `RDMA_ULP_TRACE`, `RDMA_CPI_TRACE`, `RDMA_DPI_TRACE` | Per-socket tracing |
| Constants: `RDMA_DPI_SOCK_FTR_POW2`, `RDMA_CPI_PAGE_SHIFT`, etc. | ULP config constants |

### New SDK header: `sdk_include/FunOS/networking/rdma/rdma_flow.h`

Clean SDK view of RDMA flow and MR types needed by ULPs.

**Replaces**: `rdma_internal.h`, `roce_cmpl_wu_v2.h`, `roce_mr.h`.

| What | Why |
|------|-----|
| `struct rdma_flow` | Flow object — xrdma accesses: `rdma_hton_flow`, `rdma_ntoh_flow`, `rdma_id`, `rflow`, `ulp.cmpl_dispatch_cb`, `ulp.pcb` |
| `struct rdma_ulp_state`, `union rdma_ulp_pcb` | ULP state within flow |
| `struct rdma_mr` (64 bytes, HW layout) | MR entry — xrdma reads/writes `opcode_flags` via macros |
| `struct rdma_mr_meta` | MR metadata |
| `enum roce_mr_state` — 3 values | MR state checks in `xrdma_frmr.c` |
| `RDMA_MR_STAG_TO_ID()`, `RDMA_GET_MR_STATE()`, `RDMA_SET_MR_STATE()`, `RDMA_SET_MR_FLAGS()` | MR state/flags macros — 15 usages |
| `rdma_from_hton_flow()` | Flow lookup — 14 call sites |
| `rdma_from_ntoh_flow()` | Flow lookup — 1 call site |
| `mrinfo_from_id()` | MR lookup — 4 call sites |
| `rdma_get_mr_meta()` | MR metadata lookup — 1 call site |
| `roce_flow_color_chk()` | Flow color validation — 7 call sites |
| `roce_mr_update_entry()` | MR page table update — 1 call site |

**Not included**: `struct rdma` (global state), `struct rdma_mr_tab`,
config tables, init routines, DCQCN, rnic manager, iwarp internals.

### Headers that DON'T need promotion

| Header | Action |
|--------|--------|
| `hw/ddr_buf/ddr_buf.h` | **Drop include** — zero symbols used |
| `hw/hbm_buf/hbm_buf.h` | **Drop include** — zero symbols used |
| `hw/rnic/hw_rnic.h` | **Drop include** — zero symbols used |
| `hw/dam/dam.h` | **Hardcode** `DAM_MAX_REFERENCES_PER_INDEX` (255) in `xrdma_dam.c` |
| `services/rdma_cm/rdma_cm_internal.h` | **Replace** with new `rdma_cpi.h` |
| `networking/rdma/ulp/common/rdma_cpi_dpi_wuh.h` | **Split** — move `xrdma_*`/`smbd_*` handler decls into their own headers; keep only generic `rdma_cpi_*` decls in FunOS. No SDK promotion needed. |
| `utils/common/pervp_stats.h` | Handled in Storage-xDPU-XStore |
| `props_bridges/sock_bridge.h` | Handled in Storage-xDPU-XStore |

### SDK promotion summary

| Action | Count | Details |
|--------|-------|---------|
| **New SDK headers** | 3 | `rdma_dpi.h`, `rdma_cpi.h`, `rdma_flow.h` |
| **Drop unused includes** | 3 | `ddr_buf.h`, `hbm_buf.h`, `hw_rnic.h` |
| **Replace include** | 1 | `rdma_cm_internal.h` → `rdma_cpi.h` |
| **Split handler decls** | 1 | `rdma_cpi_dpi_wuh.h` — remove xrdma/smbd entries |
| **Hardcode constant** | 1 | `DAM_MAX_REFERENCES_PER_INDEX` in `xrdma_dam.c` |
| **`*_internal.h` in SDK** | 0 | None |

---

## Work breakdown (execution order)

### Phase 1: Decouple in FunOS (no code moves yet)
1. **Create 3 SDK headers.** `rdma_dpi.h`, `rdma_cpi.h`, `rdma_flow.h` in
   `sdk_include/`.
2. **Clean up xrdma includes.** Drop 3 unused includes, replace
   `rdma_cm_internal.h`, hardcode DAM constant.
3. **Split `rdma_cpi_dpi_wuh.h`.** Move xrdma/smbd handler decls out.
4. **Opaque PCB.** Make `xrdma_dp_socket` opaque in FunOS, register ULP ops.
5. **Rework RDMA CPI.** Drop internal includes in `rdma_cpi.c` and
   `rdma_cpi_dpi_*.h`; use the new SDK headers.
6. **Bridges.** Relocate XRDMA JSON generators behind function-pointer
   registration.
7. **roce_json.** Switch to registered query tables.

### Phase 2: Move code to Storage-xDPU-XStore
8. **Add `src/xrdma/` directory** with `build.mk` and `include/xrdma.h`;
   add `xrdma` to `SUB_DIRS` in `src/build.mk`.
9. **Copy XRDMA core** (27 files) into `src/xrdma/core/`.
10. **Copy xrdma_cm.c** into `src/xrdma/cm/`.
11. **Copy XRDMA JSON generators** into `src/xrdma/bridges/`.
12. **Copy XRDMA apps & tests** into `src/storage_apps/`.
13. **Update XStore/csnetlib includes** to use repo-local xrdma.h.

### Phase 3: Remove from FunOS and validate
14. **Remove XRDMA source and xrdma.h** from FunOS tree.
15. **Update 4 build.mk files** in FunOS (remove SRC_FILES).
16. **Validate FunOS build** (POSIX, QEMU/s2) — no XRDMA references remain.
17. **Validate Storage-xDPU-XStore build** compiles cleanly with XRDMA
    using RDMA SDK headers from FunSDK.
18. **Run full RDMA test suite** and smoke-test XRDMA on QEMU / FoD.

## Risks and open questions
- **WU / channel handler linkage** — handler names are referenced in
  FunOS common code but defined in `libfunstorage.a`; verify linker
  resolution or switch to dynamic handler registration.
- **rdma_cm ULP dispatch** — confirm `rdma_cm` supports runtime ULP
  registration; if not, add a small hook.
- **Test ownership** — generic RDMA tests stay in FunOS; XRDMA-only tests
  move. Split points in `rdma_test.c` / `rdma_cpi_test.c` need review
  with the test owner.
- **FunSDK version gating** — the new SDK headers will need version gating
  in Storage-xDPU-XStore until the minimum SDK version includes them.
- **Include paths** — XRDMA source currently uses `#include
  <networking/rdma/ulp/xrdma/...>` (FunOS-internal paths). After the
  move, update to local includes or add `-I` flags.
- **xrdma.h removal from FunSDK** — verify no other repos depend on it
  besides Storage-xDPU-XStore (which switches to repo-local copy).
- **`struct rdma_flow` dependencies** — `struct roce_flow` / `struct
  iwarp_flow` must be available in SDK or forward-declared in
  `rdma_flow.h`.

## Out of scope
- Refactoring XRDMA internals beyond what's needed for the split.
- Changes to other RDMA ULPs (SMBD, native) except where they share
  headers affected by the split.
- Performance or functional changes to XRDMA behavior.
- Changes to existing XStore/csnetlib code that already consumes XRDMA
  through its public header.
