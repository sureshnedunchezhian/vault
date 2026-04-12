# ANS[VNET,SLB] / HostNet[VFP] Offsite — MHP Opens
## Consolidated Design Decisions, Discussions & Open Items

**Sources:** Offsite meeting chat (4 weeks: Mar 1–27, 2026) + AzDASH Forum group chat (full history since Feb 2026)
**Key Participants:** Owen Chiaventone, Osman Ertugay, Sai Vavilala, Trilok Nuwal, Sowmini Varadhan, Vivek Bhanu, Abhijeet Kumar, Sachin, Chaitanya Raje, Pranjal Shrivastava, Vipin, Tamilmani

---

## 1. SWIFT-v1 Architecture (Top Priority)

### Decision
- Proceed with **SWIFT-lite (SWIFT-v1)**, NOT SWIFT-1.5.
- **SLBHP remains unaware of containers** — continues to program policies at multi-tenant ENI scope.

### Implications
- Per-container ENIs as first-class objects **do not work** for SWIFT-v1 due to shared NAT pools and stateful return traffic handling at MT ENI level.
- Container demux logic will move away from scattered VFP rule matches toward a cleaner model agreed between VNET and HostNet teams.

### Open Items
| # | Item | Owner | Status |
|---|------|-------|--------|
| 1 | Whether additional "containers" map to **secondary CAs** for inbound processing (VNI→VLAN mapping) | VNET (Owen, Abhijeet) | **Open** |
| 2 | Consolidate and tag SWIFT questions in AzDASH forum | Osman | **Open** |
| 3 | Offline sync with SWIFT team (Vipin, Tamilmani) | Abhijeet, Sai | **Open** |

---

## 2. AzDASH Object & ID Model

### Decisions
- **ID spaces for different object types may overlap** — e.g., a Security Policy can share the same GUID as an ENI.
  - **Rationale:** Sometimes we generate objects not based on a single Merlin object (e.g., ENI-specific security policy). If the GUID is allowed to be the same as the ENI GUID, live migration and rehydration are simpler — VCPA doesn't need to store generated GUIDs. *(Owen)*
  - **Caveat:** At DASH level, the object ID needs to be "**object type + GUID**" to disambiguate. *(Trilok)*
- **VNETs are globally scoped per node** — mappings pulled by one ENI/vport are visible to all others on the same node (shared cache semantics). Both VFP-NG and OVL4 designs are the same in this respect. *(Osman)*
- **Runtime programming needs only `objectid:cookie`**, not full object payloads. *(Trilok)*

### Open Items
| # | Item | Owner | Status |
|---|------|-------|--------|
| 4 | Whether diagnostic "get" APIs (which ENIs reference an object) should live in AzDASH or separate diag interface | Osman, Trilok | **Open** |
| 5 | Whether diagnostics should proxy through VCPA or be called remotely | Osman | **Open** |

---

## 3. ENI Semantics

### Decisions
- **ENI type (VNET vs non-VNET) is immutable after creation.**
  - Determined by presence of `ENIVNETProperties` at create time.
  - Updates that change ENI type must be **rejected** — would break in-flight traffic and flow state.
  - Proto constraints alone are insufficient; enforcement must be in control-plane logic.

---

## 4. ORT Routes & Route Programming

### Decisions / Agreed Facts
- **3 groups of routes** in ORT programming:
  - **UDR**: ~2K routes (64 max next hops), relatively lower change frequency
  - **VNET+Peering**: 5K entries (dual stack w/ multiple prefixes → 5K–20K routes), relatively lower change frequency
  - **Gateway (GW)**: ~10K routes (64 max next hops), updates supported every 10s
- **Goal-state cookies are per-group** (GW, UDR, VNET). GW group updated only on actual route changes. *(Trilok, Osman)*
- Strong leaning toward **combined ORT model** (VNET + SLB) instead of adding new layers — avoid "VFP-style explosion" of layers.

### Open Items
| # | Item | Owner | Status |
|---|------|-------|--------|
| 6 | Production stats on current route table sizes | Owen, Trilok, Chaitanya | **Open** |
| 7 | LPM vs priority-based ORT rules for SLB integration | Osman | **In Discussion** |

---

## 5. Mappings & Hot Cache (Infinite VNET)

### Decisions / Agreed Facts
- **Mapping cache is node-global**, not per-port.
  - Once a mapping is active, all ENIs can consume it.
  - Spurious duplicate mapping requests are acceptable and non-impacting. *(Osman)*
- **VDPA interface semantics:**
  - `query mapping` → activates mapping
  - `evict mapping` → deactivates
  - `list all active mappings` → required for rehydration after crash/LM
- **Hot cache (Infinite VNET)**: Active mappings must be re-enumerated during rehydration.
- Per-port active mapping state tracking is a **major VDPA design factor**, especially for MHP.

### Open Items
| # | Item | Owner | Status |
|---|------|-------|--------|
| 8 | gRPC vs **FlatBuffers + shared memory/UDS** for VDPA ↔ VFP-NG local interface (perf concern with gRPC serialization) | Osman | **Open** |

---

## 6. Encapsulation & Data Path

### Decisions
- **VXLAN is primary encapsulation**; NVGRE retained for legacy compute scenarios.
- **Inbound main-flow handling (SLB):** Uses VNI→VLAN mapping (VNET2VLAN or ProviderVNI2VLAN) to select correct policy context.
- **Inbound response flows:** Default VLAN-ID to 0, except for VNET cases.

### Open Items
| # | Item | Owner | Status |
|---|------|-------|--------|
| 9 | Whether encapsulation beyond VXLAN/NVGRE is expected or required | VNET | **Open** |
| 10 | Inbound traffic model — secondary CA model or something else? | Owen, Abhijeet | **Open** |

---

## 7. Security Policies & Traffic Controls

### Decisions
- Security policies may be **generated ENI-specific objects** — GUID reuse across object types simplifies LM and rehydration.
- **ICMP Redirect Fast Path:** Source IPs restricted to small, known prefix sets (e.g., MUX CIDRs). Expressible via prefix conditions, not dynamic rules.

### DSCP Tagging
- Configured **per ENI**.
- Evaluated **first** in outbound processing.
- Based on **5-tuple match**.
- Applied to the **outermost header** after SDN transformations.
- Expected to be **small rule sets (<10 rules)** per ENI.

### VTAP
- Modeled as **post-SDN appliance encapsulation**.
- Uses VXLAN with VNI determined by traffic class (ServiceTunnel, Internet, PrivateEndpoint, etc.).
- Appliance decapsulates and forwards after inspection.
- Implemented as part of **ENI spec**, not a separate pipeline stage.

### Open Items
| # | Item | Owner | Status |
|---|------|-------|--------|
| 11 | Exhaustive list of DSCP / VTAP / policy rules used today by VCPA/NMA | Chaitanya Raje | **Action Item (Open)** |
| 12 | VTAP mode — for appliance or host? | Pranjal (raised question) | **Open** |

---

## 8. Metering, Counters & CID Plumbing

### Options Discussed
1. Add CID directly to ENI spec
2. Cache CID via VDPA (legacy VFP-style)
3. Query NmAgent (discouraged — inbound gRPC concerns)

### Direction
- Strong preference toward **push-based models** rather than pull.

### Open Items
| # | Item | Owner | Status |
|---|------|-------|--------|
| 13 | Final CID ↔ ENI/MAC mapping approach | TBD | **Open** |

---

## 9. Live Migration (LM)

### Consensus
- Current LM model has **blackouts and long-lived flow issues**.
- SLB involvement may be required beyond current single-mapping assumptions.

### Design Direction
- ENI-scoped, stable IDs (**MeterClass, cookies**) must survive source→target LM.
- VCPA rehydration relies on **deterministic IDs and mapping replay**.
- Active mapping re-enumeration is critical for post-LM recovery.

### Open Items
| # | Item | Owner | Status |
|---|------|-------|--------|
| 14 | Deep dive with SLB, HostNet, and dataplane owners to close LM gaps | Sai (scheduling) | **Open** |

---

## 10. Multi-VNET Container Behavior

### Open Items
| # | Item | Owner | Status |
|---|------|-------|--------|
| 15 | How containers on different VNETs interact when peered to the same VNET | VNET | **Open** |

---

## 11. Storage / SNAT Bypass

### Direction
- Requires **prefix (LPM) matching** for storage VIP ranges.
- Leaning toward combined ORT model (VNET + SLB) instead of new layers.

---

## 12. Lab Access & Bootstrapping

### Action Items
| # | Item | Owner | Status |
|---|------|-------|--------|
| 16 | Provide next steps for Owen & Abhijeet to get lab machine access and bootstrap agents | Vivek Bhanu | **Open** |

---

## 13. Process & Scheduling

### Decisions
- **AzDASH Forum** is the authoritative place for API and deep-dive technical discussion.
- Continue progress **asynchronously** between offsites.
- **One more offsite will be needed** to close remaining items. *(Vivek)*
- **Mappings call** was planned for Thu 3/26 but **deferred** — VNET team (Owen, Abhijeet, Sachin) need internal alignment on requirements first. Rescheduled to Monday. *(Sowmini, Owen agreed)*

---

## Consolidated Open Items Tracker

| # | Open Item | Owner(s) | Category | Status |
|---|-----------|----------|----------|--------|
| 1 | Secondary CA model for inbound container processing | Owen, Abhijeet | SWIFT-v1 | Open |
| 2 | Tag & consolidate SWIFT questions in AzDASH | Osman | SWIFT-v1 | Open |
| 3 | Offline sync with SWIFT team (Vipin, Tamilmani) | Abhijeet, Sai | SWIFT-v1 | Open |
| 4 | Diagnostic APIs — AzDASH vs separate interface | Osman, Trilok | AzDASH | Open |
| 5 | Diagnostics proxy through VCPA vs remote | Osman | AzDASH | Open |
| 6 | Production route table size stats | Owen, Trilok, Chaitanya | ORT | Open |
| 7 | LPM vs priority-based ORT rules for SLB | Osman | ORT/SLB | In Discussion |
| 8 | gRPC vs FlatBuffers + SHM for VDPA interface | Osman | Mappings | Open |
| 9 | Non-VXLAN/NVGRE encap requirements | VNET | Encapsulation | Open |
| 10 | Inbound traffic model finalization | Owen, Abhijeet | Data Path | Open |
| 11 | Exhaustive DSCP/VTAP/policy rule list | Chaitanya Raje | Security | Action Item |
| 12 | VTAP mode — appliance vs host | Pranjal (raised) | Security | Open |
| 13 | CID ↔ ENI/MAC mapping approach | TBD | Metering | Open |
| 14 | LM deep dive with SLB + HostNet | Sai (scheduling) | Live Migration | Open |
| 15 | Multi-VNET peered container behavior | VNET | Containers | Open |
| 16 | Lab access for agent bootstrapping | Vivek Bhanu | Infra | Open |

---

## Key Architectural Principles Emerging

1. **Minimize runtime payloads** → cookie-based programming, not full objects
2. **Prefer shared/global objects** over ENI-local duplication (node-global mappings)
3. **Deterministic IDs** → enable stateless rehydration and robust LM
4. **Optimize local interfaces** → FlatBuffers + lightweight transport over gRPC
5. **Unified ORT** → avoid VFP-style layer explosion; combine VNET + SLB where possible
6. **Immutable ENI type** → prevent post-create type changes that break traffic
7. **AzDASH Forum as source of truth** → async progress between offsites
