# SDK Promotion Proposal for XRDMA Extraction

## Context
XRDMA code in `networking/rdma/ulp/xrdma/` uses 14 FunOS-internal headers
that are not in FunSDK today. For XRDMA to compile in Storage-xDPU-XStore,
these dependencies must be resolved. The goal is **minimal promotion** —
only expose what XRDMA actually needs, no internal leaks.

**Excluded from this analysis** (handled separately in storage repo):
`utils/common/pervp_stats.h`, `props_bridges/sock_bridge.h`.

---

## Category 1: Headers to promote to SDK (as-is or near as-is)

These are shared ULP-common infrastructure headers used by **both** XRDMA
and SMBD. They are stable abstractions, not internal implementation details.

### 1.1 `networking/rdma/ulp/common/rdma_dpi_buf.h` → **Promote**
- Used by: `xrdma_buf.c`, `xrdma_buf.h`, `xrdma_frmr.c`, `xrdma_recv.c`, `xrdma_send.c`
- Also used by: SMBD (`smbd_fp.c`, `smbd_recv.c`, etc.)
- Symbols: `struct rdma_dpi_buf`, `struct rdma_dpi_bufs_table`,
  `rdma_dpi_get_buf_item()`, `rdma_dpi_put_buf_item()`,
  `rdma_dpi_init_buf_free_list()`, `rdma_dpi_get_buf_noblock()`,
  `rdma_dpi_free_buf()`, `rdma_dpi_free_bufs_table_push()`,
  `rdma_dpi_query_bufs_table()`
- **Rationale**: Well-defined buffer management API for ULPs. Stable.

### 1.2 `networking/rdma/ulp/common/rdma_dpi_frmr.h` → **Promote**
- Used by: `xrdma_frmr.c`, `xrdma_frmr.h`
- Also used by: SMBD (`smbd_frmr.c`, `smbd_frmr.h`)
- Symbols: `struct rdma_dpi_frmr_table`, `struct rdma_dpi_create_frmrs_state`,
  `rdma_dpi_get_frmr_item()`, `rdma_dpi_put_frmr_item()`,
  `rdma_dpi_frmr_tab_from_restab()`, `rdma_dpi_init_frmr_free_list()`,
  `rdma_dpi_init_frmr_res_items_push()`,
  `rdma_dpi_init_frmr_res_items_ext_mem_push()`
- **Rationale**: FRMR (Fast Registration Memory Region) API shared by ULPs. Stable.

### 1.3 `networking/rdma/ulp/common/rdma_dpi_res_mgr.h` → **Promote**
- Used by: `xrdma_operation.c/h`, `xrdma_recv.c/h`, `xrdma_frmr.c`, `xrdma_buf.c`
- Also used by: SMBD (`smbd_operation.c/h`, `smbd_recv.c`, etc.)
- Symbols: `enum rdma_dpi_res_types`, `enum rdma_dpi_res_mem_types`,
  `struct rdma_dpi_res_table`, `struct rdma_dpi_res_table_counters`,
  `struct rdma_dpi_res_table_query_ovw`,
  `rdma_dpi_alloc_res_table_push()`, `rdma_dpi_free_res_table_push()`,
  `rdma_dpi_get_res_item()`, `rdma_dpi_get_res_item_nf()`,
  `rdma_dpi_free_res_item()`, `rdma_dpi_res_fid()`,
  `rdma_dpi_query_restab()`, `rdma_dpi_inc_res_in_use()`
- **Rationale**: Generic ULP resource management. Stable.

### 1.4 `services/rdma_cm/ulp/common/rdma_cpi_states.h` → **Promote**
- Used by: `xrdma_internal.h` → propagates to 12 xrdma files
- Also used by: SMBD (`smbd_internal.h`), `rdma_cpi_dpi_internal.h`
- Symbols: `enum rdma_cpi_socket_states` — 7 values
  (`RDMA_CPI_SOCKET_STATE_INIT`, `_CONNECTING`, `_CONNECTED`,
  `_DISC_NOTIFIED`, `_DISCONNECTING`, `_DISCONNECTED`, `_MAX`)
- **Rationale**: Simple enum, used pervasively for socket state machine.
  59 usages across 4 xrdma files. Already a de-facto public API.

### 1.5 `networking/rdma/ulp/common/rdma_cpi_dpi_internal.h` → **Promote**
- Used by: `xrdma.c`, `xrdma_internal.h`, `xrdma_send.h`
- Also used by: SMBD, rdma_cpi.c, rdma_cpi_util.c, rdma_dpi.c
- Symbols: `struct rdma_cpi_socket`, `struct rdma_cpi_mem_descriptor`,
  `struct xrdma_conn_params` (xrdma-specific but embedded here),
  socket config union fields
- **Rationale**: Core CPI/DPI shared types needed by any ULP.
- **Note**: Contains `struct xrdma_conn_params` — this xrdma-specific
  type should either stay (accepted coupling) or be split into a separate
  header that xrdma registers. Recommend: keep as-is for now since SMBD
  has its own `smbd_conn_params` in the same header.

---

## Category 2: Headers to promote with xrdma-specific content removed

### 2.1 `networking/rdma/ulp/common/rdma_cpi_dpi_wuh.h` → **Promote (split)**
- Used by: `xrdma.c`, `xrdma_operation.c`
- Also used by: SMBD, rdma_cpi.c
- Contains: ~30 generic `rdma_cpi_*` handler decls, ~7 `xrdma_*` handler
  decls, ~5 `smbd_*` handler decls
- **Proposal**: Promote with **only the generic `rdma_cpi_*` handlers**.
  Move the `xrdma_*` declarations into xrdma's own internal header.
  Move the `smbd_*` declarations into SMBD's own internal header.
  This is the cleanest split — the promoted header becomes purely
  ULP-agnostic infrastructure.

### 2.2 `networking/rdma/rdma_internal.h` → **Promote subset as new SDK header**
- Used by: 21 xrdma files (most widely used)
- Symbols actually needed by xrdma:
  - `rdma_from_hton_flow()` (14 uses) — flow lookup
  - `rdma_from_ntoh_flow()` (1 use)
  - `mrinfo_from_id()` (4 uses) — MR lookup
  - `struct rdma_flow` — flow object
  - `struct rdma_mr`, `struct rdma_mr_meta` — MR types
  - `RDMA_MR_STAG_TO_ID()`, `RDMA_GET_MR_STATE()`,
    `RDMA_SET_MR_STATE()`, `RDMA_SET_MR_FLAGS()` — MR macros
  - `enum roce_mr_state` values
- **Proposal**: Create a new **`sdk_include/FunOS/networking/rdma/rdma_ulp_internal.h`**
  that exposes only:
  - `struct rdma_flow` (opaque or minimal definition)
  - Flow lookup functions: `rdma_from_hton_flow()`, `rdma_from_ntoh_flow()`
  - `struct rdma_mr`, `struct rdma_mr_meta` (or opaque + accessors)
  - MR lookup: `mrinfo_from_id()`
  - MR state macros: `RDMA_MR_STAG_TO_ID()`, `RDMA_GET/SET_MR_STATE/FLAGS()`
  - `enum roce_mr_state`
- **Do NOT promote**: RDMA PCB internals, config tables, init routines,
  global state — those stay internal to FunOS.

### 2.3 `services/rdma_cm/rdma_cm_internal.h` → **Not needed directly**
- Used by: `xrdma.c` only
- But the **only symbols xrdma uses** are the `RDMA_CPI_SOCKET_STATE_*`
  enums, which come from `rdma_cpi_states.h` (promoted in 1.4 above).
- **Proposal**: Replace `#include <services/rdma_cm/rdma_cm_internal.h>`
  with `#include <services/rdma_cm/ulp/common/rdma_cpi_states.h>` (or
  the new SDK path) in xrdma.c. **No promotion needed.**

---

## Category 3: Promote single symbol (trivial)

### 3.1 `hw/dam/dam.h` → **Promote one macro**
- Used by: `xrdma_dam.c` only
- Only symbol used: `DAM_MAX_REFERENCES_PER_INDEX` (value: 255)
- **Proposal**: Add this single `#define` to an existing SDK header
  (e.g., `sdk_include/FunOS/networking/rdma/rdma.h` or a small
  `sdk_include/FunOS/hw/dam/dam_constants.h`). Or just hardcode 255
  in xrdma_dam.c with a compile-time assert.

---

## Category 4: Drop — unused includes

### 4.1 `hw/ddr_buf/ddr_buf.h` → **Remove include**
- Included by `xrdma_dam.c` but **zero symbols used**. Uses nvmem
  abstraction instead. Safe to delete the `#include`.

### 4.2 `hw/hbm_buf/hbm_buf.h` → **Remove include**
- Same as above — included but no symbols used.

### 4.3 `hw/rnic/hw_rnic.h` → **Remove include**
- Included by `xrdma_operation.c` but **zero symbols used**.

---

## Category 5: Single-function dependencies (small promotions)

### 5.1 `networking/rdma/roce/roce_cmpl_wu_v2.h` → **Promote one function**
- Used by: `xrdma.c` only
- Only symbol: `roce_flow_color_chk()` — 7 call sites for flow color
  validation
- **Proposal**: Add `roce_flow_color_chk()` declaration to a new or
  existing SDK header (e.g., `sdk_include/FunOS/networking/rdma/rdma.h`
  or new `rdma_roce.h`).

### 5.2 `networking/rdma/roce/roce_mr.h` → **Promote one function**
- Used by: `xrdma_medium.c` only
- Only symbol: `roce_mr_update_entry()` — 1 call site
- **Proposal**: Add `roce_mr_update_entry()` declaration alongside
  the MR types in the new `rdma_ulp_internal.h` (see 2.2).

---

## Summary table

| Header | Action | What gets promoted |
|--------|--------|--------------------|
| `rdma_dpi_buf.h` | Promote as-is | Full header (shared ULP infra) |
| `rdma_dpi_frmr.h` | Promote as-is | Full header (shared ULP infra) |
| `rdma_dpi_res_mgr.h` | Promote as-is | Full header (shared ULP infra) |
| `rdma_cpi_states.h` | Promote as-is | Socket state enum (7 values) |
| `rdma_cpi_dpi_internal.h` | Promote as-is | CPI/DPI shared types |
| `rdma_cpi_dpi_wuh.h` | Promote with split | Generic `rdma_cpi_*` handlers only; remove xrdma_*/smbd_* |
| `rdma_internal.h` | New SDK header | Subset: flow lookup, MR types/macros, MR lookup |
| `rdma_cm_internal.h` | **No promotion** | Replace include with `rdma_cpi_states.h` |
| `dam.h` | One `#define` | `DAM_MAX_REFERENCES_PER_INDEX` (or hardcode) |
| `ddr_buf.h` | **Drop include** | Unused |
| `hbm_buf.h` | **Drop include** | Unused |
| `hw_rnic.h` | **Drop include** | Unused |
| `roce_cmpl_wu_v2.h` | One function | `roce_flow_color_chk()` |
| `roce_mr.h` | One function | `roce_mr_update_entry()` |

**Total new SDK surface**: 5 headers promoted as-is + 1 new `rdma_ulp_internal.h`
(subset of `rdma_internal.h`) + 1 split (`rdma_cpi_dpi_wuh.h`) + 3 single-symbol
additions. 3 unused includes dropped. 1 include swapped.
