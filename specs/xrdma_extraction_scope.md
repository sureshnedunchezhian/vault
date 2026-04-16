# Scope: Extract XRDMA ULP from FunOS into a Standalone Repo

## Summary
The XRDMA ULP (Upper Layer Protocol) currently lives inside the FunOS source
tree. We want to relocate it to its own repository and deliver it to FunOS as
a **prebuilt library shipped via FunSDK**, similar to other SDK-delivered
libraries. FunOS will continue to consume XRDMA through the public SDK
header (`sdk_include/FunOS/networking/rdma/ulp/xrdma/xrdma.h`) only.

This document describes the full scope of work required to make that
separation clean.

## Why
- Decouple XRDMA lifecycle (development, review, release) from FunOS.
- Enforce a stable public API: today several FunOS files reach directly into
  XRDMA internals, which blocks independent evolution of the ULP.
- Align XRDMA with the existing pattern used by other SDK-delivered
  components.

## Current footprint
| Area | Location | Size |
|---|---|---|
| XRDMA core | `networking/rdma/ulp/xrdma/` | 27 files, ~9.3K LOC |
| XRDMA CM glue | `services/rdma_cm/ulp/xrdma/xrdma_cm.c` | ~740 LOC |
| XRDMA apps | `apps/rdma/xrdma_ping.c`, `apps/rdma/xrdma_test_utils.{c,h}` | ~2.8K LOC |
| XRDMA tests | `tests/xrdma_test.c` + xrdma branches in `tests/rdma_test.c`, `tests/rdma_cpi_test.c` | ~6K LOC |
| Public API (stays) | `sdk_include/FunOS/networking/rdma/ulp/xrdma/xrdma.h` | — |

## What moves out of FunOS (into the new XRDMA repo)
1. All 27 files under `networking/rdma/ulp/xrdma/`.
2. `services/rdma_cm/ulp/xrdma/xrdma_cm.c` (XRDMA-specific CM dispatch).
3. `apps/rdma/xrdma_ping.c`, `apps/rdma/xrdma_test_utils.{c,h}`.
4. `tests/xrdma_test.c`, plus the xrdma-only branches split out of
   `tests/rdma_test.c` and `tests/rdma_cpi_test.c`.
5. XRDMA-specific JSON dump helpers currently hosted in
   `props_bridges/dpsock_bridge.c` and `props_bridges/cpsock_bridge.c`
   (see coupling #3 below).

## Current in-repo couplings that must be cleaned up
Today, multiple FunOS files reach past the public SDK header and pull
**internal** XRDMA headers. These need to be reworked against a well-defined
SDK surface before XRDMA can leave the tree.

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

**Fix**: Move the XRDMA-specific JSON generators into the XRDMA repo.
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

**Fix**: Move XRDMA-specific stats and FTR registration into the XRDMA
library, registered dynamically at init. Requires a small extension to the
per-VP stats and FTR frameworks to accept dynamic registrations; fallback
is to leave these thin static entries in FunOS (minimal coupling).

### 6. Build system
- `networking/rdma/build.mk` compiles 14 XRDMA sources (`ulp/xrdma/*.c`).
- `services/rdma_cm/build.mk` compiles `ulp/xrdma/xrdma_cm.c`.
- `apps/build.mk` compiles XRDMA apps.
- `tests/build.mk` compiles `xrdma_test.c`.

**Fix**: Remove all of the above SRC_FILES entries. Add a link step against
the FunSDK-shipped XRDMA static library. Mirror an existing SDK-shipped
library's integration pattern (e.g. libfuncrypto) in FunSDK.

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

## New XRDMA repo layout (proposed)
```
xrdma/
  src/                # ex-networking/rdma/ulp/xrdma/*
  cm/                 # ex-services/rdma_cm/ulp/xrdma/xrdma_cm.c
  bridges/            # ex-props_bridges/* XRDMA generators
  apps/               # xrdma_ping, xrdma_test_utils
  tests/              # xrdma_test + split-out xrdma branches
  Makefile            # builds libxrdma.a
```
Delivered into FunSDK as `libxrdma.a` + public headers, consumed by FunOS
link.

## Work breakdown (execution order)
1. **Define SDK surface.** Draft the new SDK headers listed above. No code
   moves yet.
2. **Opaque PCB.** Make `xrdma_dp_socket` opaque in FunOS, move
   alloc/free and ULP ops to registration.
3. **Rework RDMA CPI.** Drop internal includes in `rdma_cpi.c` and
   `rdma_cpi_dpi_*.h`; use the new SDK accessors.
4. **Bridges.** Relocate XRDMA JSON generators; register them from XRDMA.
5. **roce_json.** Switch to registered query tables.
6. **Relocate source.** Move XRDMA core into the new repo; produce the
   static lib; ship via FunSDK.
7. **Relocate CM, apps, tests.** Move remaining XRDMA-specific code out.
8. **Stats & FTR.** Move or dynamically register.
9. **Build system.** Remove XRDMA SRC_FILES from 4 build.mk files; link
   the SDK-shipped library.
10. **Validate.** Full FunOS builds (POSIX, QEMU/s2); run the RDMA test
    suite; smoke-test XRDMA on QEMU and on FoD.

## Risks and open questions
- **Per-VP stats / FTR dynamic registration** — current frameworks use
  fixed X-macros. Either extend the frameworks or keep these small static
  entries in FunOS as an accepted residual coupling.
- **WU / channel handler linkage** — handler names are referenced in
  FunOS common code but defined in the prebuilt library; verify linker
  resolution or switch to dynamic handler registration with the WU
  framework.
- **rdma_cm ULP dispatch** — confirm `rdma_cm` supports runtime ULP
  registration; if not, add a small hook.
- **Test ownership** — generic RDMA tests stay in FunOS; XRDMA-only tests
  move. Split points in `rdma_test.c` / `rdma_cpi_test.c` need review
  with the test owner.
- **FunSDK release coupling** — XRDMA changes now require an SDK bump to
  land in FunOS; need to agree on cadence / release branches.

## Out of scope
- Refactoring XRDMA internals beyond what's needed for the split.
- Changes to other RDMA ULPs (SMBD, native) except where they share
  `rdma_cpi_dpi_*` headers affected by the split.
- Performance or functional changes to XRDMA behavior.
