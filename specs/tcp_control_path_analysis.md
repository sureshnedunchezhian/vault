# FunOS TCP Control Path Analysis

## Overview

This document analyzes the TCP control path in FunOS for both client and server sides, including
interactions with the application layer (`tcp_pkt_gen.c` client and `tcp_pkt_rcv.c` server).
It covers Work Unit (WU) handlers, flow directions, VP queue assignments, and the complete
connection establishment and teardown sequences.

---

## Architecture: VP Layers and Flow Directions

FunOS TCP uses a **multi-VP architecture** with bidirectional flows connecting the application
layer to the TCP stack. Each socket connection has a flow pair:

| Side   | Outbound Flow (App→TCP) | Inbound Flow (TCP→App) |
|--------|-------------------------|------------------------|
| Client | `cton_f` (client-to-network) | `ntoc_f` (network-to-client) |
| Server | `ston_f` (server-to-network) | `ntos_f` (network-to-server) |

TCP internally uses a 3-VP model per connection:
- **FSM/Entry VP** — handles socket API entry and FSM transitions (high-priority queue)
- **XMT VP** — handles packet transmission (same VP as FSM for efficiency)
- **RCV VP** — handles packet reception from the network

```mermaid
graph TB
    subgraph "Application Layer"
        APP_CLIENT["tcp_pkt_gen<br/>(Client App VP)"]
        APP_SERVER["tcp_pkt_rcv<br/>(Server App VP)"]
    end

    subgraph "Socket Layer (socket.c)"
        SOCK_DISPATCH["socket_flow_dispatch_func()<br/>Opcode → WUID mapping"]
        SOCK_HTON["socket_hton_flow<br/>(host-to-network)"]
        SOCK_NTOH["socket_ntoh_flow<br/>(network-to-host)"]
    end

    subgraph "TCP Stack"
        TCP_FSM["TCP FSM VP<br/>(tcp_fsm.c)<br/>High-Priority Queue"]
        TCP_XMT["TCP XMT VP<br/>(tcp_xmt.c)<br/>Same VP as FSM"]
        TCP_RCV["TCP RCV VP<br/>(tcp_rcv.c)"]
    end

    subgraph "Network"
        LE["Lookup Engine (LE)<br/>FAE Entry"]
        NIC["NIC / Network"]
    end

    APP_CLIENT -- "cton_f" --> SOCK_DISPATCH
    APP_SERVER -- "ston_f" --> SOCK_DISPATCH
    SOCK_DISPATCH --> SOCK_HTON
    SOCK_HTON -- "flow_bind()" --> TCP_FSM

    TCP_FSM --> TCP_XMT
    TCP_XMT -- "TX packets" --> NIC
    NIC -- "RX packets<br/>(LE hit)" --> TCP_RCV
    TCP_RCV --> TCP_FSM

    TCP_FSM --> SOCK_NTOH
    SOCK_NTOH -- "ntoc_f" --> APP_CLIENT
    SOCK_NTOH -- "ntos_f" --> APP_SERVER
```

---

## TCP State Machine

The TCP FSM (`tcp_fsm.c`) defines 13 states:

```c
enum tcp_state {
    TCP_FSM_STATE_INVALID,       // 0 - Initial
    TCP_FSM_STATE_ACCEPT,        // 1 - Pending accept
    TCP_FSM_STATE_CLOSED,        // 2 - Closed
    TCP_FSM_STATE_LISTEN,        // 3 - Listening
    TCP_FSM_STATE_SYNSENT,       // 4 - SYN sent (active open)
    TCP_FSM_STATE_SYNRCVD,       // 5 - SYN received (passive open)
    TCP_FSM_STATE_CLOSING,       // 6 - Both sides sent FIN
    TCP_FSM_STATE_LASTACK,       // 7 - Waiting for last ACK
    TCP_FSM_STATE_TIMEWAIT,      // 8 - TIME_WAIT
    TCP_FSM_STATE_ESTABLISHED,   // 9 - Established (fast-path start)
    TCP_FSM_STATE_FINWAIT1,      // 10 - FIN sent, awaiting ACK
    TCP_FSM_STATE_FINWAIT2,      // 11 - FIN ACK'd, awaiting peer FIN
    TCP_FSM_STATE_CLOSEWAIT,     // 12 - Received FIN, awaiting app close
};
```

```mermaid
stateDiagram-v2
    [*] --> INVALID
    INVALID --> CLOSED : tcp_fsm_host_connect_init()
    INVALID --> LISTEN : tcp_fsm_host_listen()

    CLOSED --> SYNSENT : tcp_fsm_host_connect()<br/>Send SYN

    LISTEN --> SYNRCVD : tcp_netw_recvseg_listen()<br/>Recv SYN, Send SYN-ACK

    SYNSENT --> ESTABLISHED : tcp_fsm_netw_recvseg_in_active()<br/>Recv SYN-ACK, Send ACK

    SYNRCVD --> ESTABLISHED : tcp_fsm_host_accept()<br/>Recv ACK of SYN-ACK

    ESTABLISHED --> FINWAIT1 : tcp_fsm_host_disconnect()<br/>Send FIN
    ESTABLISHED --> CLOSEWAIT : tcp_fsm_netw_recvseg_in_active()<br/>Recv FIN, Send ACK

    FINWAIT1 --> FINWAIT2 : Recv ACK of FIN
    FINWAIT1 --> CLOSING : Recv FIN (simultaneous close)
    FINWAIT1 --> TIMEWAIT : Recv FIN+ACK

    FINWAIT2 --> TIMEWAIT : Recv FIN, Send ACK

    CLOSEWAIT --> LASTACK : tcp_fsm_host_disconnect()<br/>Send FIN

    CLOSING --> TIMEWAIT : Recv ACK of FIN

    LASTACK --> CLOSED : Recv ACK of FIN

    TIMEWAIT --> CLOSED : Timeout (2MSL)

    ESTABLISHED --> CLOSED : RST (tcp_fsm_host_abort)
    SYNSENT --> CLOSED : RST / Timeout
    SYNRCVD --> CLOSED : RST / Timeout
```

---

## Connection Establishment: Client (Active Open)

### Sequence of WU Handlers

```mermaid
sequenceDiagram
    participant dpcsh as dpcsh<br/>(JSON cmd)
    participant AppVP as App VP<br/>(tcp_pkt_gen)
    participant SockVP as Socket Layer<br/>(socket.c)
    participant TCPVP as TCP FSM VP<br/>(tcp.c / tcp_fsm.c)
    participant LE as Lookup Engine
    participant Net as Network

    Note over dpcsh: User issues tcp_pkt_gen command

    dpcsh->>AppVP: CHANNEL_HANDLER(tcp_pkt_gen)<br/>TOP_LEVEL, parse JSON config

    AppVP->>AppVP: tcp_pkt_tx_connect_numclients()<br/>Loop over clients

    AppVP->>AppVP: tcp_pkt_tx_connect_flow_allocated()<br/>flow_init(cton_f, ntoc_f)

    AppVP->>SockVP: tcp_host_create_req_push()<br/>Create TCP socket

    SockVP-->>AppVP: tcp_pkt_tx_connect_socket_allocated()<br/>Returns socketid, flows

    Note over AppVP: flow_set_dispatch_func(ntoc_f)<br/>tcp_host_flow_bind(socketid, cton_f, ntoc_f)

    AppVP->>SockVP: socket_host_connect_req_push()<br/>(sport, dport, saddr, daddr)

    SockVP->>TCPVP: tcp_host_connect_req()<br/>via flow_bind(hton_f)

    Note over TCPVP: TCP_WORKERPOOL_HACK:<br/>VP rewrite for affinity

    TCPVP->>TCPVP: tcp_fsm_host_connect_init()<br/>Store addrs, set state=CLOSED

    TCPVP->>LE: LE add entry<br/>(5-tuple → tcp_netw_recvseg_active)

    LE-->>TCPVP: tcp_host_connect_req_le_added()<br/>LE success callback

    TCPVP->>TCPVP: tcp_fsm_host_connect()<br/>state: CLOSED→SYNSENT

    TCPVP->>Net: tcp_xmt_syn()<br/>Send SYN packet

    Note over TCPVP: Start RTO timer

    Net-->>TCPVP: SYN-ACK received<br/>tcp_netw_recvseg_active()

    TCPVP->>TCPVP: tcp_fsm_netw_recvseg_in_active()<br/>state: SYNSENT→ESTABLISHED

    TCPVP->>SockVP: socket_netw_connect_rsp()<br/>via ntoh_flow

    SockVP-->>AppVP: tcp_pkt_tx_netw_connect_rsp_wuh()<br/>on ntoc_f, ISS/IRS

    Note over AppVP: state=TCP_PGEN_STATE_CONNECTED<br/>Call start_pgen → pgen_send_data
```

### Key WU Handlers — Client Establishment

| # | WU Handler | File | VP | Direction | Purpose |
|---|-----------|------|----|-----------|---------| 
| 1 | `tcp_pkt_gen` | tcp_pkt_gen.c | App | CHANNEL (TOP_LEVEL) | Entry: parse config, launch connections |
| 2 | `tcp_pkt_tx_connect_flow_allocated` | tcp_pkt_gen.c | App | CHANNEL | Init cton_f/ntoc_f flows |
| 3 | `tcp_pkt_tx_connect_socket_allocated` | tcp_pkt_gen.c | App | CHANNEL | Handle socket creation response |
| 4 | `tcp_host_connect_req` | tcp.c | TCP FSM | WU (App→TCP) | Process connect request |
| 5 | `tcp_fsm_host_connect_init` | tcp_fsm.c | TCP FSM | Internal | Init TCP flow state |
| 6 | `tcp_host_connect_req_le_added` | tcp.c | TCP FSM | WU (LE callback) | LE entry added |
| 7 | `tcp_fsm_host_connect` | tcp_fsm.c | TCP FSM | Internal | CLOSED→SYNSENT, send SYN |
| 8 | `tcp_netw_recvseg_active` | tcp.c | TCP RCV | WU (Net→TCP) | Process SYN-ACK |
| 9 | `tcp_fsm_netw_recvseg_in_active` | tcp_fsm.c | TCP FSM | Internal | SYNSENT→ESTABLISHED |
| 10 | `tcp_pkt_tx_netw_connect_rsp_wuh` | tcp_pkt_gen.c | App | WU (TCP→App) | Connect response on ntoc_f |

---

## Connection Establishment: Server (Passive Open)

### Sequence of WU Handlers

```mermaid
sequenceDiagram
    participant dpcsh as dpcsh<br/>(JSON cmd)
    participant AppVP as App VP<br/>(tcp_pkt_rcv)
    participant SockVP as Socket Layer<br/>(socket.c)
    participant TCPVP as TCP FSM VP<br/>(tcp.c / tcp_fsm.c)
    participant LE as Lookup Engine
    participant Net as Network

    Note over dpcsh: User issues tcp_pkt_rcv command

    dpcsh->>AppVP: CHANNEL_HANDLER(tcp_pkt_rcv)<br/>TOP_LEVEL, parse JSON config

    AppVP->>AppVP: tcp_pktrcv_listen_create()<br/>Allocate tcp_pkt_rx_flow

    AppVP->>AppVP: tcp_pkt_rx_listen_flow_allocated()<br/>flow_init(ston_f, ntos_f)<br/>TOP_LEVEL

    Note over AppVP: flow_set_dispatch_func(ntos_f)<br/>tcp_host_flow_bind(socketid, ston_f, ntos_f)

    AppVP->>SockVP: socket_host_listen_start_req_push()<br/>(backlog=1024, port, saddr)

    SockVP->>TCPVP: tcp_host_listen_req()<br/>via flow_bind(hton_f)

    TCPVP->>TCPVP: tcp_fsm_host_listen()<br/>state: INVALID→LISTEN

    TCPVP->>LE: LE add listener entry<br/>(port → tcp_netw_recvseg_listen)

    LE-->>TCPVP: tcp_host_listen_req_le_added()

    TCPVP->>SockVP: socket_netw_listen_rsp()

    SockVP-->>AppVP: tcp_pkt_rx_netw_listen_rsp_wuh()<br/>on ntos_f

    Note over AppVP: Store in listen_socket_tbl[]<br/>state=TCP_PRCV_STATE_START_LISTEN

    Note over Net: Client sends SYN

    Net-->>TCPVP: tcp_netw_recvseg_listen()<br/>(LE hit on listen port)

    TCPVP->>TCPVP: Create new TCP flow<br/>state: SYNRCVD

    TCPVP->>Net: tcp_xmt_syn_ack()<br/>Send SYN-ACK

    TCPVP->>SockVP: socket_netw_acceptmsg()<br/>(accept_req with new socketid)

    SockVP-->>AppVP: tcp_pkt_rx_netw_accept_req_wuh()<br/>on ntos_f

    AppVP->>AppVP: tcp_pkt_rx_new_allocate()<br/>Allocate new tcp_pkt_rx_flow

    AppVP->>SockVP: socket_host_accept_rsp_push()<br/>Accept the connection

    SockVP->>TCPVP: tcp_host_accept_rsp()<br/>VP rewrite if needed

    Note over Net: Client sends ACK (3rd leg)

    Net-->>TCPVP: tcp_netw_recvseg_active()<br/>ACK completes handshake

    TCPVP->>TCPVP: tcp_fsm_host_accept()<br/>state: SYNRCVD→ESTABLISHED

    TCPVP->>SockVP: socket_netw_acceptmsg()

    SockVP-->>AppVP: tcp_pkt_rx_netw_acceptmsg_wuh()<br/>on ntos_f, ISS/IRS

    Note over AppVP: Connection established<br/>Ready to receive data
```

### Key WU Handlers — Server Establishment

| # | WU Handler | File | VP | Direction | Purpose |
|---|-----------|------|----|-----------|---------| 
| 1 | `tcp_pkt_rcv` | tcp_pkt_rcv.c | App | CHANNEL (TOP_LEVEL) | Entry: parse config |
| 2 | `tcp_pkt_rx_listen_flow_allocated` | tcp_pkt_rcv.c | App | CHANNEL (TOP_LEVEL) | Init ston_f/ntos_f |
| 3 | `tcp_host_listen_req` | tcp.c | TCP FSM | WU (App→TCP) | Process listen request |
| 4 | `tcp_fsm_host_listen` | tcp_fsm.c | TCP FSM | Internal | Set state=LISTEN |
| 5 | `tcp_host_listen_req_le_added` | tcp.c | TCP FSM | WU (LE callback) | LE listener registered |
| 6 | `tcp_pkt_rx_netw_listen_rsp_wuh` | tcp_pkt_rcv.c | App | WU (TCP→App) | Listen ack on ntos_f |
| 7 | `tcp_netw_recvseg_listen` | tcp.c | TCP RCV | WU (Net→TCP) | SYN received, create new flow |
| 8 | `tcp_pkt_rx_netw_accept_req_wuh` | tcp_pkt_rcv.c | App | WU (TCP→App) | Accept request on ntos_f |
| 9 | `tcp_host_accept_rsp` | tcp.c | TCP FSM | WU (App→TCP) | App accepts connection |
| 10 | `tcp_pkt_rx_netw_acceptmsg_wuh` | tcp_pkt_rcv.c | App | WU (TCP→App) | Connection established |

---

## Connection Teardown: Client-Initiated Close

```mermaid
sequenceDiagram
    participant ClientApp as Client App VP<br/>(tcp_pkt_gen)
    participant SockC as Socket Layer
    participant TCPVP_C as TCP FSM VP<br/>(Client Side)
    participant Net as Network
    participant TCPVP_S as TCP FSM VP<br/>(Server Side)
    participant SockS as Socket Layer
    participant ServerApp as Server App VP<br/>(tcp_pkt_rcv)

    Note over ClientApp: Send complete or terminate flag set<br/>state=TCP_PGEN_STATE_SEND_HALT

    ClientApp->>SockC: socket_host_disconnect_req_push()

    SockC->>TCPVP_C: tcp_host_disconnect_req()

    TCPVP_C->>TCPVP_C: tcp_fsm_host_disconnect()<br/>ESTABLISHED→FINWAIT1

    TCPVP_C->>Net: tcp_xmt_fin()<br/>Send FIN+ACK

    Note over TCPVP_C: Start FIN timer

    Net-->>TCPVP_S: FIN received<br/>tcp_netw_recvseg_active()

    TCPVP_S->>TCPVP_S: tcp_fsm_netw_recvseg_in_active()<br/>ESTABLISHED→CLOSEWAIT

    TCPVP_S->>SockS: socket_netw_disconnectmsg()

    SockS-->>ServerApp: tcp_pkt_rx_netw_disconnectmsg_wuh()<br/>on ntos_f (peer disconnect)

    ServerApp->>SockS: socket_host_disconnect_req_push()<br/>Acknowledge close

    SockS->>TCPVP_S: tcp_host_disconnect_req()

    TCPVP_S->>TCPVP_S: tcp_fsm_host_disconnect()<br/>CLOSEWAIT→LASTACK

    TCPVP_S->>Net: tcp_xmt_fin()<br/>Send FIN+ACK

    Net-->>TCPVP_C: FIN+ACK received<br/>tcp_netw_recvseg_active()

    TCPVP_C->>TCPVP_C: FINWAIT1→TIMEWAIT<br/>(or FINWAIT2→TIMEWAIT)

    TCPVP_C->>SockC: socket_netw_disconnect_rsp()

    SockC-->>ClientApp: tcp_pkt_tx_netw_disconnect_rsp_wuh()<br/>on ntoc_f

    Note over ClientApp: tcp_pkt_socket_teardown()<br/>Destroy flows, free resources

    Net-->>TCPVP_S: ACK of FIN

    TCPVP_S->>TCPVP_S: LASTACK→CLOSED

    TCPVP_S->>SockS: socket_netw_disconnect_rsp()

    SockS-->>ServerApp: tcp_pkt_rx_netw_disconnect_rsp_wuh()<br/>on ntos_f

    Note over ServerApp: tcp_rcv_socket_teardown()<br/>Destroy flows, free resources

    Note over TCPVP_C: TIMEWAIT→CLOSED (2MSL timeout)<br/>tcp_netw_le_removed() → LE cleanup
```

### Key WU Handlers — Teardown

| # | WU Handler | File | VP | Direction | Purpose |
|---|-----------|------|----|-----------|---------| 
| 1 | `tcp_host_disconnect_req` | tcp.c | TCP FSM | WU (App→TCP) | Process disconnect |
| 2 | `tcp_fsm_host_disconnect` | tcp_fsm.c | TCP FSM | Internal | ESTABLISHED→FINWAIT1 or CLOSEWAIT→LASTACK |
| 3 | `tcp_fsm_netw_fini_disconnectmsg_send` | tcp_fsm.c | TCP FSM | Internal | Send FIN packet |
| 4 | `tcp_pkt_tx_netw_disconnect_rsp_wuh` | tcp_pkt_gen.c | App (Client) | WU (TCP→App) | Disconnect response |
| 5 | `tcp_pkt_tx_netw_disconnectmsg_wuh` | tcp_pkt_gen.c | App (Client) | WU (TCP→App) | Peer disconnect notify |
| 6 | `tcp_pkt_rx_netw_disconnectmsg_wuh` | tcp_pkt_rcv.c | App (Server) | CHANNEL (TCP→App) | Peer disconnect on ntos_f |
| 7 | `tcp_pkt_rx_netw_disconnect_rsp_wuh` | tcp_pkt_rcv.c | App (Server) | WU (TCP→App) | Disconnect ack |
| 8 | `tcp_netw_le_removed` | tcp.c | TCP FSM | WU (LE callback) | LE entry removed |
| 9 | `tcp_pkt_socket_teardown` | tcp_pkt_gen.c | App (Client) | CHANNEL | Socket cleanup |
| 10 | `tcp_rcv_socket_teardown` | tcp_pkt_rcv.c | App (Server) | CHANNEL | Socket cleanup |

---

## Abort Path (RST)

```mermaid
sequenceDiagram
    participant App as App VP
    participant Sock as Socket Layer
    participant TCP as TCP FSM VP
    participant Net as Network

    alt App-initiated abort
        App->>Sock: socket_host_abort_req_push()
        Sock->>TCP: tcp_fsm_host_abort_req_wuh()
        TCP->>TCP: Any state → CLOSED
        TCP->>Net: Send RST packet
        TCP->>Sock: socket_netw_abort_rsp()
        Sock-->>App: tcp_pkt_tx_netw_abort_rsp_wuh()<br/>on ntoc_f/ntos_f
    else Peer-initiated abort (RST received)
        Net-->>TCP: RST received<br/>tcp_netw_recvseg_active()
        TCP->>TCP: Any state → CLOSED
        TCP->>Sock: socket_netw_abortmsg()
        Sock-->>App: tcp_pkt_tx_abortmsg_wuh()<br/>on ntoc_f/ntos_f
    end

    Note over App: Cleanup flows and resources
```

---

## VP Queue Architecture

### Workerpool Registration

```
┌─────────────────────────────────────────────────────────────────┐
│                      WORKERPOOL HIERARCHY                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  App Layer Workerpools (tcp_pkt_gen.c / tcp_pkt_rcv.c):         │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  tcp_test_pool          (100 VPs, default)                │  │
│  │  tcp_prcv_random_pool   (100 VPs, random assignment)      │  │
│  │  Size: TCP_PKT_TX_RX_WP_RSRC_POOL = 100 (silicon)        │  │
│  │                                       48 (POSIX/S1)       │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
│  TCP Stack Workerpools (tcp.c):                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  wp_tcp_storage[cluster]     (per-cluster, round-robin)   │  │
│  │  wp_tcp_tls_storage[cluster] (per-cluster, TLS mode)      │  │
│  │  Flags: WP_SIZE_ANY, WP_FLAG_SHRINK                       │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### VP Assignment Modes for App Flows

The client app (`tcp_pkt_gen`) supports three workerpool modes:

| Mode | Assignment Strategy | Code |
|------|-------------------|------|
| **TG_STATIC_WP** | Fixed VP based on socket index: `FADDR(GID_PC(pc), LID_VP(core, vp), 0)` | Deterministic, per-chip formula |
| **TG_RANDOM_WP** | Random VP: `WP_ANY(tcp_prcv_random_pool)` | Load-balanced |
| **TG_DEFAULT_TEST_WP** | Round-robin: `WP_GET(tcp_test_pool, idx % count)` | Default mode |

### TCP VP Assignment and Queue Priority

```mermaid
graph LR
    subgraph "App VP (tcp_test_pool)"
        A_CTON["cton_f / ston_f<br/>(outbound flow)"]
        A_NTOC["ntoc_f / ntos_f<br/>(inbound flow)"]
    end

    subgraph "Socket VP"
        S_HTON["socket_hton_flow"]
        S_NTOH["socket_ntoh_flow"]
    end

    subgraph "TCP Storage VP (wp_tcp_storage)"
        T_HTON["tcp_hton_flow<br/>HIGH PRIORITY queue<br/>faddr_vp_to_high_priority(tcp_vp)"]
        T_NTOH["tcp_ntoh_flow<br/>Normal priority<br/>tcp_vp"]
        T_XMT["tcp_xmt_dest<br/>xmt_flow, xmt_pb_flow<br/>Same VP as FSM"]
    end

    subgraph "Network"
        LE_Q["LE / ETP WU Queue<br/>fae_etp_wu_queue"]
    end

    A_CTON --> S_HTON
    S_HTON -- "flow_bind()" --> T_HTON
    T_HTON --> T_XMT

    T_NTOH --> S_NTOH
    S_NTOH -- "flow_callee()" --> A_NTOC

    LE_Q -- "LE hit" --> T_NTOH
    T_XMT -- "TX" --> LE_Q
```

Key points:
- **HTON flow** (App→TCP) uses **high-priority VP queue** via `faddr_vp_to_high_priority(tcp_vp)` — ensures control messages (connect, disconnect) preempt data path
- **NTOH flow** (Net→TCP) uses normal priority on the same TCP storage VP
- **XMT VP** is the same as the FSM VP (`*xmt_vp = *tcp_vp`) for zero-copy efficiency
- VP assignment is **round-robin per cluster**: `atomic_fetch_add(&tcp_storage_rr[cluster], 1)`

---

## App-to-TCP WU Handler Dispatch

Both apps register a **dispatch function** on their inbound flow (`ntoc_f` / `ntos_f`) that maps
TCP socket opcodes to local WU handlers:

### Client Dispatch Table (`tcp_pkt_gen_module_init`)

```c
socket_req[FUN_SOCKET_OP_CONNECT]       = WUID(tcp_pkt_tx_netw_connect_rsp_wuh);
socket_req[FUN_SOCKET_OP_DISCONNECT]    = WUID(tcp_pkt_tx_netw_disconnect_rsp_wuh);
socket_req[FUN_SOCKET_OP_DISCONNECTMSG] = WUID(tcp_pkt_tx_netw_disconnectmsg_wuh);
socket_req[FUN_SOCKET_OP_ABORT]         = WUID(tcp_pkt_tx_netw_abort_rsp_wuh);
socket_req[FUN_SOCKET_OP_ABORTMSG]      = WUID(tcp_pkt_tx_abortmsg_wuh);
socket_req[FUN_SOCKET_OP_RECVMSG]       = WUID(tcp_pkt_tx_recvmsg_no_frame_wuh);
socket_req[FUN_SOCKET_OP_SENDUPDMSG]    = WUID(tcp_pkt_tx_sendupdmsg_no_frame_wuh);
socket_req[FUN_SOCKET_OP_RRECVMSG]      = WUID(tcp_pkt_tx_rrecvmsg_wuh);
socket_req[FUN_SOCKET_OP_TLS_UPGRADE]   = WUID(tcp_pkt_tx_netw_tls_upgrade_rsp_wuh);
```

### Server Dispatch Table (`tcp_pkt_rcv_module_init`)

```c
socket_req[FUN_SOCKET_OP_LISTEN]        = WUID(tcp_pkt_rx_netw_listen_rsp_wuh);
socket_req[FUN_SOCKET_OP_ACCEPT]        = WUID(tcp_pkt_rx_netw_accept_req_wuh);
socket_req[FUN_SOCKET_OP_ACCEPTMSG]     = WUID(tcp_pkt_rx_netw_acceptmsg_wuh);
socket_req[FUN_SOCKET_OP_CONNECT]       = WUID(tcp_pkt_rx_netw_connect_rsp_wuh);
socket_req[FUN_SOCKET_OP_DISCONNECT]    = WUID(tcp_pkt_rx_netw_disconnect_rsp_wuh);
socket_req[FUN_SOCKET_OP_DISCONNECTMSG] = WUID(tcp_pkt_rx_netw_disconnectmsg_wuh);
socket_req[FUN_SOCKET_OP_ABORT]         = WUID(tcp_pkt_rx_netw_abort_rsp_wuh);
socket_req[FUN_SOCKET_OP_ABORTMSG]      = WUID(tcp_pkt_rx_netw_abortmsg_wuh);
socket_req[FUN_SOCKET_OP_RECVMSG]       = WUID(tcp_pkt_rx_recvmsg_no_frame_wuh);
socket_req[FUN_SOCKET_OP_SENDUPDMSG]    = WUID(tcp_pkt_rx_sendupdmsg_no_frame_wuh);
socket_req[FUN_SOCKET_OP_RRECVMSG]      = WUID(tcp_pkt_rx_rrecvmsg_wuh);
```

---

## Complete Client–Server Interaction

```mermaid
sequenceDiagram
    participant CApp as Client App VP<br/>(tcp_pkt_gen)
    participant CSock as Client Socket
    participant CTCP as Client TCP FSM VP
    participant Net as Network
    participant STCP as Server TCP FSM VP
    participant SSock as Server Socket
    participant SApp as Server App VP<br/>(tcp_pkt_rcv)

    Note over SApp,STCP: === LISTEN PHASE ===
    SApp->>SSock: socket_host_listen_start_req_push()
    SSock->>STCP: tcp_host_listen_req()
    STCP->>STCP: tcp_fsm_host_listen() → LISTEN
    STCP-->>SApp: tcp_pkt_rx_netw_listen_rsp_wuh()

    Note over CApp,CTCP: === CONNECT PHASE (3-Way Handshake) ===
    CApp->>CSock: socket_host_connect_req_push()
    CSock->>CTCP: tcp_host_connect_req()
    CTCP->>CTCP: CLOSED→SYNSENT
    CTCP->>Net: SYN →
    Net->>STCP: → SYN (tcp_netw_recvseg_listen)
    STCP->>STCP: Create flow, SYNRCVD
    STCP->>Net: ← SYN-ACK
    STCP-->>SApp: tcp_pkt_rx_netw_accept_req_wuh()
    SApp->>SSock: socket_host_accept_rsp_push()
    Net->>CTCP: SYN-ACK → (tcp_netw_recvseg_active)
    CTCP->>CTCP: SYNSENT→ESTABLISHED
    CTCP->>Net: ACK →
    CTCP-->>CApp: tcp_pkt_tx_netw_connect_rsp_wuh()
    Net->>STCP: → ACK
    STCP->>STCP: SYNRCVD→ESTABLISHED
    STCP-->>SApp: tcp_pkt_rx_netw_acceptmsg_wuh()

    Note over CApp,SApp: === DATA TRANSFER ===
    loop Send data
        CApp->>CSock: flow_callee_push(cton_f, sendmsg)
        CSock->>CTCP: socket_host_sendmsg()
        CTCP->>Net: TCP data segment →
        Net->>STCP: → TCP data (tcp_netw_recvseg_active)
        STCP-->>SApp: tcp_pkt_rx_recvmsg_no_frame_wuh()
        STCP-->>CApp: tcp_pkt_tx_sendupdmsg_no_frame_wuh()<br/>(send credits)
    end

    Note over CApp,SApp: === TEARDOWN (Client-Initiated) ===
    CApp->>CSock: socket_host_disconnect_req_push()
    CSock->>CTCP: tcp_host_disconnect_req()
    CTCP->>CTCP: ESTABLISHED→FINWAIT1
    CTCP->>Net: FIN →
    Net->>STCP: → FIN
    STCP->>STCP: ESTABLISHED→CLOSEWAIT
    STCP-->>SApp: tcp_pkt_rx_netw_disconnectmsg_wuh()
    SApp->>SSock: socket_host_disconnect_req_push()
    SSock->>STCP: tcp_host_disconnect_req()
    STCP->>STCP: CLOSEWAIT→LASTACK
    STCP->>Net: ← FIN
    Net->>CTCP: FIN → (FINWAIT1→TIMEWAIT)
    CTCP-->>CApp: tcp_pkt_tx_netw_disconnect_rsp_wuh()
    Net->>STCP: → ACK (LASTACK→CLOSED)
    STCP-->>SApp: tcp_pkt_rx_netw_disconnect_rsp_wuh()
```

---

## TCP Flow Internal Structure

The `struct tcp_flow` (`tcp_internal.h`) contains the complete per-connection state:

```
┌──────────────────────────────────────────────────────────┐
│                    struct tcp_flow                        │
├──────────────────────────────────────────────────────────┤
│  tcp_hton_flow        (FLOW_ALIGN_SIZE)  App→TCP         │
│  tcp_ntoh_flow        (FLOW_ALIGN_SIZE)  TCP→App         │
│  tcp_flow_timer       Timer management                   │
├──────────────────────────────────────────────────────────┤
│  tcp_state            FSM state (enum tcp_state)         │
│  tcp_timer_type       RXMT/PERSIST/KEEPALIVE/etc.        │
│  tcp_sport, tcp_dport Source/dest ports                   │
│  tcp_saddr, tcp_daddr Source/dest addresses               │
├──────────────────────────────────────────────────────────┤
│  tcp_iss              Initial send sequence               │
│  tcp_irs              Initial receive sequence            │
│  tcp_snd_una          Send unacknowledged                 │
│  tcp_snd_lst          Send last/next                      │
│  tcp_snd_max          Maximum sent                        │
│  tcp_rcv_nxt          Receive next expected               │
│  tcp_rcv_wnd          Receive window                      │
├──────────────────────────────────────────────────────────┤
│  tcp_xmt_dest         XMT VP fabric address               │
│  wuqueue              ETP WU queue info                   │
├──────────────────────────────────────────────────────────┤
│  tcp_xmt              (ALIGNED 256) Transmit state        │
│  tcp_rcv              (ALIGNED 256) Receive state         │
└──────────────────────────────────────────────────────────┘
```

### Timer Types

| Timer | Purpose | Triggered By |
|-------|---------|-------------|
| `TCP_FSM_TIMER_TYPE_RXMT` | Retransmission timeout | SYN/FIN/data not ACK'd |
| `TCP_FSM_TIMER_TYPE_PERSIST` | Zero window probe | Receiver window = 0 |
| `TCP_FSM_TIMER_TYPE_KEEPALIVE` | Keep-alive probe | Idle connection |
| `TCP_FSM_TIMER_TYPE_FINWAIT2` | FIN-WAIT-2 timeout | In FINWAIT2 state |
| `TCP_FSM_TIMER_TYPE_TIMEWAIT` | TIME-WAIT (2MSL) | In TIMEWAIT state |

---

## Summary

### WU Handler Count by Category

| Category | Client (tcp_pkt_gen) | Server (tcp_pkt_rcv) | TCP Stack |
|----------|---------------------|---------------------|-----------|
| Establishment | 3 WU + 5 CHANNEL | 4 WU + 4 CHANNEL | 6 WU |
| Teardown | 3 WU + 1 CHANNEL | 2 WU + 2 CHANNEL | 4 WU |
| Data Path | 4 WU | 3 WU | 2 WU |
| Timers | 2 WU | 1 WU | 5 types |

### Flow Direction Summary

```
Client App                    TCP Stack                    Server App
    │                            │                            │
    ├──cton_f──→ socket_hton ──→ tcp_hton (HP) ──→ XMT ──→ NET
    │                            │                            │
    ←──ntoc_f──← socket_ntoh ←── tcp_ntoh ←────── RCV ←── NET
    │                            │                            │
    │                      NET ──→ tcp_ntoh (HP) ──→ tcp_hton│
    │                            │                            │
    │                      NET ←── XMT ←── tcp_hton ←── ston_f
    │                            │                    ntos_f──┤
```

### Key Design Patterns

1. **Flow binding** bridges VP boundaries: `flow_bind(app_flow, tcp_flow)` connects app and TCP VPs
2. **Dispatch functions** (`tcp_pkt_tx_ntoc_dispatch_func`) map opcodes to WU handlers on the inbound flow
3. **High-priority queue** for control path: HTON flow dest uses `faddr_vp_to_high_priority(tcp_vp)`
4. **Round-robin VP assignment** across clusters for TCP storage VPs
5. **Channel handlers** (TOP_LEVEL) for threaded init sequences; WU handlers for async message processing
