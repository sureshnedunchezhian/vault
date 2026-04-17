# SDK Promotion Proposal for XRDMA Extraction

## Goal
Minimal SDK promotion — expose only what XRDMA needs, no internal headers
leaked. No file with `internal` in its name goes into SDK.

**Excluded** (handled in storage repo): `pervp_stats.h`, `sock_bridge.h`.

---

## Proposal: 3 new SDK headers + additions to 1 existing header

### New: `sdk_include/FunOS/networking/rdma/ulp/rdma_dpi.h`

Consolidates the 3 ULP-common data-plane infrastructure APIs that both
XRDMA and SMBD use. These are clean abstractions, not internals.

**Source**: cherry-pick from `networking/rdma/ulp/common/rdma_dpi_buf.h`,
`rdma_dpi_frmr.h`, `rdma_dpi_res_mgr.h`.

| What | Why |
|------|-----|
| `struct rdma_dpi_buf`, `struct rdma_dpi_bufs_table` | Buffer alloc/free for ULP data plane |
| `rdma_dpi_get_buf_item()`, `rdma_dpi_put_buf_item()`, `rdma_dpi_init_buf_free_list()`, `rdma_dpi_get_buf_noblock()`, `rdma_dpi_free_buf()`, `rdma_dpi_free_bufs_table_push()`, `rdma_dpi_query_bufs_table()` | Buffer management API |
| `struct rdma_dpi_frmr_table`, `struct rdma_dpi_create_frmrs_state` | FRMR table management |
| `rdma_dpi_get_frmr_item()`, `rdma_dpi_put_frmr_item()`, `rdma_dpi_frmr_tab_from_restab()`, `rdma_dpi_init_frmr_free_list()`, `rdma_dpi_init_frmr_res_items_push()`, `rdma_dpi_init_frmr_res_items_ext_mem_push()` | FRMR alloc/free API |
| `enum rdma_dpi_res_types` (7 values), `enum rdma_dpi_res_mem_types` (4 values) | Resource type enums |
| `struct rdma_dpi_res_table`, `struct rdma_dpi_res_table_counters`, `struct rdma_dpi_res_table_query_ovw` | Resource table types |
| `rdma_dpi_alloc_res_table_push()`, `rdma_dpi_free_res_table_push()`, `rdma_dpi_get_res_item()`, `rdma_dpi_get_res_item_nf()`, `rdma_dpi_free_res_item()`, `rdma_dpi_res_fid()`, `rdma_dpi_query_restab()`, `rdma_dpi_inc_res_in_use()` | Resource management API |

---

### New: `sdk_include/FunOS/networking/rdma/ulp/rdma_cpi.h`

Consolidates ULP-common control-plane types and socket state machine.

**Source**: cherry-pick from `networking/rdma/ulp/common/rdma_cpi_dpi_internal.h`
and `services/rdma_cm/ulp/common/rdma_cpi_states.h`.

| What | Why |
|------|-----|
| `enum rdma_cpi_socket_states` — 7 values (`_INIT`, `_CONNECTING`, `_CONNECTED`, `_DISC_NOTIFIED`, `_DISCONNECTING`, `_DISCONNECTED`, `_MAX`) | Socket state machine — 59 usages across 4 xrdma files, also used by SMBD |
| `struct rdma_cpi_socket` | Control-plane socket — xrdma accesses fields: `app_cmf`, `client`, `dp_sock`, `funxrdma_params`, `pd`, `socket_config`, `socket_id`, `state`, `vp` |
| `struct rdma_cpi_listen_socket` | Listen socket type |
| `struct rdma_cpi_mem_descriptor` | Memory descriptor for CPI |
| `struct xrdma_conn_params` | Connection params (xrdma-specific but embedded in CPI socket) |
| `struct rdma_dpi_frmr` | FRMR descriptor (defined in cpi_dpi_internal.h) |
| `struct rdma_dpi_buf_registration`, `struct rdma_dpi_frmr_bufreg` | Buffer registration types |
| `typedef enum socket_type_e` | Socket type enum |
| Trace/log macros: `RDMA_ULP_TRACE`, `RDMA_CPI_TRACE`, `RDMA_DPI_TRACE` | Per-socket tracing |
| Constants: `RDMA_DPI_SOCK_FTR_POW2`, `RDMA_CPI_PAGE_SHIFT`, etc. | ULP config constants |

**Not included** from `rdma_cm_internal.h`: nothing — xrdma's only use of
that header was for socket state enums, which are covered by `rdma_cpi_states.h`
content above.

---

### New: `sdk_include/FunOS/networking/rdma/rdma_flow.h`

Clean SDK view of RDMA flow and MR types needed by ULPs. Replaces all
xrdma usage of `networking/rdma/rdma_internal.h`.

**Source**: cherry-pick minimal subset from `rdma_internal.h`.

| What | Why |
|------|-----|
| `struct rdma_mr` (64 bytes, HW layout) | MR entry — xrdma reads/writes `opcode_flags` via macros |
| `struct rdma_mr_meta` | MR metadata — xrdma accesses via `rdma_get_mr_meta()` |
| `enum roce_mr_state` — 3 values (`INVALID`, `FREE`, `VALID`) | MR state checks in `xrdma_frmr.c` |
| `RDMA_MR_STAG_TO_ID()`, `RDMA_GET_MR_STATE()`, `RDMA_SET_MR_STATE()`, `RDMA_SET_MR_FLAGS()` | MR state/flags macros — 15 usages in `xrdma_frmr.c` |
| `struct rdma_flow` | Flow object — xrdma accesses: `rdma_hton_flow`, `rdma_ntoh_flow`, `rdma_id`, `rflow`, `rflow.loc_qpn`, `rflow.qp_caps`, `ulp.cmpl_dispatch_cb`, `ulp.pcb` |
| `struct rdma_ulp_state`, `union rdma_ulp_pcb` | ULP state within flow — xrdma sets `cmpl_dispatch_cb`, reads `pcb.xrdma_dp_sock` |
| `rdma_from_hton_flow()` | Flow lookup from hton flow — 14 call sites |
| `rdma_from_ntoh_flow()` | Flow lookup from ntoh flow — 1 call site |
| `mrinfo_from_id()` | MR lookup by ID — 4 call sites |
| `rdma_get_mr_meta()` | MR metadata lookup — 1 call site |
| `roce_flow_color_chk()` | Flow color validation — 7 call sites (from `roce_cmpl_wu_v2.h`) |
| `roce_mr_update_entry()` | MR page table update — 1 call site (from `roce_mr.h`) |

**Not included**: `struct rdma` (global state), `struct rdma_mr_tab`,
`struct rdma_pd_tab`, config tables, init routines, DCQCN internals,
rnic manager, iwarp internals — all stay in FunOS.

**Note**: `struct rdma_flow` depends on `struct flow` (already in SDK),
`struct roce_flow`/`struct iwarp_flow` (need to verify these are in SDK
or add forward declarations), `struct rdma_ulp_state` (included here).

---

### Addition to existing: `sdk_include/FunOS/networking/rdma/rdma.h`

Add one constant currently in `hw/dam/dam.h`:

| What | Why |
|------|-----|
| `DAM_MAX_REFERENCES_PER_INDEX` (= 255) | Used in `xrdma_dam.c` for refcount validation — 4 call sites |

**Alternative**: hardcode `255` in `xrdma_dam.c` with a `static_assert`
against the real define (when building in-tree). Avoids polluting the
RDMA SDK header with a DAM constant. **Recommend this alternative.**

---

## Headers that DON'T need promotion

### Drop unused includes (clean up in xrdma before move)

| Header | Included by | Reason to drop |
|--------|-------------|----------------|
| `hw/ddr_buf/ddr_buf.h` | `xrdma_dam.c` | Zero symbols used — nvmem abstraction used instead |
| `hw/hbm_buf/hbm_buf.h` | `xrdma_dam.c` | Zero symbols used — same |
| `hw/rnic/hw_rnic.h` | `xrdma_operation.c` | Zero symbols used |

### Replace include (no promotion needed)

| Header | Action |
|--------|--------|
| `services/rdma_cm/rdma_cm_internal.h` | Replace with new `rdma_cpi.h` — xrdma only used socket state enums |

### Split xrdma-specific content out

| Header | Action |
|--------|--------|
| `networking/rdma/ulp/common/rdma_cpi_dpi_wuh.h` | Remove `xrdma_*` and `smbd_*` handler declarations from this file. Move xrdma handler decls into xrdma's own header (moves with xrdma). Keep only generic `rdma_cpi_*` handler decls. **No SDK promotion needed** — xrdma defines these handlers, so their declarations move with it. |

---

## Summary

| Action | Count | Details |
|--------|-------|---------|
| **New SDK headers** | 3 | `rdma_dpi.h`, `rdma_cpi.h`, `rdma_flow.h` |
| **Drop unused includes** | 3 | `ddr_buf.h`, `hbm_buf.h`, `hw_rnic.h` |
| **Replace include** | 1 | `rdma_cm_internal.h` → `rdma_cpi.h` |
| **Split handler decls** | 1 | `rdma_cpi_dpi_wuh.h` — remove xrdma/smbd entries |
| **Hardcode constant** | 1 | `DAM_MAX_REFERENCES_PER_INDEX` in `xrdma_dam.c` |
| **No change** | 0 | No `*_internal.h` in SDK |

**Total new SDK surface**: 3 clean, purpose-named headers. Zero internal
headers exposed.
