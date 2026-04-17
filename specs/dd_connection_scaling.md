# Direct Drive — Why Connection Scaling Matters

## Document Purpose

This document is a **technical reference** explaining why connection scaling is a first-order
engineering challenge for the Direct Drive (DD) DPU system. It covers the architecture that
drives connection growth, the memory constraints that make it hard, and the strategies to
solve it.

**References:**
- "DD connection scaling requirements" meeting, November 21, 2025
  (Omar Cardona, Madhav Pandya, Suresh KN, Ravindran Suresh, Amit Surana, Greg Kramer,
  Savin Viswanath)
- xDPU DD memory budgeting exercise (Omar Cardona, Jaspal, Madhav Pandya, ~2024)

---

## 1. The Direct Drive Connection Model

### 1.1 What Drives Connection Count

In Direct Drive, every compute node (VDC) that accesses storage on a DD tenant establishes
**persistent network connections** to the DD storage node (BSS). These connections are
**long-lived** — they are created once when a VM boots and persist for hours, days, or months.
DDX tears down a connection only after it has been idle for ~1 hour (configurable). This is
fundamentally different from a high-CPS (connections-per-second) workload.

The number of connections on a DD node is determined by **how many compute nodes need to reach
it**, which is a function of **storage density**:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                   WHY CONNECTIONS GROW                                 │
│                                                                       │
│  More disk capacity per node                                          │
│       │                                                               │
│       ▼                                                               │
│  More tenants packed onto the node  (bin-packing)                     │
│       │                                                               │
│       ▼                                                               │
│  More VMs accessing those tenants                                     │
│       │                                                               │
│       ▼                                                               │
│  More compute nodes connecting to the DD node                         │
│       │                                                               │
│       ▼                                                               │
│  More network connections (4 per client: 2 TCP + 2 RDMA)              │
│                                                                       │
│  ┌──────────────┬──────────────┬──────────────┬───────────────┐       │
│  │  12×15TB     │  12×30TB     │  12×60TB     │  12×120TB     │       │
│  │  ~10K conn   │  ~20K conn   │  ~40K conn   │  ~80K conn    │       │
│  │  (baseline)  │  (PV2)       │  (future)    │  (future)     │       │
│  └──────────────┴──────────────┴──────────────┴───────────────┘       │
│                                                                       │
│  + Compression → even higher density → even more connections          │
└─────────────────────────────────────────────────────────────────────────┘
```

### 1.2 Connection Topology

Each VDC-to-BSS client pair establishes **4 connections** — two per transport (TCP and RDMA),
separated into small-IO and large-IO channels to prevent head-of-line blocking:

```
                     VDC Compute Node (Client)
                     ════════════════════════
                        │    │    │    │
            ┌───────────┘    │    │    └───────────┐
            │           ┌────┘    └────┐           │
            ▼           ▼              ▼           ▼
     ┌────────────┬────────────┬────────────┬────────────┐
     │  TCP       │  TCP       │  RDMA      │  RDMA      │
     │  Small IO  │  Large IO  │  Small IO  │  Large IO  │
     │  (≤16K)    │  (>16K)    │  (≤16K)    │  (>16K)    │
     └─────┬──────┴─────┬──────┴─────┬──────┴─────┬──────┘
           │            │            │            │
           └────────────┴────────────┴────────────┘
                              │
                              ▼
                     BSS Storage Node (DD DPU)
                     ════════════════════════

     A single DD node with 10K connected clients has:
       10K TCP-small + 10K TCP-large + 10K RDMA-small + 10K RDMA-large
       = 40,000 total connections
```

> **Note:** DD MVP (targeted Nov 2026) is **TCP-only** — no RDMA initially. First production
> will have 20K TCP connections (10K small + 10K large) per transport direction.

### 1.3 Traffic Profile

The fleet-wide IO size distribution is remarkably consistent:

```
     IO Size Distribution (consistent across all Azure regions)
     ─────────────────────────────────────────────────────
     ████████████████████████████████████████  80%  Small IO (≤16K)
     ██████████                                20%  Large IO (>16K)
```

- MTU: 4088 bytes (Azure standard)
- Per-sector metadata adds slightly beyond the nominal IO size on the wire
- Estimated ~50% of connections actively transmitting at any given time

---

## 2. The Memory Problem

### 2.1 Why Connection Scaling Is a Memory Problem

Each RDMA connection requires **pre-posted receive buffers** (RQ buffers) where incoming data
lands. The number and size of these buffers scale **linearly with connection count**, creating
the dominant memory pressure on the DPU:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                       │
│   RQ Buffer Memory  =  Connections  ×  RQ Depth  ×  Buffer Size       │
│                                                                       │
│   With full depth (256 RQEs × 8K per buffer):                         │
│                                                                       │
│   ┌──────────┬────────────┬──────────────────────────────────────────┐ │
│   │  Conns   │  RQ Memory │  vs. 512 MB DPU DDR                     │ │
│   ├──────────┼────────────┼──────────────────────────────────────────┤ │
│   │   1,000  │     2 GB   │  ████████████████████████▒  4× DDR      │ │
│   │  10,000  │    20 GB   │  ████████████████████████▒  39× DDR     │ │
│   │  20,000  │    40 GB   │  ████████████████████████▒  78× DDR     │ │
│   │ 100,000  │   200 GB   │  ████████████████████████▒  390× DDR    │ │
│   └──────────┴────────────┴──────────────────────────────────────────┘ │
│                                                                       │
│   Even 1,000 connections at full depth EXCEEDS the DPU's 512 MB DDR.  │
│                                                                       │
│   This is why dynamic credit management is essential — without it,    │
│   connection scaling is physically impossible on the DPU.             │
│                                                                       │
└─────────────────────────────────────────────────────────────────────────┘
```

### 2.2 DPU Memory Constraints

```
     DPU Memory Layout
     ═══════════════════════════════════════════

     ┌──────────────────────────────────────┐
     │            DDR (DRAM)                │  512 MB — FIXED, non-negotiable
     │                                      │  (current DPU SKU)
     │  Used for: connection state,         │
     │  control structures, RQ buffers,     │
     │  WU stacks, networking metadata      │
     │                                      │
     │  Possible upgrade: 1 TB (dual DIMM)  │
     │  → "super expensive"                 │
     ├──────────────────────────────────────┤
     │            HBM                       │  On-package, high bandwidth
     │                                      │
     │  Used for: packet buffers,           │
     │  transient data, BAM allocations     │
     └──────────────────────────────────────┘

     Fleet host memory (for context):
       Gen 9:  512 GB or 768 GB (majority)
       Gen 10: some 1 TB systems (few)
       Trend:  pressure to REDUCE to 512 GB
```

### 2.3 Per-Connection Memory Breakdown

Beyond RQ buffers, each connection consumes memory for state and metadata:

```
     Per-Connection Memory Footprint
     ═══════════════════════════════
     ┌─────────────────────────────────────────────┐
     │  RQ Buffers (dominant)     N × 8K each      │  ← Scales with credits granted
     ├─────────────────────────────────────────────┤
     │  Connection state (QP context, timers)      │  ← Fixed per connection
     ├─────────────────────────────────────────────┤
     │  RDMA E-stats              per-connection   │  ← Added post-budgeting exercise
     ├─────────────────────────────────────────────┤
     │  DCQCN state               per-connection   │  ← Added post-budgeting exercise
     ├─────────────────────────────────────────────┤
     │  WU structures (shared)    per-VP pool      │  ← Shared across connections on VP
     ├─────────────────────────────────────────────┤
     │  LE table entry            per-connection   │  ← Flow lookup state
     └─────────────────────────────────────────────┘

     Key: structures marked "Added post-budgeting" were NOT in the original
     ~2024 memory model and need to be accounted for in the updated model.
```

---

## 3. DPU Networking Architecture for Connection Scaling

### 3.1 Receive Path — How Connections Map to VPs

```
     Incoming Packet (from VDC compute node)
     ═══════════════════════════════════════

            ┌──────────────────────────┐
            │  ERP (Ethernet Rx Proc)  │
            │  ─ RSS / 5-tuple hash    │
            └───────────┬──────────────┘
                        │
                        ▼
            ┌──────────────────────────┐
            │   Lookup Engine (LE)     │
            │   128 queues             │
            │   (8 PCs × 16 queues)    │
            │                          │
            │   5-tuple → LE queue     │
            │   LE table → VP result   │◄── SW installs at connection setup
            └───────────┬──────────────┘
                        │
                        ▼
            ┌──────────────────────────┐
            │   Assigned VP            │
            │                          │
            │   Flow PINNED to this VP │
            │   for connection         │
            │   lifetime (no migration)│
            └──────────────────────────┘

     Key insight: SOFTWARE controls which VP a connection lands on.
     This enables load-aware distribution of connections across VPs.
```

### 3.2 Per-VP Resource Model

The networking stack implements a **resource capping contract** with the storage layer:

```
     ┌─────────────────────────────────────────────────────────────────┐
     │                        VP Resource Model                       │
     │                                                                │
     │                      ┌──────────────┐                          │
     │                      │     VP 0     │                          │
     │                      │              │                          │
     │    Conn A ──────────►│  Outstanding │                          │
     │    Conn B ──────────►│  operations  │  MAX 10,000              │
     │    Conn C ──────────►│  across ALL  │  (across all connections │
     │      ...  ──────────►│  connections │   on this VP)            │
     │    Conn N ──────────►│              │                          │
     │                      └──────────────┘                          │
     │                                                                │
     │  ┌────────────────────────────────────────────────────────┐    │
     │  │  Contract:                                             │    │
     │  │  • Connection count: UNLIMITED                         │    │
     │  │  • Outstanding work requests per VP: ≤ 10,000          │    │
     │  │  • Storage ULP guarantees it won't exceed this cap     │    │
     │  │  • Resources (WU structs, buffers) pre-allocated       │    │
     │  │    for contracted workload → no alloc failures         │    │
     │  └────────────────────────────────────────────────────────┘    │
     │                                                                │
     │  This is the handshake between DDX/XStore and networking.      │
     │  The 10K limit is on OPERATIONS, not connections.              │
     └─────────────────────────────────────────────────────────────────┘
```

---

## 4. The Solution: Dynamic Credit Management

### 4.1 How SMB Direct Credits Work

```
     SMB Direct Credit-Based Flow Control
     ═════════════════════════════════════

     Client (VDC)                              Server (DD DPU)
     ────────────                              ───────────────

     1. Connect
        ─────────────────────────────────────►
                                               Negotiate parameters
        ◄─────────────────────────────────────
                                               Grant initial credits (e.g. 2)

     2. Send IO request
        (consumes 1 credit)
        ─────────────────────────────────────►
                                               Process IO
                                               Return credit (or not!)
        ◄─────────────────────────────────────

     3. Client has 0 credits → BLOCKED
        (cannot send until server grants more)

     Key: The server controls how many outstanding
     requests the client can have by managing credits.
     The client's request for 255 buffers is ADVISORY —
     the server may grant as few as 1.
```

### 4.2 Dynamic Credit Adjustment — Solving the Memory Problem

```
     Without Dynamic Credits                With Dynamic Credits
     ═══════════════════════                ═══════════════════════

     10K connections                        10K connections
     × 256 RQ depth                         × 2 RQ depth (minimum)
     × 8K buffer                            × 8K buffer
     ────────────────                       ────────────────
     = 20 GB  ❌                            = 160 MB  ✅ fits in DDR!
       (39× the DDR)                          (31% of DDR)

     20K connections                        20K connections
     × 256 RQ depth                         × 2 RQ depth
     × 8K buffer                            × 8K buffer
     ────────────────                       ────────────────
     = 40 GB  ❌                            = 320 MB  ✅ fits in DDR!

     100K connections                       100K connections
     × 256 RQ depth                         × 2 RQ depth
     × 8K buffer                            × 8K buffer
     ────────────────                       ────────────────
     = 200 GB ❌                            = 1.6 GB  ⚠️ needs HBM
```

### 4.3 Credit Lifecycle with Reclamation

```
     Active Connection                       Idle Connection
     ═══════════════════                     ═══════════════════

     Client sends IO continuously            Client idle (no IO)
           │                                       │
           ▼                                       ▼
     Server processes IO                     DDX sends keepalive
     Server returns credits                  Server responds
     (maintains buffer pool)                 Server does NOT return credit
           │                                       │
           ▼                                       ▼
     Client keeps sending                    Buffers reclaimed to pool
     (credits replenished)                   (available for active connections)

     Result: Memory dynamically flows to where it's needed.
     Idle connections cost ~0 buffers; active ones get fair share.
```

### 4.4 Why Not Shared Receive Queue (SRQ)?

| Criteria | Dynamic Credits | SRQ |
|---|---|---|
| Protocol change | None | None (technically), but complex |
| Client-side changes | None | None |
| Implementation complexity | Low (server-side credit math) | High ("fraught with peril" — Greg Kramer) |
| Industry support | Universal (standard SMB Direct) | Poor vendor support |
| Windows SMB Direct usage | Yes (standard mechanism) | No (deliberately not used) |
| DPU HW support needed | None | Potentially |

**Decision: Dynamic credit adjustment.** No protocol changes, no client changes, server-side
only.

---

## 5. Stability Requirement: Must Never Crash

```
     ┌─────────────────────────────────────────────────────────────────┐
     │                                                                │
     │         REQUIRED BEHAVIOR UNDER OVERLOAD                       │
     │                                                                │
     │   Throughput                                                   │
     │      ▲                                                         │
     │      │          ┌──────────────────── Plateau (REQUIRED)       │
     │      │         ╱                                               │
     │      │        ╱                                                │
     │      │       ╱          ← Linear scaling region                │
     │      │      ╱                                                  │
     │      │     ╱                                                   │
     │      │    ╱                                                    │
     │      │   ╱                                                     │
     │      │  ╱                                                      │
     │      │ ╱                                                       │
     │      └──────────────────────────────────────► Connections       │
     │           10K     20K     40K     100K                         │
     │                                                                │
     │   ✅ Performance degrades gracefully (plateau)                 │
     │   ⚠️  Node struggles with latency/throughput                   │
     │   ❌ Node crashes  ← ABSOLUTELY FORBIDDEN                     │
     │                       "7 zeros every other day" — Greg Kramer  │
     │                                                                │
     └─────────────────────────────────────────────────────────────────┘
```

This requires:
- Proper memory allocation failure handling (never assert on alloc failure)
- Backpressure via credit throttling (don't accept more work than resources allow)
- Per-VP resource caps (contract between storage and networking)
- Graceful rejection of new connections when at capacity

---

## 6. Data Transfer Modes and DPU Impact

### 6.1 RDMA Read/Write vs Send/Receive

```
     Mode 1: RDMA Read/Write (Pull Model)
     ═════════════════════════════════════
     Optimal for: LOW latency fabrics (same rack/cluster)

     Client                                 Server (DPU)
     ──────                                 ────────────
     1. Send IO request (small msg) ──────► Receive request
                                            Fetch data from disk
     2. ◄──── RDMA Read (HW DMA) ─────────  Post read response
        Data arrives via DMA                 VP NOT involved per-packet ✅
        (zero-copy, HW handles)

     Advantage: VP bypass — RNIC HW handles bulk data transfer


     Mode 2: Immediate Outdata / Send-Receive (Push Model)
     ═════════════════════════════════════════════════════
     Optimal for: HIGH latency fabrics (cross-zone, hundreds of µs RTT)

     Client                                 Server (DPU)
     ──────                                 ────────────
     1. Send IO request + data ────────────► Receive request + data
        (inline, single message)             VP involved per-packet ⚠️
                                             (processes each send)

     Advantage: Saves 1 RTT (no separate RDMA read round-trip)
     Disadvantage: VP involved per-packet — more CPU pressure


     Performance Crossover:
     ─────────────────────────────────────────────────────────
     IOPS ▲
          │  Read/Write wins     │  Send/Receive wins
          │  (low latency)       │  (high latency)
          │         ╲            │           ╱
          │          ╲           │          ╱
          │           ╲──────── ─┼─────── ╱
          │            ╲         │       ╱
          │             ╲        │      ╱
          └──────────────┴───────┴──────────────► RTT
                         crossover point
```

### 6.2 Current Fleet Behavior

| Platform | Default Mode | Notes |
|---|---|---|
| Overlake | RDMA Read/Write only | Does not support immediate outdata yet |
| VDC (non-Overlake) | Immediate outdata | Cannot estimate peer distance |
| DPU target | RDMA Read/Write (within cluster) | Needs profiling; low-RTT expected |

---

## 7. Connection Scaling Roadmap

```
     ┌─────────────────────────────────────────────────────────────────┐
     │                                                                │
     │  Phase 1: DD MVP (Nov 2026)                                    │
     │  ─────────────────────────                                     │
     │  • TCP-only (no RDMA)                                          │
     │  • Target: 10K connections per transport                       │
     │  • Hardware: PV2 (12×30TB) → expect ~20K in production         │
     │  • Memory: dynamic credit management required                  │
     │                                                                │
     │  Phase 2: RDMA Addition                                        │
     │  ─────────────────────                                         │
     │  • Add RDMA transport alongside TCP                            │
     │  • Total: 20K TCP + 20K RDMA = 40K connections                 │
     │  • Profile DPU RNIC send/receive vs read/write performance     │
     │                                                                │
     │  Phase 3: Full Scale                                           │
     │  ──────────────────                                            │
     │  • 60TB/120TB disks + compression                              │
     │  • Target: up to 100K connections per transport                 │
     │  • Memory: likely requires DDR expansion or HBM utilization    │
     │  • DPU-specific telemetry for active-connection monitoring     │
     │                                                                │
     └─────────────────────────────────────────────────────────────────┘
```

---

## 8. Key Decisions and Open Items

### Decisions Made

1. **10K per transport is the MVP floor**, not a ceiling — 20K for PV2 production
2. **Dynamic credit adjustment** is the approach (not SRQ)
3. **Per-VP model:** 10K outstanding operations cap, unlimited connections
4. **DD MVP is TCP-only**
5. **Must never crash** — graceful degradation under overload
6. **Assume 50% active connections** until DPU telemetry provides real data

### Open Items

1. Update memory budgeting model with actual implemented structures (RDMA E-stats, DCQCN)
2. Create combined high-fidelity model: Networking + DD + BOSS
3. Profile DPU RNIC: Send/Receive path vs RDMA Read/Write performance
4. Define per-DPU-node IOPS and throughput targets
5. Build DPU-specific telemetry for active connection monitoring
6. Assess Armada cluster topology impact on transport mode selection

---

## Appendix: Glossary

| Term | Definition |
|---|---|
| DD | Direct Drive — Azure storage arch where DPU directly serves disk I/O |
| DDX | Direct Drive Extensions — SMB Direct protocol implementation for DD |
| BSS | Block Storage Server — the DD storage node |
| VDC | Virtual Data Center — the compute node running VMs |
| CPS | Connections Per Second |
| RDMA | Remote Direct Memory Access |
| SRQ | Shared Receive Queue — RDMA feature for pooling receive buffers |
| RQ / RQE | Receive Queue / Receive Queue Element |
| LE | Lookup Engine — HW accelerator for flow classification |
| VP | Virtual Processor — FunOS processing core |
| WU | Work Unit — FunOS scheduling primitive |
| ERP | Ethernet Receive Processor |
| DDR | Double Data Rate memory (DRAM attached to DPU) |
| HBM | High Bandwidth Memory (on-package memory) |
| RTT | Round-Trip Time |
| MTU | Maximum Transmission Unit (4088 bytes in Azure) |
| DCQCN | Data Center Quantized Congestion Notification |
| DDM5 | DD Milestone 5 — stabilization phase part 2 |
| PV2 | Production Vehicle 2 — next-gen hardware platform |
| BOSS | Block Object Storage System |
| Overlake | Azure compute platform / NIC |
