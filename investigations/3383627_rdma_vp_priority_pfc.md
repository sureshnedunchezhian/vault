# RDMA VP Priority Analysis & TX/RX Decoupling Fix

**ADO 3383627** | Branch: `suresh/rdma_fc` | Commit: `ea3e1915b1e`

## Problem Statement

At 11Mpps RDMA writes with 1KB PTMU, the sender generates ~11Mpps ACKs returning from the responder. All 64 VPs are saturated. BAM builds up, flow controls back to ERP → PSW → PFC.

**VP profiling shows the bottleneck:**
```
rnic_nu_cont_wu:                          count=165,797  avg=1.76us  (TX continuation - HIGH prio)
WU_RNIC_RX_RESPONSE_NO_ERR_BASE1 (ACK):  count=165,811  avg=1.57us  (ACK processing - was LOW prio)
rnic_send_pkts_wu:                        count=124,352  avg=738ns   (TX packet build - LOW prio)
roce_fm_send_req_batch:                   count=41,452   avg=5.77us  (ULP post_send - LOW prio)
```

~50 WUs sitting in low priority queue, no buildup in high priority queue.

## Root Cause

```mermaid
graph TD
    subgraph "BEFORE FIX - Priority Inversion"
        A[ULP posts RDMA Write] -->|LOW prio| B[roce_fm_send_req_batch]
        B -->|LOW prio| C[rnic_send_pkts_wu]
        C --> D[rnic_tx_func → RNIC TXP HW]
        D -->|HIGH prio| E[rnic_nu_cont_wu]
        E -->|calls| C
        D --> F[Packet on wire → Responder]
        F --> G[ACK back from network]
        G --> H[RNIC RXP HW]
        H -->|LOW prio| I[rnic_rx_nwk_valid_ack]
        I -->|clears gated_wu_cnt| J[More TX triggered]
        J -->|LOW prio| C
    end

    style E fill:#ff6666,stroke:#333,color:#000
    style I fill:#6666ff,stroke:#333,color:#fff
    style H fill:#6666ff,stroke:#333,color:#fff
```

**Three problems:**
1. **ACKs on LOW priority** — RNIC RXP dispatches to `sq_vp` which is LOW priority (`faddr_vp_to_low_priority()` at `roce_qp.c:865`). ACKs compete with TX work.
2. **TX continuation on HIGH priority** — `rnic_nu_cont_wu` dispatched to `faddr_vp_to_high_priority(xmt_vp)` (`rnic_tx.c:1274`). Starves ACK processing.
3. **No TX backpressure** — At `next_wr`, bare `wu_send_ungated` generates unlimited TX work with no resource gating.

**Positive feedback loop:** TX generates packets → ACKs return but starved by HIGH prio TX continuations → BAM fills → ERP flow control → PSW → PFC.

## Fix

```mermaid
graph TD
    subgraph "AFTER FIX - Decoupled TX/RX"
        A[ULP posts RDMA Write] -->|LOW prio| B[roce_fm_send_req_batch]
        B -->|LOW prio| C[rnic_send_pkts_wu]
        C --> D[rnic_tx_func → RNIC TXP HW]
        D --> F[Packet on wire → Responder]
        F --> G[ACK back from network]
        G --> H[RNIC RXP HW]
        H -->|HIGH prio| I[rnic_rx_nwk_valid_ack]
        C -->|EQM gated, LOW prio| C2[next_wr: res_flow_available check]
        C2 -->|resources OK| C
        C2 -->|resources RED| EQM[EQM waits for resources]
        EQM -->|LOW prio| R[roce_tx_restart → rnic_send_pkts]
    end

    style I fill:#00cc66,stroke:#333,color:#000
    style H fill:#00cc66,stroke:#333,color:#000
    style EQM fill:#ffcc00,stroke:#333,color:#000
```

### Change 1: ACK RX → HIGH Priority
**File:** `roce_qp.c:611-613, 672-674`

Program `faddr_vp_to_high_priority(sq_vp)` into RNIC QP context via `hw_rnic_qp_trans_rts()` and `hw_rnic_qp_trans_reset()`. All RNIC RXP-dispatched WUs (ACKs, read responses, requests, errors) now land on HIGH priority queue.

### Change 2: Eliminate `rnic_nu_cont_wu` for TX Request Path
**File:** `rnic_tx.c:1976`

Set `need_nu_cont_wu = false` for TX request packets. The `next_wr` dispatch already provides TX continuation — the NU continuation WU was redundant. Removes ~165K HIGH priority WUs/sec per VP. RSP flow (read-response) retains the continuation.

### Change 3: EQM-Gated TX Restart at `next_wr`
**File:** `rnic_tx.c:2005-2018`

Add `res_flow_available()` check at `next_wr` (gated on `enable_tx_restart_via_eqm` modcfg). When NU DMA resources are constrained, the flow is registered with EQM for restart via `roce_tx_restart`. This provides VP-level backpressure instead of unbounded `wu_send_ungated`.

The `res_fail` path (resource exhaustion during packet build) already has EQM restart built in — `res_flow_available()` inside `rnic_tx_res_chk()` calls `flow_resched_req()` when resources are RED.

## VP Priority Map

### RNIC Mode (HW Offload)

```mermaid
graph TD
    subgraph HIGH["HIGH PRIORITY QUEUE"]
        RX1["RNIC RXP: ACKs, Responses, Requests<br/>(all NO_ERR + ERR handlers)"]
        RX3[rnic_ntohf_restart_more]
        TX_RSP["rnic_nu_cont_wu<br/>(RSP flow only)"]
        CTRL1[rnic_tx_flush_qp_done]
        ADMIN["roce_vp_handler<br/>roce_le_entry_add_done"]
    end

    subgraph LOW["LOW PRIORITY QUEUE"]
        TX1[rnic_send_pkts_wu]
        TX2[roce_fm_send_req_batch]
        TX3[roce_send_rsp_wu]
        EQM1[roce_tx_restart]
        EQM2[roce_rsp_restart]
        EQM3[roce_ntohf_restart]
        CTRL2["rnic_flush_qp<br/>rnic_clear_qp"]
        LOCAL[roce_comp_local_term_wr_wu]
    end

    style RX1 fill:#00cc66,stroke:#333,color:#000
    style RX3 fill:#00cc66,stroke:#333,color:#000
    style TX_RSP fill:#ff9966,stroke:#333,color:#000
    style TX1 fill:#6699ff,stroke:#333,color:#000
    style TX2 fill:#6699ff,stroke:#333,color:#000
    style TX3 fill:#6699ff,stroke:#333,color:#000
    style EQM1 fill:#ffcc00,stroke:#333,color:#000
    style EQM2 fill:#ffcc00,stroke:#333,color:#000
    style EQM3 fill:#ffcc00,stroke:#333,color:#000
```

| WU Handler | Priority | Path | Dispatch Mechanism |
|---|---|---|---|
| **RNIC RXP: all NO_ERR handlers** (ACK, read resp, send/write req, CNP, etc.) | **HIGH** | RX | RNIC HW → `faddr_vp_to_high_priority(sq_vp)` in QP context |
| **RNIC RXP: all ERR handlers** (unexpected PSN, NAK, size violation, etc.) | **HIGH** | RX | RNIC HW → same QP context VP |
| `rnic_nu_cont_wu` | **HIGH** | TX RSP | NU DMA completion (RSP flow only after fix) |
| `rnic_ntohf_restart_more` | **HIGH** | RX restart | `faddr_vp_to_high_priority(xmt_vp)` |
| `rnic_tx_flush_qp_done` | **HIGH** | Control | NU DMA continuation |
| `roce_vp_handler` | **HIGH** | Admin | MR deregister |
| `roce_le_entry_add_done` | **HIGH** | Admin | Modify key |
| `rnic_send_pkts_wu` | **LOW** | TX | `RNIC_CALL_SEND_PKTS` → `xmt_vp` |
| `roce_fm_send_req_batch` | **LOW** | TX | ULP channel push |
| `roce_send_rsp_wu` | **LOW** | TX RSP | Read response dispatch |
| `roce_tx_restart` | **LOW** | EQM | Flow restart framework |
| `roce_rsp_restart` | **LOW** | EQM | RSP flow restart |
| `roce_ntohf_restart` | **LOW** | EQM | ntoh flow restart |
| `rnic_flush_qp` | **LOW** | Control | QP flush initiation |
| `rnic_clear_qp` | **LOW** | Control | QP clear |
| `roce_comp_local_term_wr_wu` | **LOW** | TX | Local termination |

### SW RoCE Mode (No RNIC HW)

| WU Handler | Priority | Path | Dispatch Mechanism |
|---|---|---|---|
| `roce_nu_cont_wu` | **HIGH** | TX cont | NU DMA completion → `faddr_vp_to_high_priority(xmt_vp)` |
| `roce_ntohf_restart_more` | **HIGH** | RX restart | `faddr_vp_to_high_priority(xmt_vp)` |
| `roce_vp_handler` | **HIGH** | Admin | MR deregister |
| `roce_le_entry_add_done` | **HIGH** | Admin | Modify key |
| `roce_send_pkts_wu` | **LOW** | TX | TX dispatch → `xmt_vp` |
| `roce_send_rsp_wu` | **LOW** | TX RSP | Read response dispatch |
| `roce_tx_restart` | **LOW** | EQM | Flow restart framework |
| `roce_rsp_restart` | **LOW** | EQM | RSP flow restart |
| `roce_ntohf_restart` | **LOW** | EQM | ntoh flow restart |
| `roce_fm_send_req_batch` | **LOW** | TX | ULP post_send |
| `roce_comp_local_term_wr_wu` | **LOW** | TX | Local termination |
| RX packet handlers (RC/UD) | **LOW** | RX | NU ERP→LE→VP at `xmt_vp` (LOW) |

**Key difference:** In SW RoCE, RX packets arrive via LE at LOW priority (flow destination = `xmt_vp`). `roce_nu_cont_wu` (TX continuation) is the main HIGH priority WU — creating a similar priority concern as the original RNIC bug, but in the opposite direction.

## TX Flow Diagram (RNIC Mode — After Fix)

```mermaid
sequenceDiagram
    participant ULP as ULP (xrdma/smbd)
    participant FM as roce_fm_send_req_batch
    participant TX as rnic_send_pkts
    participant RES as res_flow_available
    participant TXP as RNIC TXP HW
    participant NET as Network
    participant RXP as RNIC RXP HW
    participant ACK as rnic_rx_nwk_valid_ack
    participant EQM as EQM

    ULP->>FM: post_send (LOW prio)
    FM->>TX: rnic_send_pkts_wu (LOW prio)
    
    loop For each packet (up to nwu)
        TX->>RES: rnic_tx_res_chk()
        alt Resources available
            TX->>TXP: rnic_tx_func (need_nu_cont_wu=false)
            TXP->>NET: Packet on wire
        else Resources RED
            RES-->>EQM: flow_resched_req (automatic)
            EQM-->>TX: roce_tx_restart (LOW prio, when resources GREEN)
        end
    end
    
    TX->>RES: next_wr: res_flow_available()
    alt Resources available
        TX->>TX: wu_send_ungated (LOW prio)
    else Resources RED  
        RES-->>EQM: flow_resched_req
        EQM-->>TX: roce_tx_restart (LOW prio)
    end

    NET->>RXP: ACK from responder
    RXP->>ACK: Dispatch (HIGH prio)
    ACK->>ACK: rnic_gen_sq_cqe (completions)
    ACK->>ACK: roce_update_retry_timer
```

## Validation Results

| Test | Target | Result |
|---|---|---|
| POSIX build + write_bw | f1d1-posix | ✅ Clean exit |
| POSIX build + write_bw | f2-posix | ✅ Clean exit |
| FoD write_bw loopback | S21F2-XIO (job 5816881) | ✅ **268 Gbps** |

**FoD details:** 4 QPs, 256KB IO size, 30s runtime, `send_depth=16`
- Client: 3,842,068 IOs, 128K IOPS, **268.56 Gbps**
- All 4 connections completed, 0 failures
- `RNIC_tx_nu_dma_fails: 11,422,821` — confirms EQM backpressure is actively throttling TX when NU DMA resources are constrained

## Configuration

The EQM restart is gated on existing modcfg flag:
```
modcfg set rdma/enable_tx_restart_via_eqm true   # enable (default)
modcfg set rdma/enable_tx_restart_via_eqm false   # disable (bypass EQM check)
```

## Files Changed

| File | Change |
|---|---|
| `networking/rdma/roce/roce_qp.c` | `hw_rnic_qp_trans_rts/reset` → `faddr_vp_to_high_priority(sq_vp)` |
| `networking/rdma/roce/rnic_tx.c` | Eliminate `rnic_nu_cont_wu` for TX req path; add EQM check at `next_wr` |
