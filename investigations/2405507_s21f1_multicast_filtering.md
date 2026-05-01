# ADO #2405507 — S21F1 Multicast Filtering Analysis

## Problem Statement

S21F1 NIC (funeth driver) accepts **all multicast packets** by default. It should only accept multicast traffic for specific multicast groups the host/application is interested in. Compared to Mellanox NICs which only accept standard multicast MACs (224.0.0.1 All Hosts, 224.0.0.251/252 mDNS), S21F1 receives everything including SSDP and other unwanted multicast traffic.

---

## Root Cause

The funeth Windows NDIS miniport driver **does not implement multicast filtering** at all. The FunOS side also has no funeth-specific multicast filtering infrastructure.

### Evidence

1. **`MaxMulticastListSize = 0`** (`miniport.c:1189`) — tells Windows the NIC cannot filter specific multicast addresses.
2. **No multicast list OIDs** — `OID_802_3_MULTICAST_LIST`, `OID_802_3_MAXIMUM_LIST_SIZE`, `OID_802_3_ADD/DELETE_MULTICAST_ADDRESS` are missing from the supported OID list (`miniport.c:79-115`).
3. **All-or-nothing filtering** — `SupportedPacketFilters` (`miniport.c:1186-1188`) includes `NDIS_PACKET_TYPE_ALL_MULTICAST` but NOT `NDIS_PACKET_TYPE_MULTICAST` (selective).
4. **Driver explicitly acknowledges the gap** — `funeth_rx.c:691-693`:
   ```c
   // We set NDIS_RECEIVE_FLAGS_PERFECT_FILTERED since it is the common
   // case. However, the device lacks multicast filtering. If we received
   // any multicasts they have not been filtered.
   ```
5. **FunOS funeth RX path** (`vi.c:1838-1875`) only counts multicast stats — no filtering. The MANA BNIC multicast filtering code (`vi.c:664-675`) is MANA-specific and doesn't apply to funeth.

---

## How MANA (BasicNic) Implements Multicast Filtering

The MANA Windows driver uses a completely different approach from legacy multicast lists.

### Architecture: NDIS Receive Filters (Modern Path)

MANA uses **NDIS Receive Filters** (`OID_RECEIVE_FILTER_SET_FILTER`) — the SR-IOV/VMQ model — rather than legacy `OID_802_3_MULTICAST_LIST`. This is a per-MAC filter registration via GDMA commands to FunOS.

### Flow

```
1. Windows: OID_RECEIVE_FILTER_SET_FILTER (with MAC = 01:00:5E:00:00:FB for mDNS)
   ↓
2. HandleReceiveFilterSet() — FilterOids.c:554
   - Parses NDIS_RECEIVE_FILTER_PARAMETERS
   - Extracts destination MAC via ParseNdisReceiveFilterFields() (line 273)
   ↓
3. RegisterNdisFilter() — FilterOids.c:200
   - Calls PFRegisterFilter() with the MAC address
   ↓
4. PFRegisterFilter() — PFInterface.c:227
   - Builds REGISTER_FILTER_REQUEST (PrivilegedCommands.h:8-20)
   - Sends GDMA command via SendMessageToPFClient()
   ↓
5. FunOS BNIC: Programs hardware RX filter table with the MAC
   ↓
6. Hardware: Only packets matching filter entries are delivered to host
```

### Key Data Structure (PrivilegedCommands.h:8-20)

```c
typedef struct _REGISTER_FILTER_REQUEST {
    GDMA_REQUEST_HEADER      Header;
    BASIC_NIC_HANDLE         VportHandle;
    UINT8                    MacAddress[6];    // The multicast MAC to filter
    BOOLEAN                  AllowAnyVlanTag;
    BOOLEAN                  MatchInnerMacTni;
    BOOLEAN                  RequireVlanTag;
    BOOLEAN                  IsExceptionFilter;
    UINT16                   Vlan;
    UINT32                   Tni;
    UINT32                   Reserved2;
    UINT32                   Reserved3;
} REGISTER_FILTER_REQUEST;
```

### MANA Design Choices

| Aspect | MANA BasicNic |
|--------|--------------|
| `MaxMulticastListSize` | 0 (intentional — legacy multicast list not used) |
| Multicast OIDs | None (uses Receive Filter OIDs instead) |
| Filter mechanism | Per-MAC GDMA REGISTER_FILTER_REQUEST to FunOS |
| `allow_all_multicast` flag | NOT sent to firmware — per-address filtering only |
| Packet filter | `DIRECTED \| MULTICAST \| BROADCAST \| ALL_MULTICAST` |
| Filter lifecycle | Add via `OID_RECEIVE_FILTER_SET_FILTER`, remove via `OID_RECEIVE_FILTER_CLEAR_FILTER` |

---

## funeth vs MANA Comparison

| Aspect | MANA (BasicNic) | funeth |
|--------|-----------------|--------|
| **NIC Switch / SR-IOV** | ✅ Yes | ❌ No |
| **Receive Filter OIDs** | ✅ `OID_RECEIVE_FILTER_SET_FILTER` | ❌ Not supported |
| **Legacy Multicast OIDs** | ❌ Not used | ❌ Not implemented |
| **MaxMulticastListSize** | 0 (intentional) | 0 (not implemented) |
| **SupportedPacketFilters** | `DIRECTED \| MULTICAST \| BROADCAST \| ALL_MULTICAST` | `DIRECTED \| ALL_MULTICAST \| BROADCAST` |
| **Per-MAC multicast filtering** | ✅ Via GDMA filter commands | ❌ None |
| **FunOS-side filtering** | ✅ BNIC filter table + `allow_all_multicast` flag | ❌ Stats only, no filtering |
| **Driver acknowledges gap** | N/A | ✅ "device lacks multicast filtering" (funeth_rx.c:692) |
| **Control path to FunOS** | GDMA privileged commands | Admin queue (funeth_ctrl_dev.c) |

---

## Windows Multicast Filtering — Standard NDIS Model

### Packet Filter Flags (OID_GEN_CURRENT_PACKET_FILTER)

| Flag | Value | Meaning |
|------|-------|---------|
| `NDIS_PACKET_TYPE_DIRECTED` | 0x0001 | Unicast packets for this NIC's MAC |
| `NDIS_PACKET_TYPE_MULTICAST` | 0x0002 | **Selective** multicast — only MACs in the programmed list |
| `NDIS_PACKET_TYPE_ALL_MULTICAST` | 0x0004 | Accept **ALL** multicast packets (no filtering) |
| `NDIS_PACKET_TYPE_BROADCAST` | 0x0008 | Broadcast packets |
| `NDIS_PACKET_TYPE_PROMISCUOUS` | 0x0020 | All packets regardless of DMAC |

### Required OIDs for Legacy Multicast Filtering

| OID | Type | Purpose |
|-----|------|---------|
| `OID_802_3_MULTICAST_LIST` | Set/Query | Set/query the list of multicast MAC addresses to accept |
| `OID_802_3_MAXIMUM_LIST_SIZE` | Query | Report max supported multicast list entries |
| `OID_802_3_ADD_MULTICAST_ADDRESS` | Set | Add a single multicast MAC (optional) |
| `OID_802_3_DELETE_MULTICAST_ADDRESS` | Set | Remove a single multicast MAC (optional) |

### Windows Behavior Based on Driver Capabilities

| Driver reports | Windows behavior |
|----------------|------------------|
| `MaxMulticastListSize = 0` + no `NDIS_PACKET_TYPE_MULTICAST` | Falls back to `ALL_MULTICAST` — all multicast traffic accepted |
| `MaxMulticastListSize = N` + `NDIS_PACKET_TYPE_MULTICAST` | Programs multicast MAC list via `OID_802_3_MULTICAST_LIST`; falls back to `ALL_MULTICAST` if list exceeds N |
| NIC Switch + Receive Filters (MANA model) | Uses `OID_RECEIVE_FILTER_SET_FILTER` per-MAC |

### How to View Multicast State on Windows

```powershell
# Multicast group memberships (IP-level — what the OS wants to filter)
netsh int ipv4 show joins
netsh int ipv6 show joins

# NDIS Receive Filters (only for NICs with NIC Switch, e.g., MANA)
Get-NetAdapterReceiveFilter -Name "MANA"

# Adapter properties
Get-NetAdapterAdvancedProperty -Name "funeth" | Where-Object { $_.DisplayName -like "*Multicast*" }
```

---

## Implementation Options for funeth

### Option A: Legacy Multicast List (OID_802_3_MULTICAST_LIST)

**Windows driver changes:**
1. Set `MaxMulticastListSize` to 32-64 (typical NIC value)
2. Add `NDIS_PACKET_TYPE_MULTICAST` to `SupportedPacketFilters`
3. Add `OID_802_3_MULTICAST_LIST` and `OID_802_3_MAXIMUM_LIST_SIZE` to supported OID list
4. Implement OID handlers — maintain multicast MAC list in `FUN_NET_DEVICE`
5. Update `FunethPktTypeAllowed()` to check incoming DMAC against the list when `NDIS_PACKET_TYPE_MULTICAST` is set
6. Send multicast list to FunOS via admin queue when list changes

**FunOS changes:**
1. Add funeth multicast MAC filter table
2. Handle admin queue command for multicast list updates
3. Filter multicast in funeth RX path — drop non-matching multicast unless `ALL_MULTICAST` mode

**Pros:** Standard NDIS approach, well-understood by Windows networking team
**Cons:** Legacy model, list size limited, fallback to ALL_MULTICAST when exceeded

### Option B: FunOS-side Only Filtering

**Windows driver changes:**
- Minimal: send multicast join/leave notifications via admin queue
- Or: send the packet filter flags to FunOS and let FunOS decide

**FunOS changes:**
1. Track multicast group memberships
2. Filter in FunOS RX path before delivering to host
3. Use NU hardware capabilities if available

**Pros:** Filtering happens on DPU, reduces PCIe bandwidth
**Cons:** Requires FunOS command protocol extension, harder to test

### Option C: Both (Defense in Depth)

Implement Option A + Option B. Windows driver filters in software, FunOS filters in hardware. Belt and suspenders.

**Pros:** Maximum filtering effectiveness
**Cons:** More code, more complexity

---

## Key Code Files

### Windows funeth Driver (Storage-xDpuWindows)

| File | Key Lines | Content |
|------|-----------|---------|
| `src/funeth/miniport.c` | 79-115 | Supported OID list (missing multicast OIDs) |
| `src/funeth/miniport.c` | 1186-1189 | `SupportedPacketFilters` and `MaxMulticastListSize=0` |
| `src/funeth/miniport.c` | 3014-3025 | `FunEthSetCurrentPacketFilter()` handler |
| `src/funeth/funeth_rx.c` | 426-439 | `FunethPktTypeAllowed()` — software packet filter |
| `src/funeth/funeth_rx.c` | 550-558 | RX path: packet filter check and drop |
| `src/funeth/funeth_rx.c` | 691-693 | Comment: "device lacks multicast filtering" |
| `src/funeth/miniport.h` | 248 | `PktFilter` field in `FUN_NET_DEVICE` |

### MANA BasicNic Driver (SmartNIC-SW-GDMA)

| File | Key Lines | Content |
|------|-----------|---------|
| `src/BasicNic/Driver/Adapter.c` | 1035 | `MaxMulticastListSize = 0` (intentional) |
| `src/BasicNic/Driver/FilterOids.c` | 200 | `RegisterNdisFilter()` — calls `PFRegisterFilter` |
| `src/BasicNic/Driver/FilterOids.c` | 273 | `ParseNdisReceiveFilterFields()` — extracts MAC |
| `src/BasicNic/Driver/FilterOids.c` | 554 | `HandleReceiveFilterSet()` — OID entry point |
| `src/BasicNic/Driver/FilterOids.c` | 694 | `HandleReceiveFilterClear()` — filter removal |
| `src/BasicNic/Driver/PFInterface.c` | 227 | `PFRegisterFilter()` — GDMA command to FunOS |
| `src/BasicNic/Driver/PrivilegedCommands.h` | 8-20 | `REGISTER_FILTER_REQUEST` structure |
| `src/BasicNic/Driver/NicConfigurations.h` | 47-51 | `ADAPTER_FILTER_OPTIONS` packet filter flags |

### FunOS

| File | Key Lines | Content |
|------|-----------|---------|
| `networking/ethernet/vi.c` | 664-685 | MANA BNIC multicast filtering (not used by funeth) |
| `networking/ethernet/vi.c` | 1838-1875 | `funeth_lport_rx_pkts_bytes_inc()` — stats only |
| `functions/mana/mana_bnic.c` | 1233-1238 | BNIC control register init |
| `functions/mana/mana_bnic.h` | 247-251 | `broadcast_rx_object_enable`, `allow_all_multicast`, `promiscuous_receive` flags |
| `sdk_include/FunOS/networking/core/nfm/nfm_bnic_api.h` | 750-761 | BNIC control flag documentation |

---

## Next Steps

1. **Decide on implementation approach** (Option A, B, or C)
2. **Define multicast list size** — what `MaxMulticastListSize` to support
3. **Check if FunOS NU hardware has multicast hash/filter capabilities** for funeth flows
5. **Get `netsh int ipv4 show joins` from a live S21F1 system** to see actual multicast groups Windows subscribes to
