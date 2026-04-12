# VFP Proxy Protocol V2 — Design Specification

**Component:** Virtual Filtering Platform (VFP) — Azure SDN Datapath  
**Feature:** HAProxy Proxy Protocol V2 Injection  
**Status:** Implemented (Production)  
**Last Updated:** 2026-03-21

---

## 1. Overview

VFP implements transparent **HAProxy Proxy Protocol v2** insertion into TCP connections. This allows load balancers and SDN infrastructure to convey original client connection metadata (source/destination IPs and ports) to backend servers without requiring application-layer changes.

The proxy protocol v2 data is injected as the **first TCP payload** after the three-way handshake, using a novel sequence number manipulation technique that is invisible to both client and server endpoints.

### 1.1 Key Properties

| Property | Value |
|----------|-------|
| Protocol | TCP only (not UDP) |
| Standard | HAProxy Proxy Protocol v2 (binary format) |
| Injection Point | First payload after TCP 3-way handshake |
| Endpoint Awareness | Transparent — neither client nor server is aware |
| Hardware Offload | Flow offloaded to GFT/NIC after proxy exchange completes |
| Feature Flag | `VFX_NAT_RULE_FLAG_PROXY_V2` (0x4000) on NAT rules |

---

## 2. Protocol Format

### 2.1 Proxy Protocol V2 Binary Layout

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                    Signature (12 bytes)                        +
|                 0D 0A 0D 0A 00 0D 0A 51 55 49 54 0A          |
+               +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|               | Ver | Command |    Family     |    Length     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Address Block (variable)                    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    TLV Data (variable)                         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

### 2.2 Structures

```c
// 16-byte proxy protocol header
typedef struct _VFP_PROXY_BUFFER_HEADER {
    UCHAR  Signature[12];      // 0x0D 0A 0D 0A 00 0D 0A 51 55 49 54 0A
    UCHAR  VersionCommand;     // 0x21 = PROXY v2, 0x20 = LOCAL
    UCHAR  Family;             // 0x11 = IPv4/TCP, 0x21 = IPv6/TCP
    USHORT Length;              // Size of address block + TLV data
} VFP_PROXY_BUFFER_HEADER;

// IPv4 address block (12 bytes)
typedef struct _VFP_PROXY_BUFFER_IPV4_ADDRESS {
    VMS_IPV4_ADDR SourceAddress;
    VMS_IPV4_ADDR DestinationAddress;
    USHORT        SourcePort;
    USHORT        DestPort;
} VFP_PROXY_BUFFER_IPV4_ADDRESS;

// IPv6 address block (36 bytes)
typedef struct _VFP_PROXY_BUFFER_IPV6_ADDRESS {
    VMS_IPV6_ADDR SourceAddress;
    VMS_IPV6_ADDR DestinationAddress;
    USHORT        SourcePort;
    USHORT        DestPort;
} VFP_PROXY_BUFFER_IPV6_ADDRESS;

// TLV extension (variable length)
typedef struct _VFP_PROXY_BUFFER_TLV {
    UCHAR Type;
    UCHAR LengthHigh;
    UCHAR LengthLow;
    UCHAR Value[0];            // Variable-length
} VFP_PROXY_BUFFER_TLV;
```

### 2.3 Buffer Size Calculation

```
L = sizeof(Header)                              // 16 bytes
  + sizeof(AddressBlock)                         // 12 (IPv4) or 36 (IPv6)
  + TLVBufferLength                              // 0–138 bytes
  + (CRC32TLV ? 7 : 0)                          // Optional CRC32 TLV
```

**Maximum TLV buffer**: 138 bytes (`VFX_PROXY_PROTOCOL_V2_TLV_BUFFER_MAX_SIZE`)

**Typical total sizes**:
- IPv4 without TLVs: 28 bytes
- IPv4 with CRC32: 35 bytes
- IPv6 with full TLVs + CRC32: ~191 bytes

---

## 3. Sequence Number Manipulation

### 3.1 Core Technique

VFP creates space in the TCP sequence number space by **subtracting** the proxy buffer length (L) from the client's initial sequence number. This reserves L bytes of sequence space for the proxy payload injection.

Two complementary adjustments maintain TCP correctness:

| Direction | Adjustment | Purpose |
|-----------|-----------|---------|
| Forward (client → server) | `seq -= L` | Reserve sequence space for proxy payload |
| Reverse (server → client) | `ack += L` | Hide the reservation from the client |

### 3.2 Adjustment Function

```c
VOID VfpUpdatePacketForProxyProtocol(
    ULONG                       ProxyBufferLength,  // L
    VFP_PROXY_PACKET_ADJUSTMENT AdjustmentType,
    BOOLEAN                     RecalculateChecksum,
    PVMS_TCP_HEADER             TcpHeader
);
```

- **SeqNum adjustment**: `TcpHeader->SequenceNumber -= ProxyBufferLength`
- **AckNum adjustment**: `TcpHeader->AckNumber += ProxyBufferLength`
- **Checksum**: Differential recalculation using old/new values
- **Window**: Set to 0 during proxy exchange (flow control suppression)

### 3.3 Self-Correcting Arithmetic

After the server acknowledges the proxy payload:

```
Server receives proxy data at seq = A-L, length = L bytes
Server ACKs with ack = (A-L) + L = A
```

The sequence space is **exactly restored** to the client's original value. No permanent translation offset exists — this is the key design insight that avoids ongoing per-packet adjustment.

---

## 4. TCP State Machine

### 4.1 Extended SYN States

```c
typedef enum _VFP_SYN_STATE {
    VfpSynStateUnknown = 0,
    VfpSynStateClosed,
    VfpSynStateSynSent,                 // SYN sent, awaiting SYN-ACK
    VfpSynStateOpen,                    // Connection fully established
    VfpSynStateProxySent,               // Proxy payload injected, awaiting ACK
    VfpSynStateReadyToSendProxyData     // Handshake done, ready to inject proxy
} VFP_SYN_STATE;
```

### 4.2 State Transitions

```
                    ┌──────────────────────────────────────────────┐
                    │           OUTBOUND FLOW (Client)             │
                    │                                              │
                    │  Closed ──SYN──→ SynSent                    │
                    │                    │                          │
                    │             recv SYN-ACK                     │
                    │                    ↓                          │
                    │           ReadyToSendProxyData               │
                    │                    │                          │
                    │          inject proxy packet                  │
                    │                    ↓                          │
                    │              ProxySent                        │
                    │                    │                          │
                    │           recv ProxyAck                       │
                    │                    ↓                          │
                    │                 Open ──→ [OFFLOAD TO HW]     │
                    └──────────────────────────────────────────────┘
```

### 4.3 Adjustment Gating Condition

Seq/ack adjustments are applied **only** when this condition is true:

```c
if (tcpPacket->TcpHeader->SYN ||
    ((pairFlow->SynState != VfpSynStateOpen &&
      pairFlow->SynState != VfpSynStateProxySent) ||
     (paired && reverseFlow->SynState != VfpSynStateOpen &&
      reverseFlow->SynState != VfpSynStateProxySent)))
```

**Translation**: Adjustments happen on SYN packets OR when either flow has not yet reached Open/ProxySent. Once **both** flows are in `VfpSynStateOpen`, all adjustments cease permanently.

---

## 5. Complete Packet Flow

### 5.1 Annotated Packet Sequence

```
 Client (A)              VFP/SDN                        Server (B)
 ──────────              ───────                        ──────────

 ① SYN seq=A ack=0  ──→  adjust seq   ──→  SYN seq=A-L ack=0 win=0
    [original]            [seq -= L]        [server sees reduced ISN]

 ② SYNACK seq=B     ←──  adjust ack   ←──  SYNACK seq=B ack=A-L
    ack=A win=Y           [ack += L]        [server ACKs adjusted ISN]
    [client sees                            [original]
     original ack]

 ③ ACK seq=A ack=B  ──→  [DROP]            (never reaches server)
    [client's final                         
     handshake ACK]                         

 ④                        [INJECT]    ══→  ACK seq=A-L ack=B win=24
                          proxy packet      payload=[Proxy v2 data, L bytes]
                                            [server receives proxy info]

 ⑤ ACK seq=B ack=A  ←──  [rewrite]   ←──  ACK seq=B ack=A win=Y
    [ProxyAck]            [ack += L         [server ACKs: (A-L)+L = A]
     but ack already       naturally = A]
     correct]

 ⑥                        [INJECT]    ══→  ACK seq=A ack=B win=X
                          window restore    [restores real window to server]

 ─── Connection OPEN ─── Flow OFFLOADED to GFT/NIC ─── No more adjustments ───
```

### 5.2 Phase Details

| Phase | Packet | Key Action | Window |
|-------|--------|-----------|--------|
| ① SYN | Client → Server | `seq -= L` | Set to 0 |
| ② SYN-ACK | Server → Client | `ack += L` | Set to 0 |
| ③ Client ACK | Client → VFP | **Dropped** by VFP | — |
| ④ Proxy Inject | VFP → Server | New packet with proxy v2 payload | 24 (small) |
| ⑤ Proxy ACK | Server → Client | Triggers `ProxyAck`, state → Open | Original |
| ⑥ Window Restore | VFP → Server | Restores real TCP window | Original |

---

## 6. Window Management

During the proxy exchange, VFP suppresses data transfer by setting the TCP window to 0:

| Phase | Window Value | Reason |
|-------|-------------|--------|
| SYN/SYN-ACK | **0** | Prevent data before proxy injection |
| Proxy packet | **24** | Minimal window during proxy handshake |
| After ProxyAck | **Original** | `VfpGetMaxTcpWindow()` restores real value |
| Window restore ACK | **Scaled original** | `pairFlow->Window >> reverseFlow->WindowScale` |

This ensures neither endpoint sends application data until the proxy protocol exchange is fully complete.

---

## 7. Configuration

### 7.1 NAT Rule Flag

Proxy v2 is enabled per-NAT-rule via the flag:

```c
#define VFX_NAT_RULE_FLAG_PROXY_V2  0x4000   // Decimal: 16384
```

### 7.2 Proxy Info Structure

```c
typedef struct _VFX_PROXY_PROTOCOL_V2_INFO {
    BOOLEAN        Proxy;               // TRUE=PROXY command, FALSE=LOCAL
    VFX_IP_ADDRESS SrcAddr;             // Source IP (may differ from packet)
    VFX_IP_ADDRESS DestAddr;            // Destination IP
    USHORT         SrcPort;             // Source port
    USHORT         DestPort;            // Destination port
    VFX_TRANSPOSITION_PARTIAL_REWRITE PartialRewrites[4];  // IP byte rewrites
    UCHAR          TLVBuffer[138];      // Optional TLV extensions
    USHORT         TLVBufferLength;     // Actual TLV length
    BOOLEAN        CRC32TLV;            // Append CRC32 checksum TLV
} VFX_PROXY_PROTOCOL_V2_INFO;
```

### 7.3 vfpctrl Rule Syntax

```
vfpctrl /rule add ... /flags 16384
    /proxy [Proxy] [SrcIP] [DstIP] [SrcPort] [DstPort]
    /tlv [TLVBufferLength] [CRC32TLV]
    /partialrewrite [Index] [Offset] [Byte]   (up to 4)
```

### 7.4 Partial Rewrites

Up to 4 byte-level transposition operations can rewrite IP address bytes into the TLV data. Each rewrite specifies an offset and byte value to place at that offset in the proxy buffer.

---

## 8. Hardware Offload

### 8.1 Offload Trigger

After the proxy ACK is received, the flow is offloaded to GFT (Generic Flow Tables) / NIC hardware:

```c
attemptTcpFlowOffload = ... || ProxyAck;   // ProxyAck triggers offload
```

### 8.2 Offload Constraint

Proxy flows are **not offloaded during the handshake** because:
- Seq/ack adjustments require per-packet software processing
- Exact sequence number matching is needed for state transitions
- Window manipulation requires software control

Once in `VfpSynStateOpen`, no further software adjustment is needed, making the flow safe for hardware offload.

### 8.3 Rule Offload Conversion

Proxy info is serialized for hardware offload APIs via:

```c
ConvertProxyInfoV2()    // Current format → RO_API_PROXY_PROTOCOL_INFO_V2
ConvertProxyInfo_v1()   // Legacy v1 format for backward compatibility
```

---

## 9. Unified Flow Integration

### 9.1 Flow Fields

```c
// In VFP_UNIFIED_FLOW / process context:
BOOLEAN           proxy;                  // Connection uses proxy protocol
BOOLEAN           proxyAck;               // Server has ACKed proxy data
PVFP_PROXY_BUFFER ProxyProtocolBuffer;    // Allocated proxy buffer
```

### 9.2 Buffer Lifecycle

1. **Allocation**: `VfpConstructProxyProtocol()` in `Handler.c` — allocates ETH+IP+TCP headers + proxy data
2. **Assignment**: Attached to unified flow's `ProxyProtocolBuffer` field
3. **Injection**: `VfpQueueProxyPacket()` uses the buffer to construct and inject the TCP packet
4. **Cleanup**: Buffer freed after flow teardown or when proxy exchange completes

### 9.3 Data Structure Versioning

The `ProxyProtocolBuffer` field is preserved across unified flow data structure versions:
- **v1**: `VfpUnifiedFlow-v1.h` (line 209)
- **v2**: `VfpUnifiedFlow-v2.h` (line 209)
- **v2→v3 conversion**: `v3conversion.c` (line 459)

---

## 10. Diagnostics

### 10.1 vfpctrl Output

`PrintProxyInfo()` displays:
- Proxy/Local command bit
- Source and destination IPs and ports
- Partial rewrite details (up to 4)
- TLV buffer contents (type/length/value)
- CRC32TLV flag

### 10.2 JSON Output

`JsonWriteProxyInfo()` serializes all proxy info fields to JSON for diagnostics and telemetry.

---

## 11. Source Files

| File | Role |
|------|------|
| `common/include/VmsVfpCore.h` | Core structures: `VFX_PROXY_PROTOCOL_V2_INFO`, NAT rule data, feature flag |
| `common/include/VfpFilterCore.h` | Buffer structures: `VFP_PROXY_BUFFER*`, `VFP_SYN_STATE` enum, flow fields |
| `common/src/Handler.c` | `VfpConstructProxyProtocol()` — builds proxy v2 binary buffer |
| `common/src/StateTracking.c` | `VfpUpdatePacketForProxyProtocol()`, `VfpExtractTcpPacketInfo()` — seq/ack adjustment + state machine |
| `common/src/ProcessCore.c` | Proxy buffer assignment, cleanup, serialization in unified flow |
| `common/src/unifiedflow.c` | `VfpQueueProxyPacket()`, `VfpQueueAckPacket()` — packet injection |
| `common/src/NatPoolCore.c` | Dynamic NAT proxy integration |
| `common/src/MapObjCore.c` | Map-based NAT proxy integration |
| `common/src/ruleoffload/VfpROBPayloadConversion.c` | `ConvertProxyInfoV2()` — hardware offload conversion |
| `common/src/ruleoffload/VfpROBPayloadConversion_v1.c` | `ConvertProxyInfo_v1()` — legacy offload format |
| `common/control/vfpctrl/vfpctrlCore.c` | `PrintProxyInfo()` — diagnostic output |
| `common/control/vfpctrl/vfpjson.c` | `JsonWriteProxyInfo()` — JSON diagnostics |
| `linux/control/vfpctrl/include/vfpctrl.h` | Linux control plane documentation |
| `windows/src/control/vfpctrl/main.c` | Windows diagnostic output |
| `windows/src/inc/VfpFilter.h` | Windows-specific proxy buffer field |
| `linux/unittest/unifiedflow/unifiedflow_test.c` | Unit tests |

---

## 12. Security Considerations

- Proxy protocol data is **not authenticated** — the server trusts VFP to provide accurate client information
- CRC32 TLV provides **integrity** (not authentication) for the proxy payload
- Window suppression (set to 0) prevents data injection races during the proxy exchange
- Exact seq/ack matching during the proxy handshake prevents spoofed ACK attacks from triggering premature state transitions

---

## 13. Limitations

1. **TCP only** — proxy protocol v2 is not supported for UDP flows
2. **Max TLV size**: 138 bytes — exceeding this is silently truncated
3. **Max partial rewrites**: 4 — additional rewrites are not supported
4. **Not offloaded during handshake** — proxy flows remain in software path until ProxyAck
5. **Single injection** — proxy data is sent once; retransmission relies on TCP retransmit of the injected packet via the `VfpSynStateProxySent` re-injection path
