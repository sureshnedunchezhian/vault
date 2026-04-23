# TCP Source Port Selection: Linux vs Windows — A Deep Technical Analysis

## Executive Summary

TCP source port selection is a critical aspect of connection establishment that affects both **security** (predictability enables spoofing attacks) and **performance** (port exhaustion under high connection churn). Linux and Windows implement fundamentally different algorithms: Linux uses a **hash-based randomized selection with perturbation tables and even/odd parity scanning**, while Windows uses a **CSPRNG-seeded random pick with linear fallback**. Both follow guidance from **RFC 6056** (port randomization) and **RFC 6335** (IANA port ranges), though with significant implementation divergence. This report covers the algorithms, tunables, socket options, and RFC compliance for both platforms.

---

## 1. Relevant RFCs

### RFC 793 — Transmission Control Protocol (1981)
The original TCP specification defines 16-bit port numbers (0–65535) and reserves ports 0–1023 as "Well Known Ports." It provides **no guidance** on ephemeral port selection algorithms — leaving this entirely to the OS implementation. Early implementations used simple sequential allocation[^1].

### RFC 6335 — IANA Port Assignment Procedures (2011)
Formalizes the port number taxonomy[^2]:

| Range | Name | Assignment |
|-------|------|-----------|
| 0–1023 | Well-Known (System) Ports | IANA-assigned, privileged |
| 1024–49151 | Registered (User) Ports | IANA-registered services |
| 49152–65535 | Dynamic/Private/Ephemeral | Unassigned; for client-side use |

This RFC defines 49152–65535 as the **IANA-recommended ephemeral port range**, though operating systems may deviate.

### RFC 6056 — Recommendations for Transport-Protocol Port Randomization (2011)
The primary RFC governing source port selection security. It identifies five algorithms[^3]:

| Algorithm | Description | Predictability | Collision Risk |
|-----------|------------|---------------|---------------|
| **Algorithm 1: Simple Randomization** | Random port from ephemeral range per connection | Low | Moderate |
| **Algorithm 2: Random-Increment** | Previous port + random increment | Medium-Low | Low |
| **Algorithm 3: Hash-Based (Double Hash)** | Cryptographic hash of connection params + secret | Very Low | Low |
| **Algorithm 4: Port Range Subdivision** | Per-destination subranges, random within | Low | Very Low |
| **Algorithm 5: Bitmap/Table** | Track used ports to avoid reuse | Low | Very Low |

**Key recommendation**: Use hash-based or random-increment algorithms to balance security and performance. Sequential allocation is explicitly deprecated for security reasons.

### RFC 4953 — Defending TCP Against Spoofing Attacks (2007)
Provides threat analysis motivating port randomization — predictable source ports enable blind TCP injection and RST attacks[^4].

### RFC 1323 — TCP Extensions for High Performance (1992)
Covers window scaling and timestamps but does **not** change port selection. Mentioned here because timestamps provide an alternative defense against injection attacks[^5].

---

## 2. Linux TCP Source Port Selection

### 2.1 Architecture Overview

```
┌──────────────────────────────────────────────────────────────┐
│                    User Space: connect()                      │
└──────────────────────────┬───────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│               tcp_v4_connect() / tcp_v6_connect()            │
│                     net/ipv4/tcp_ipv4.c                      │
└──────────────────────────┬───────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│                  inet_hash_connect()                          │
│              net/ipv4/inet_hashtables.c                       │
│                                                              │
│  1. Compute port_offset = inet_sk_port_offset(sk)            │
│  2. Compute hash_port0 = inet_ehashfn(net, saddr, 0,        │
│                                        daddr, dport)         │
│  3. Call __inet_hash_connect() with offset + hash            │
└──────────────────────────┬───────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│               __inet_hash_connect()                           │
│                                                              │
│  • Get ephemeral range [low, high] from sysctl/SO_PORT_RANGE│
│  • Compute random offset from table_perturb[] + port_offset  │
│  • Scan ports with random step (GCD-coprime traversal)       │
│  • Even/odd parity: first pass tries even, second tries odd  │
│  • Check bind hash (bhash) for conflicts                     │
│  • Check established hash (ehash) for 5-tuple conflicts      │
│  • Reuse TIME_WAIT sockets if safe (tcp_tw_reuse)            │
└──────────────────────────────────────────────────────────────┘
```

### 2.2 Core Algorithm: `__inet_hash_connect()`

The implementation lives in `net/ipv4/inet_hashtables.c`[^6]. Here is the algorithm in detail:

#### Step 1: Get Port Range
```c
local_ports = inet_sk_get_local_port_range(sk, &low, &high);
step = local_ports ? 1 : 2;  // step=2 enables even/odd parity scanning
```
- If the socket has a per-socket port range (via `IP_LOCAL_PORT_RANGE`), `step=1` (no parity).
- Otherwise, `step=2` to enable even/odd parity optimization[^7].

#### Step 2: Compute Randomized Offset
```c
get_random_sleepable_once(table_perturb, INET_TABLE_PERTURB_SIZE * sizeof(*table_perturb));
index = port_offset & (INET_TABLE_PERTURB_SIZE - 1);
offset = READ_ONCE(table_perturb[index]) + (port_offset >> 32);
offset %= remaining;
```
- `table_perturb[]` is a per-system random perturbation table (size controlled by `CONFIG_INET_TABLE_PERTURB_ORDER`), initialized once with random data[^8].
- `port_offset` is computed from `secure_tcp_port_hash()` which uses **SipHash** over `(saddr, daddr, dport, secret)` — this is the **RFC 6056 Algorithm 3 (hash-based)** approach[^9].
- The perturbation table adds extra randomness beyond the per-connection hash.

#### Step 3: Random Step Width (Anti-Predictability)
```c
max_rand_step = READ_ONCE(net->ipv4.sysctl_ip_local_port_step_width);
if (max_rand_step && remaining > 1) {
    upper_bound = min(range, max_rand_step);
    scan_step = get_random_u32_inclusive(1, upper_bound);
    while (gcd(scan_step, range) != 1) {
        scan_step++;
        ...
    }
    scan_step *= step;
}
```
- Instead of scanning ports sequentially (offset, offset+1, offset+2...), the kernel uses a **random coprime step** to traverse the port space in a non-sequential pattern[^10].
- The GCD check ensures the step covers all ports (full period traversal).

#### Step 4: Even/Odd Parity Scanning
```c
/* In first pass we try ports of @low parity.
 * inet_csk_get_port() does the opposite choice.
 */
if (!local_ports)
    offset &= ~1U;  // Force even offset in first pass
...
other_parity_scan:
    // Second pass tries opposite parity
```
- Connect-side uses **even** ports first; bind-side (`inet_csk_get_port`) uses **odd** first.
- This reduces conflicts between outbound connections and listening sockets[^11].

#### Step 5: Port Validation
For each candidate port, the kernel checks:
1. Is the port in `ip_local_reserved_ports`? → Skip
2. Is there a bind bucket conflict in `bhash`? → Check further
3. Is the 5-tuple `(saddr, sport, daddr, dport, proto)` already established in `ehash`? → Skip
4. Can we reuse a TIME_WAIT socket? → Reuse if `tcp_tw_reuse` is enabled

#### Step 6: Perturbation Table Update
```c
WRITE_ONCE(table_perturb[index], READ_ONCE(table_perturb[index]) + i + step);
```
After successful port selection, the perturbation table entry is updated to prevent the next connection to the same destination from starting at the same offset[^12].

### 2.3 Linux Sysctl Tunables

| Sysctl | Default | Description |
|--------|---------|-------------|
| `net.ipv4.ip_local_port_range` | `32768 60999` | Ephemeral port range [low, high] |
| `net.ipv4.ip_local_reserved_ports` | (empty) | Comma-separated ports excluded from ephemeral use |
| `net.ipv4.ip_unprivileged_port_start` | `1024` | Lowest port non-root processes can bind |
| `net.ipv4.tcp_tw_reuse` | `0` | Allow reusing TIME_WAIT sockets for outbound: 0=off, 1=global, 2=loopback-only |
| `net.ipv4.ip_local_port_step_width` | (varies) | Max random step width for port scanning |

**Viewing and setting:**
```bash
# View current range
cat /proc/sys/net/ipv4/ip_local_port_range
# Set wider range
sysctl -w net.ipv4.ip_local_port_range="1024 65535"
# Reserve specific ports
sysctl -w net.ipv4.ip_local_reserved_ports="8080,8443,3000-3100"
# Enable TIME_WAIT reuse
sysctl -w net.ipv4.tcp_tw_reuse=1
```

### 2.4 Linux Socket Options Affecting Port Selection

| Socket Option | Level | Description |
|--------------|-------|-------------|
| `IP_LOCAL_PORT_RANGE` | `IPPROTO_IP` | Per-socket ephemeral port range override (Linux 6.3+) |
| `IP_BIND_ADDRESS_NO_PORT` | `IPPROTO_IP` | Defer port allocation until `connect()` — allows kernel to consider 5-tuple |
| `SO_REUSEADDR` | `SOL_SOCKET` | Allow binding to address in TIME_WAIT; no effect on auto port selection |
| `SO_REUSEPORT` | `SOL_SOCKET` | Allow multiple sockets to share same (addr, port); no effect on auto port selection |
| `SO_BINDTODEVICE` | `SOL_SOCKET` | Bind to specific interface; ports can be reused across different interfaces |

#### `IP_BIND_ADDRESS_NO_PORT` (Linux 4.2+)
This is particularly important for high-connection-rate clients. Without it, `bind()` to a specific source IP with port 0 allocates a port immediately based only on the source side. With this option, port allocation is deferred to `connect()`, allowing the kernel to use the full 5-tuple for conflict detection, dramatically increasing available connections[^13]:

```c
// Without IP_BIND_ADDRESS_NO_PORT: port chosen at bind() — only 28K ports available
bind(sock, src_addr, 0);  // port allocated here
connect(sock, dst_addr);

// With IP_BIND_ADDRESS_NO_PORT: port chosen at connect() — 28K ports × N destinations
setsockopt(sock, IPPROTO_IP, IP_BIND_ADDRESS_NO_PORT, &one, sizeof(one));
bind(sock, src_addr, 0);  // no port allocated
connect(sock, dst_addr);  // port allocated here, considering full 5-tuple
```

### 2.5 Algorithm Classification (per RFC 6056)

Linux primarily implements **Algorithm 3 (Hash-Based)** from RFC 6056, enhanced with:
- Perturbation tables for additional randomization
- Random coprime step traversal (going beyond RFC 6056's suggestions)
- Even/odd parity separation between connect and bind paths
- Per-socket port range override capability

---

## 3. Windows TCP Source Port Selection

### 3.1 Architecture Overview

Windows TCP/IP is implemented in the kernel-mode driver `tcpip.sys`. Source port selection occurs during `connect()` processing.

```
┌──────────────────────────────────────────────────────────────┐
│                 User Space: connect() / WSAConnect()          │
└──────────────────────────┬───────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│                      AFD.sys (Ancillary Function Driver)      │
└──────────────────────────┬───────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│                      tcpip.sys                                │
│                                                              │
│  TcpAllocatePort():                                          │
│  1. Read ephemeral range from registry/netsh config          │
│  2. Pick random port from CSPRNG (seeded at boot)            │
│  3. Check if port is in use (active connection table)        │
│  4. If in use → linear scan to next available port           │
│  5. Assign port to connection                                │
└──────────────────────────┬───────────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────────┐
│              WFP (Windows Filtering Platform)                 │
│              (Post-selection callout, cannot alter port)      │
└──────────────────────────────────────────────────────────────┘
```

### 3.2 Ephemeral Port Range History

| Windows Version | Default Ephemeral Range | Selection Algorithm |
|----------------|------------------------|-------------------|
| XP / Server 2003 | 1025–5000 | Sequential (next available) |
| Vista / Server 2008 | 49152–65535 | Randomized (CSPRNG) |
| 7 / Server 2008 R2 | 49152–65535 | Randomized |
| 8 / Server 2012+ | 49152–65535 | Randomized + SO_REUSE_UNICASTPORT |
| 10 / 11 / Server 2016+ | 49152–65535 | Randomized |

The shift to 49152–65535 in Vista aligned Windows with the IANA recommendation in RFC 6335[^14].

### 3.3 Selection Algorithm Details

#### Pre-Vista (Sequential)
```
port = last_assigned_port + 1
if port > max_port:
    port = min_port
while port_in_use(port):
    port++
```
This was **highly predictable** and vulnerable to the attacks described in RFC 4953.

#### Vista and Later (Randomized)
```
port = csprng_random() % (max_port - min_port + 1) + min_port
while port_in_use(port):
    port = (port + 1)
    if port > max_port:
        port = min_port
```
- Uses a **cryptographically secure PRNG** seeded at boot time[^15].
- Falls back to **linear probing** if the random port is occupied.
- This corresponds to a simplified version of **RFC 6056 Algorithm 1 (Simple Randomization)** with linear fallback.

### 3.4 Windows Configuration Options

#### `netsh` Commands
```powershell
# View current ephemeral port range
netsh int ipv4 show dynamicport tcp
netsh int ipv6 show dynamicport tcp

# Set custom range (start port + count)
netsh int ipv4 set dynamicport tcp start=10000 num=55535
netsh int ipv6 set dynamicport tcp start=10000 num=55535

# Exclude specific ports from ephemeral use
netsh int ipv4 add excludedportrange protocol=tcp startport=8080 numberofports=1

# Show excluded ranges
netsh int ipv4 show excludedportrange tcp
```

#### Registry Settings

| Registry Path | Value | Type | Description |
|--------------|-------|------|-------------|
| `HKLM\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters` | `MaxUserPort` | DWORD | Max ephemeral port (legacy, pre-Vista: default 5000, max 65534) |
| Same path | `TcpTimedWaitDelay` | DWORD | TIME_WAIT duration in seconds (default 240, min 30, max 300) |

### 3.5 Windows Socket Options for Port Selection

| Socket Option | Description | Available Since |
|--------------|-------------|----------------|
| `SO_REUSE_UNICASTPORT` | Allows multiple sockets to share same local (IP, port) for connection load-balancing | Windows 8 / Server 2012 |
| `SO_PORT_SCALABILITY` | Enlarges effective port pool by relaxing uniqueness constraints; enables port sharing for high-throughput scenarios | Windows 8 / Server 2012 |
| `SO_REUSEADDR` | Allows binding to address in TIME_WAIT | All versions |

#### `SO_REUSE_UNICASTPORT`
Enables **port sharing** across sockets/processes on the same endpoint. The OS load-balances incoming connections across participating sockets[^16].

#### `SO_PORT_SCALABILITY`
Designed for high-concurrency servers to avoid port exhaustion. It relaxes the constraint that each `(local IP, local port)` pair must be globally unique[^17].

### 3.6 Algorithm Classification (per RFC 6056)

Windows primarily implements **Algorithm 1 (Simple Randomization)** from RFC 6056, with:
- CSPRNG for initial random selection
- Linear fallback scan for occupied ports
- No per-connection hashing (unlike Linux's hash-based approach)
- No perturbation tables or parity optimization

---

## 4. Comparative Analysis

### 4.1 Algorithm Comparison

| Feature | Linux | Windows |
|---------|-------|---------|
| **Primary Algorithm** | Hash-based (RFC 6056 Algo 3) | Simple Random (RFC 6056 Algo 1) |
| **Hash Function** | SipHash over (saddr, daddr, dport, secret) | N/A (pure CSPRNG) |
| **Fallback** | Coprime-step traversal of full range | Linear scan |
| **Perturbation** | table_perturb[] adds cross-connection randomness | None |
| **Parity Optimization** | Even/odd separation between connect/bind | None |
| **TIME_WAIT Reuse** | `tcp_tw_reuse` sysctl (fine-grained) | `TcpTimedWaitDelay` (timer-based) |
| **Per-socket Range** | `IP_LOCAL_PORT_RANGE` (Linux 6.3+) | Not available |
| **Deferred Allocation** | `IP_BIND_ADDRESS_NO_PORT` | Not available |
| **Port Sharing** | `SO_REUSEPORT` (kernel load-balanced) | `SO_REUSE_UNICASTPORT` |

### 4.2 Default Ephemeral Port Range

| OS | Default Range | Size | IANA Compliant |
|----|--------------|------|---------------|
| Linux | 32768–60999 | 28,232 | Partial (starts below 49152) |
| Windows (Vista+) | 49152–65535 | 16,384 | Yes |
| FreeBSD | 49152–65535 | 16,384 | Yes |
| IANA Recommendation | 49152–65535 | 16,384 | — |

Linux's wider default range provides more ports but overlaps with the registered port range (1024–49151).

### 4.3 Security Properties

| Property | Linux | Windows |
|----------|-------|---------|
| **Port prediction resistance** | Very high (per-connection hash + perturbation) | High (CSPRNG) |
| **Cross-connection correlation** | Low (different hash per 3-tuple) | Low (independent random draws) |
| **Sequential scanning resistance** | High (coprime step traversal) | Medium (linear fallback) |
| **Reboot persistence** | No (secret regenerated) | No (CSPRNG reseeded) |

### 4.4 Performance Under High Connection Churn

| Scenario | Linux Advantages | Windows Advantages |
|----------|-----------------|-------------------|
| Many connections to same dest | Hash-based offset reduces collisions | — |
| Port exhaustion | `IP_BIND_ADDRESS_NO_PORT` defers allocation | `SO_PORT_SCALABILITY` relaxes constraints |
| TIME_WAIT buildup | `tcp_tw_reuse` allows safe reuse | `TcpTimedWaitDelay` shortens wait |
| Multi-process servers | `SO_REUSEPORT` with eBPF | `SO_REUSE_UNICASTPORT` |

---

## 5. Summary of All Modes/Options

### Linux Port Selection Modes

1. **Default Mode** — Hash-based randomized selection from `ip_local_port_range`, with even/odd parity, coprime step traversal, and perturbation tables
2. **Per-Socket Range** — `IP_LOCAL_PORT_RANGE` socket option overrides system range (step=1, no parity)
3. **Deferred Allocation** — `IP_BIND_ADDRESS_NO_PORT` delays port selection to `connect()` for full 5-tuple awareness
4. **TIME_WAIT Reuse** — `tcp_tw_reuse=1` allows reusing ports from TIME_WAIT sockets
5. **Reserved Ports** — `ip_local_reserved_ports` excludes specific ports from selection
6. **Port Sharing** — `SO_REUSEPORT` allows multiple sockets on same (addr, port)

### Windows Port Selection Modes

1. **Default Mode (Vista+)** — CSPRNG random selection from dynamic port range with linear fallback
2. **Legacy Mode (XP/2003)** — Sequential allocation from 1025–5000
3. **Custom Range** — `netsh int ipv4 set dynamicport tcp` adjusts range
4. **Excluded Ports** — `netsh int ipv4 add excludedportrange` reserves ports
5. **Port Scalability** — `SO_PORT_SCALABILITY` increases effective port pool
6. **Port Sharing** — `SO_REUSE_UNICASTPORT` enables multi-socket port sharing
7. **TIME_WAIT Tuning** — `TcpTimedWaitDelay` registry value adjusts TIME_WAIT duration

---

## 6. Key Implementation Files

### Linux Kernel Source

| File | Purpose |
|------|---------|
| `net/ipv4/inet_hashtables.c` | `inet_hash_connect()`, `__inet_hash_connect()` — core port selection |
| `include/net/inet_hashtables.h` | Hash table structures, bind bucket definitions |
| `net/ipv4/tcp_ipv4.c` | `tcp_v4_connect()` — calls `inet_hash_connect()` |
| `net/ipv4/inet_connection_sock.c` | `inet_csk_get_port()` — bind-side port selection |
| `net/core/secure_seq.c` | `secure_tcp_port_hash()` — SipHash-based port offset |
| `include/net/ip.h` | `inet_sk_port_offset()` declaration |

### Windows (Closed Source)

| Component | Purpose |
|-----------|---------|
| `tcpip.sys` | Kernel-mode TCP/IP driver; contains `TcpAllocatePort()` |
| `AFD.sys` | Ancillary Function Driver; user-kernel socket transition |
| WFP API | Post-selection filtering (cannot modify port choice) |

---

## Confidence Assessment

| Claim | Confidence | Source |
|-------|-----------|--------|
| Linux algorithm details | **High** | Direct kernel source code analysis |
| Linux sysctl/socket options | **High** | Kernel documentation + source |
| Windows Vista+ randomization | **High** | Microsoft documentation |
| Windows pre-Vista sequential | **High** | Microsoft KB articles |
| Windows internal algorithm (CSPRNG + linear fallback) | **Medium** | Inferred from documentation and debugging references; `tcpip.sys` is closed-source |
| RFC compliance mapping | **High** | Direct RFC text analysis |
| `SO_REUSE_UNICASTPORT` / `SO_PORT_SCALABILITY` behavior | **Medium-High** | Microsoft WinSock documentation |
| `IP_LOCAL_PORT_RANGE` per-socket option | **High** | Linux 6.3+ kernel source |

---

## Footnotes

[^1]: [RFC 793 §3.3](https://datatracker.ietf.org/doc/html/rfc793#section-3.3) — Original TCP specification, port number definitions.

[^2]: [RFC 6335 §6](https://datatracker.ietf.org/doc/html/rfc6335#section-6) — IANA port range taxonomy.

[^3]: [RFC 6056 §3](https://datatracker.ietf.org/doc/html/rfc6056#section-3) — Port randomization algorithms.

[^4]: [RFC 4953](https://datatracker.ietf.org/doc/html/rfc4953) — Defending TCP Against Spoofing Attacks.

[^5]: [RFC 1323](https://datatracker.ietf.org/doc/html/rfc1323) — TCP Extensions for High Performance.

[^6]: [`net/ipv4/inet_hashtables.c`](https://github.com/torvalds/linux/blob/master/net/ipv4/inet_hashtables.c) — Linux kernel `__inet_hash_connect()` implementation, lines ~1042-1210.

[^7]: `net/ipv4/inet_hashtables.c` line 1074 — `step = local_ports ? 1 : 2` determines parity scanning mode.

[^8]: `net/ipv4/inet_hashtables.c` lines 1083-1088 — `table_perturb[]` initialization and offset computation.

[^9]: `net/core/secure_seq.c` — `secure_tcp_port_hash()` uses SipHash with `net_secret` for per-connection offset.

[^10]: `net/ipv4/inet_hashtables.c` lines 1096-1113 — Random coprime step width with GCD validation.

[^11]: `net/ipv4/inet_hashtables.c` lines 1090-1094 — Even/odd parity comment and implementation.

[^12]: `net/ipv4/inet_hashtables.c` line 1204 — Post-selection perturbation table update.

[^13]: Linux kernel commit introducing `IP_BIND_ADDRESS_NO_PORT` (kernel 4.2) — defers port allocation to `connect()`.

[^14]: [Microsoft Support: Default dynamic port range changed in Vista/2008](https://support.microsoft.com/en-us/topic/the-default-dynamic-port-range-for-tcp-ip-has-changed-in-windows-vista-and-in-windows-server-2008-3e0c58a9-999a-2fbc-c5a3-4d2ef0a0b2df)

[^15]: Windows `tcpip.sys` uses a CSPRNG seeded at boot; observable via WinDbg breakpoint on `tcpip!TcpAllocatePort`.

[^16]: [Microsoft WinSock: SO_REUSE_UNICASTPORT](https://learn.microsoft.com/en-us/windows/win32/winsock/so-reuse-unicastport-setsockopt)

[^17]: [Microsoft WinSock: SO_PORT_SCALABILITY](https://learn.microsoft.com/en-us/windows/win32/winsock/so-port-scalability-setsockopt)
