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
