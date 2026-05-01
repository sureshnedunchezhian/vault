# Vault

Shared documents, specs, presentations, and investigation notes.

## Specs

| Document | Description |
|----------|-------------|
| [MHP Opens Consolidated Report](specs/MHP_Opens_Consolidated_Report.md) | ANS/HostNet offsite — MHP design decisions & open items |
| [Flash Architecture](specs/flash_arch.html) ([view](https://sureshnedunchezhian.github.io/vault/specs/flash_arch.html)) | Flash assembler architecture reference |
| [TCP VP Selection](specs/tcp_vp_selection.md) | TCP VP selection from worker pool |
| [VFP Proxy Protocol V2 Spec](specs/vfp_proxy_protocol_v2_spec.md) | VFP Proxy Protocol V2 design specification |
| [XRDMA Extraction Scope](specs/xrdma_extraction_scope.md) | Scope of work to move XRDMA ULP out of FunOS into a standalone repo |
| [DD Connection Scaling](specs/dd_connection_scaling.md) | Why connection scaling matters for Direct Drive — architecture, memory constraints, and solutions |
| [NFM Architecture](specs/nfm_arch.md) | NU Forwarding Module architecture — modules, APIs, VP context, state machines, issues & improvements (includes Mermaid diagrams) |
| [Flash & Funnel Build Options](specs/flash_funnel_build_options.md) | Where should .flash/.yaml source programs live — FunOS vs tool repos, projectdb.json deps, artifact flows (includes Mermaid diagrams) |
| [TCP Control Path Analysis](specs/tcp_control_path_analysis.md) | FunOS TCP control path — client/server establishment, teardown, abort, WU handlers, VP queues, flow directions, app interaction with tcp_pkt_gen/tcp_pkt_rcv (includes Mermaid diagrams) |
| [Source Port Selection (TCP & RDMA CM)](specs/tcp_source_port_selection.md) | Linux vs Windows TCP and RDMA CM source port selection — algorithms (hash-based, CSPRNG, simple random), sysctl/registry tunables, socket options, Service ID encoding, RFC 6056/6335 compliance |
| [F2 TMM Timer Architecture](specs/f2_tmm_timer_architecture.md) | F2 TMM timer block — RTL state machine, FGI/TWC commands, `nucleus/timer.h` + `hw_timer.h` SW API, `flow_timer` wrapper, race-window analysis between flow_timer and tcp_pkt_gen/rcv (includes Mermaid diagrams) |


## Presentations

| Document | Description |
|----------|-------------|
| [AI Dev Presentation](presentations/ai_dev_presentation.html) ([view](https://sureshnedunchezhian.github.io/vault/presentations/ai_dev_presentation.html)) | AI-assisted development workflow |

## Investigations

| Document | Description |
|----------|-------------|
| [ADO 3088420 — COMe PXE Boot](investigations/3088420_come_pxe_boot.md) | COMe not reachable after PXE boot analysis |
| [ADO 2568748 — BAM ETP Cmdlist Resource Check](investigations/2568748_bam_alloc_etp_cmdlist_analysis.md) | BAM ETP cmdlist allocation paths missing xmt_res_nu resource checks in TCP control-plane |
| [QP Teardown Crash — 32 QPs Loopback](investigations/qp_teardown_crash_32qps.md) | BUG_ON(qp_state != RESET) in roce_qp_fini during write_bw loopback teardown with 32 QPs on F1D1 |
| [ADO 3383627 — RDMA VP Priority & PFC](investigations/3383627_rdma_vp_priority_pfc.md) | RNIC TX/RX VP priority inversion causing BAM buildup and PFC at 11Mpps — root cause, fix, VP priority map, Mermaid flow diagrams |
| [ADO 2405507 — S21F1 Multicast Filtering](investigations/2405507_s21f1_multicast_filtering.md) | S21F1 funeth NIC accepts all multicast — root cause analysis, MANA vs funeth comparison, NDIS OID gap analysis, implementation options |
