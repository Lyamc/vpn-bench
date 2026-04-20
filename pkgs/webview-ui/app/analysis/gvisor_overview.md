# Overview

**gVisor** is **not a VPN**. It is an **application kernel** written in Go that re-implements a Linux-compatible system call surface entirely in userspace. It is included in this VPN comparison because its `pkg/tcpip` package — usually called **netstack** — is reused as a standalone, pure-Go userspace TCP/IP stack by several mesh VPNs in this benchmark suite (notably Tailscale's `wgengine/netstack` and Hyprspace's "service network" feature). This document focuses on gVisor as a building block for VPN data planes, while also covering its primary role as a container sandbox so the security and performance trade-offs are clear.

> **Note on the feature checklists below.** The same section structure as the other VPN overviews is preserved for diff-ability, but most of the rows do not apply cleanly to gVisor: a TCP/IP stack is not a tunneling layer, has no peer concept, no encryption layer, and no NAT traversal. Rows are marked **N/A** wherever the question only makes sense for a VPN. **gVisor should not be added to `feature_matrix.md` and should not be scored against the per-VPN totals there** — the comparison surface is not commensurate.

**Two Distinct Components:**

1. **runsc / Sentry** (`runsc/`, `pkg/sentry/`): An OCI-compatible container runtime that intercepts every guest application syscall and serves it from a Go-based "Sentry" process. This is gVisor's main product and is unrelated to mesh VPNs.

2. **netstack** (`pkg/tcpip/`): A pure-Go, dependency-free implementation of IPv4, IPv6, ARP, ICMP, TCP, UDP, raw sockets, packet sockets, IP fragmentation, NAT, netfilter/iptables, and nftables. The package documentation explicitly states "Netstack aims to be usable independent of gVisor" and it is the part that VPN projects pull in.

**Key Architecture Components:**

- **Sentry** (`pkg/sentry/kernel/`): The application kernel — implements processes, schedulers, signals, VFS, memory management, the syscall table (`pkg/sentry/syscalls/linux/`).
- **Platform** (`pkg/sentry/platform/`): The syscall-interception backend. Available platforms: `systrap` (default since 2023, uses `SECCOMP_RET_TRAP` + `SIGSYS`), `kvm` (uses KVM virtualization extensions), `ptrace` (legacy, uses `PTRACE_SYSEMU`).
- **Gofer** (`runsc/fsgofer/`): A separate process that brokers filesystem access for the sandbox over a 9P/lisafs connection.
- **netstack / tcpip** (`pkg/tcpip/`): The userspace network stack used by both the Sentry and external consumers.
  - `pkg/tcpip/stack/`: Stack object, NICs, neighbor table, packet buffers, GRO engine.
  - `pkg/tcpip/transport/{tcp,udp,icmp,packet,raw}/`: Transport-layer protocols.
  - `pkg/tcpip/network/{ipv4,ipv6,arp}/`: Network-layer protocols including fragmentation, IGMP, MLD, NDP.
  - `pkg/tcpip/link/{fdbased,xdp,channel,sharedmem,tun,...}/`: Pluggable link-layer endpoints.
  - `pkg/tcpip/nftables/`, `pkg/sentry/socket/netfilter/`: nftables and iptables-compatible filtering.
- **runsc** (`runsc/`): The OCI runtime CLI that wires Sentry, Gofer, platforms, and netstack together for container execution.
- **Shim** (`shim/`): containerd integration shim (`containerd-shim-runsc-v1`).

**Where this matters for VPNs:** Tailscale uses netstack to terminate tunneled TCP connections in userspace and tune them aggressively (8 MB buffers, Reno, RACK disabled) to absorb loss/reordering. Hyprspace uses netstack for its in-VPN "service network." Neither project uses Sentry, runsc, the platforms, or any of the syscall implementation — they import only `gvisor.dev/gvisor/pkg/tcpip` and friends.

# Protocol

gVisor itself does not define a wire protocol. Its netstack implements **standard internet protocols** that the consuming application drives:

**Network-layer protocols implemented in `pkg/tcpip/network/`:**
- **IPv4** (`ipv4/ipv4.go`): Full IPv4 with reassembly, IGMPv2/v3, ICMPv4
- **IPv6** (`ipv6/`): Full IPv6 with NDP, MLDv1/v2, ICMPv6, extension headers, fragment reassembly
- **ARP** (`arp/`): ARP request/response handling for Ethernet link layers

**Transport-layer protocols in `pkg/tcpip/transport/`:**
- **TCP** (`tcp/`): A full RFC-compliant TCP implementation with SACK, RACK loss detection (default), window scaling, timestamps, fast retransmit, CUBIC and Reno congestion control, SYN cookies, TIME_WAIT reuse, keep-alive
- **UDP** (`udp/`)
- **ICMP** (`icmp/`)
- **Raw sockets** (`raw/`)
- **Packet sockets** (`packet/`)

**Link-layer endpoint backends in `pkg/tcpip/link/`:**
- **fdbased** (`fdbased/`): The most common backend; reads/writes packets via a file descriptor (TUN, TAP, AF_PACKET, seqpacket sockets). Supports `readv`, `recvmmsg`, and `PACKET_RX_RING` (PACKET_MMAP) dispatch modes. This is what `runsc` uses against the AF_PACKET socket attached to the sandbox veth.
- **xdp** (`xdp/`): AF_XDP-backed link endpoint for kernel-bypass NIC ingress/egress.
- **channel** (`channel/`): In-process Go channels — used by Tailscale and Hyprspace to inject already-decapsulated packets directly into netstack with no syscalls.
- **sharedmem** (`sharedmem/`): Shared-memory ring buffers for inter-process zero-copy networking.
- **tun** (`tun/`): A virtual `/dev/net/tun` Device implementation backing the Sentry's TUN/TAP support.
- **muxed**, **nested**, **pipe**, **veth**, **loopback**, **packetsocket**, **sniffer**, **waitable**, **qdisc/fifo**: Composability and tooling endpoints.
- **ethernet** (`ethernet/`): Wraps a non-Ethernet endpoint to add Ethernet framing.

**Sentry-side wire protocols:** `runsc` itself uses 9P (`pkg/p9`) and lisafs (`pkg/lisafs`) over Unix sockets to talk to the Gofer, urpc (`pkg/urpc`) for control, and flipcall (`pkg/flipcall`) for fast cross-process method calls. None of these are network protocols that touch peers.

## Protocol Features Checklist

### Transport
> The matrix rows below ask which transport a VPN can tunnel its encrypted traffic over (e.g. WireGuard-over-UDP, DERP-over-TCP). gVisor netstack is not a tunneling layer — it terminates these protocols as endpoints for application sockets. The rows are therefore **N/A** for gVisor; what netstack actually implements is noted instead.

- [ ] **UDP transport** - N/A (no tunnel). netstack does implement UDP as a transport endpoint in `pkg/tcpip/transport/udp/`.
- [ ] **TCP fallback** - N/A (no tunnel). netstack does implement TCP as a transport endpoint in `pkg/tcpip/transport/tcp/`.
- [ ] **QUIC support** - N/A; no QUIC implementation in netstack regardless.
- [ ] **WebSocket support** - N/A; WebSocket is application-layer.

### IP Support
- [x] **IPv4 support** - Full IPv4 with fragmentation, IGMP, ICMPv4
- [x] **IPv6 support** - Full IPv6 with NDP, MLD, extension headers
- [x] **Dual-stack** - Stack supports both protocols simultaneously per NIC

### Network Layer Mode
- [x] **Layer 3 (IP) mode** - Native IP routing via `Stack.SetRouteTable`
- [ ] **Layer 2 (Ethernet) mode** - N/A in the VPN sense (gVisor exposes no overlay to user devices). netstack does have Ethernet link endpoints, ARP, and a neighbor table.
- [ ] **Bridging support** - N/A in the VPN sense. netstack can forward between NICs (`link/nested`, `veth`), which is bridge-like at the stack level only.

### Advanced
- [ ] **Multipath/bonding** - No path bonding (single egress per route)
- [x] **QoS/traffic shaping** - `link/qdisc/fifo` queueing discipline; pluggable qdisc interface
- [ ] **Multicast support** - N/A in the VPN sense (no overlay to carry multicast across). netstack does implement IGMPv2/v3, MLDv1/v2, multicast routing tables, and `IP_ADD_MEMBERSHIP` for guest applications.

# Encryption

**gVisor netstack does not implement any transport encryption.** It is a plain TCP/IP stack — TLS, IPsec, WireGuard, Noise, and similar are entirely the responsibility of the application above it (or, for VPN consumers like Tailscale, of the surrounding WireGuard layer that has already decrypted the packets *before* they are handed to netstack).

What gVisor does provide on the cryptographic side:

- **`pkg/crypto/`**: A small support package used by the Sentry for in-kernel crypto needs (e.g. backing `getrandom(2)`, ChaCha20-based RNG state). It is not exposed to peers and is not used by netstack.
- **TCP sequence number / timestamp secret randomization** (`pkg/tcpip/transport/tcp/protocol.go`): netstack reads 16 bytes of secure random for `seqnumSecret` and `tsOffsetSecret` at startup so that ISN and TS offset generation is unpredictable across connections.
- **SipHash / Jenkins hashing** (`pkg/tcpip/hash/`): Used for socket hashing, GRO bucket selection, and similar — not security-critical.
- **`getrandom`, `/dev/urandom`** are implemented by the Sentry on behalf of guest applications (see `pkg/sentry/devices/tundev`, `pkg/rand`).

For the Sentry's own host-facing channel to the Gofer, communication runs over inherited Unix sockets in the same user namespace, so there is no on-wire encryption required.

## Encryption Checklist

### Key Exchange
- [ ] **Modern key exchange** - N/A (no encryption layer)
- [ ] **Perfect Forward Secrecy** - N/A
- [ ] **Post-quantum readiness** - N/A
- [ ] **Key rotation** - N/A

### Symmetric Encryption
- [ ] **Authenticated encryption** - N/A (netstack carries plaintext IP)
- [ ] **Hardware-accelerated crypto** - N/A
- [ ] **Constant-time operations** - N/A

### Protocol Security
- [x] **Replay protection** - TCP sequence-number windowing only (not cryptographic)
- [ ] **Noise Protocol or equivalent** - N/A
- [ ] **No cleartext metadata** - N/A; all packet data is plaintext

# Performance

netstack is the part of gVisor that VPN benchmarks care about, so this section focuses on it. gVisor's own `g3doc/architecture_guide/performance.md` is candid: networking is the area where the project sees the largest **implementation cost** versus a native Linux kernel, and most workloads pay a measurable throughput penalty when running under runsc.

**Threading Model:**

- **Goroutine-per-component**: Link endpoints (e.g. `fdbased`) spawn one or more dispatcher goroutines that read raw packets from their FD and call into the network dispatcher.
- **TCP processor pool** (`pkg/tcpip/transport/tcp/dispatcher.go`): TCP creates `runtime.GOMAXPROCS(0)` processor goroutines at startup. Endpoints are queued onto a processor by hashing the 4-tuple, so per-flow ordering is preserved while different flows fan out across cores.
- **Inline UDP/ICMP processing**: Datagram protocols are processed on the dispatcher goroutine with no queueing.
- **Outgoing packets**: Sent on whatever goroutine called `Write` (syscall, TCP processor, or link dispatcher) until they reach a queueing discipline; `link/qdisc/fifo` then has its own writer goroutine that batches packets out the link endpoint.
- **Link FD fan-out** (`fdbased`): Multiple FDs can be passed in; if the underlying socket is `AF_PACKET`, the endpoint enables `PACKET_FANOUT` so the host kernel hashes packets across FDs and per-flow ordering is preserved without netstack coordination.

**Link-layer Packet I/O Modes** (`pkg/tcpip/link/fdbased/endpoint.go`):

- **`Readv`** — one packet per `readv()` syscall, the universal fallback.
- **`RecvMMsg`** — uses `recvmmsg()` to drain multiple packets per syscall (sockets only).
- **`PacketMMap`** — uses `PACKET_RX_RING` for kernel-to-userspace zero-copy packet delivery on AF_PACKET FDs. This is what `runsc` uses to pull packets off the sandbox veth.
- **`AF_XDP`** (`link/xdp/`) — kernel-bypass via UMEM rings.
- **`BatchSize = 47`** for outbound writes, sized so a single 65 KB GVisor-GSO TCP segment (which fragments to 46×1420 + 1×216) fits exactly in one batch.

**GSO / GRO:**

- **GSO** (`stack/gso.go`, `link/fdbased/endpoint.go:598`): Two flavors — `HostGSOSupported` (defers segmentation to the kernel via `virtio_net_hdr` on the AF_PACKET FD, up to ~64 KB) and `GVisorGSOSupported` (in-Go segmentation when the link layer can't offload). TCP fast-path emits GSO super-segments instead of one MSS-sized packet at a time, drastically reducing per-packet overhead.
- **GRO** (`pkg/tcpip/stack/gro/gro.go`): A receive-side GRO engine with 8 buckets × 8 packets each, coalescing into 64 KB super-packets before they hit TCP. Enabled per link endpoint via the `GRO` option, which `runsc` exposes as `--gvisor-gro`.

**TCP buffer defaults** (`pkg/tcpip/transport/tcp/protocol.go`):

- `MinBufferSize = 4 KiB`
- `DefaultSendBufferSize = 1 MiB`
- `DefaultReceiveBufferSize = 1 MiB`
- `MaxBufferSize = 4 MiB`
- `moderateReceiveBuffer = true` — receive-window auto-tuning enabled by default.
- `sackEnabled = true`
- Default congestion control: **Reno**, with CUBIC also available (`availableCongestionControl: []string{"reno", "cubic"}`). gVisor exposes `tcp.NewProtocol` (Reno) and `tcp.NewProtocolCUBIC` constructors. There is a `TODO(b/345835636)` to make CUBIC the default everywhere; the existence of this TODO is why Tailscale explicitly forces Reno (its release notes call out a CUBIC integer-overflow bug).
- Default loss recovery: **`TCPRACKLossDetection`**. Tailscale explicitly disables RACK and falls back to Reno-style fast retransmit because gVisor's RACK implementation triggers spurious retransmits on real-world reordering.

**Buffer / packet management:**

- `pkg/buffer/`: Reference-counted, scatter/gather buffer chains shared by all packets. Avoids per-packet allocations on the hot path.
- `pkg/tcpip/stack/PacketBuffer`: Holds link/network/transport header offsets and a buffer chain; supports clone/slice without copying payload data.
- `pkg/tcpip/segment.go`: TCP segment objects are reused via a free list; the segment heap reorders out-of-order arrivals.
- `pkg/sleep/`: A custom multi-waker primitive used everywhere instead of channels to avoid select-case overhead.

**MTU handling:**

- `defaultDevMtu = 1500` for the TUN device (`link/tun/device.go`)
- XDP endpoint: `MTU = 1500` (sized so the packet fits in a 2048-byte XDP frame)
- PMTUD is honored at the IPv4/IPv6 layer; ICMP "fragmentation needed" / "packet too big" updates per-route MTU.

**Sentry-side performance overhead** (relevant only when running under runsc, not when consuming netstack as a library): every guest syscall round-trips through the platform layer (`systrap`/`kvm`/`ptrace`). gVisor's own performance guide concedes that `redis` and other syscall-bound workloads pay heavy structural cost; it explicitly states "networking is mostly bound by **implementation costs**, and gVisor's network stack is improving quickly."

## Performance Optimizations Checklist

### Threading
- [x] **Multi-threaded processing** - Per-CPU TCP processor pool sized to `GOMAXPROCS`
- [x] **Per-core packet queues** - 4-tuple hashed endpoint queues; AF_PACKET fanout for link FDs

### Packet I/O
- [x] **Batch UDP receives** - `RecvMMsg` dispatch mode uses `recvmmsg()`
- [x] **Batch UDP sends** - `BatchSize = 47` write batches via `writev`/`sendmmsg`
- [x] **Large batch sizes** - 47-packet outbound batch tuned for GSO; PACKET_MMAP rings on ingress

### UDP Offload
- [ ] **UDP GSO (Generic Segmentation Offload)** - Not supported. `pkg/tcpip/stack/registration.go:1428` defines only `GSONone`, `GSOTCPv4`, `GSOTCPv6`, `GSOGvisor` — there is no UDP GSO type. netstack *does* support TCP GSO (host-offloaded via `virtio_net_hdr` plus an in-Go `GVisorGSOSupported` fallback), which is what TCP fast-path uses.
- [ ] **UDP GRO (Generic Receive Offload)** - Not supported. The `pkg/tcpip/stack/gro/` engine (8×8 buckets, 64 KB max super-packet) coalesces TCP segments only, not UDP datagrams.

### Buffer Management
- [x] **Buffer pool reuse** - `pkg/buffer` chained refcounted buffers; segment free list
- [ ] **Large UDP socket buffers** - UDP defaults are modest; tuning is left to the consumer (Tailscale overrides explicitly)

### Userspace TCP Stack (optional)
- [x] **Userspace TCP implementation** - This *is* the userspace TCP stack
- [x] **Large TCP RX/TX buffers** - 1 MiB defaults, 4 MiB max; consumers like Tailscale override to 8 MiB / 6 MiB
- [ ] **Tuned congestion control** - Knobs are present (Reno + CUBIC available, SACK on by default), but the *defaults are not tuned for VPN-style adverse networks*. Tailscale explicitly forces Reno on top of netstack to work around a CUBIC integer-overflow bug — the in-tree `TODO(b/345835636)` to make CUBIC the default exists precisely because the default is known-problematic. The tuning lives in the consumer, not in gVisor.
- [ ] **Reordering tolerance** - Despite shipping SACK, RACK, and a segment heap, gVisor's *default* RACK loss-detection (`recovery: TCPRACKLossDetection`) is what Tailscale found triggers spurious retransmits under real-world reordering — Tailscale disables RACK entirely when using netstack. Out-of-the-box behavior is poor for reordering; the consumer must intervene.

### Receive Path
- [x] **TCP/packet coalescing on ingress** - GRO engine coalesces TCP segments before TCP processing
- [x] **RX checksum offload** - `LinkEndpointCapabilities.CapabilityRXChecksumOffload` skips redundant checksum validation when the link layer attests it

### MTU Handling
- [x] **Conservative MTU** - 1500 default; Tailscale overrides downward
- [x] **Path MTU discovery** - PMTUD via ICMP "frag needed" / "packet too big"

### Peer Management
- [ ] **Lazy peer removal** - N/A (no peer concept; ARP/NDP neighbor cache only)
- [ ] **Endpoint caching** - N/A
- [ ] **Efficient keepalive timers** - TCP keepalive only (2h idle / 75s interval / 9 probes)

### Packet Processing
- [x] **Zero-allocation parsing** - `header.IPv4`/`header.TCP` etc. are byte-slice views, not allocations
- [x] **Zero-copy filtering** - `PacketBuffer` slices/clones share buffer chains; netfilter/nftables operate on the same buffer

### State Synchronization
- [ ] **Delta updates** - N/A (no control plane)
- [ ] **Compression** - N/A

### Data Plane Compression
- [ ] **Tunnel compression** - N/A (not a tunnel)
- [ ] **Configurable compression level** - N/A

# Security

This is gVisor's primary design goal — but the security boundary is around the **Sentry**, not around netstack. A VPN that imports `gvisor.dev/gvisor/pkg/tcpip` gets the network stack but **none of the sandbox guarantees**: there is no syscall interception, no platform layer, no seccomp filter, no Gofer. The threat model below applies to `runsc` users, not to Tailscale or Hyprspace.

**Sentry threat model** (from `g3doc/architecture_guide/security.md`):

The Sentry is a Go re-implementation of the Linux system call surface. Its job is to ensure that an attacker who has full control of a guest process cannot reach the host kernel's syscall implementations. The defense layers are:

1. **No syscall pass-through.** Every supported syscall has an independent implementation in `pkg/sentry/syscalls/linux/`. A guest cannot craft arguments to a host kernel call directly; the Sentry interprets the call and may make a *different* host call as part of servicing it.
2. **Memory-safe Go.** The Sentry is a pure-Go binary with no CGo. Buffer overflows, use-after-free, and similar C-kernel bug classes are categorically eliminated for the kernel implementation itself. `unsafe` usage is confined to files suffixed `*_unsafe.go` to make auditing tractable.
3. **Minimal host syscall surface.** The Sentry runs under a restrictive seccomp filter (`pkg/seccomp/`). It is not allowed to `open(2)` files, create new sockets, or call most syscalls. Allowed calls are essentially: FD duplication/closing, futex/synchronization, timers, signal management, memory mapping, and the single AF_PACKET socket used for networking.
4. **No raw filesystem access.** The Sentry runs in an empty mount namespace. Filesystem requests are forwarded to the **Gofer** process, which itself runs under a separate seccomp filter and a chroot. With `directfs` enabled the Sentry holds FDs directly but still cannot `open(2)` new ones.
5. **Platform interception** (`pkg/sentry/platform/`):
   - **systrap** (default): `seccomp` → `SECCOMP_RET_TRAP` → `SIGSYS` → Sentry handler.
   - **kvm**: Sentry runs as both guest and VMM via KVM ioctls; uses hardware virt extensions to switch address spaces cheaply.
   - **ptrace**: Legacy `PTRACE_SYSEMU` mode, kept for environments without KVM and without `SECCOMP_RET_TRAP`.
6. **Defense-in-depth project rules**: No CGo, no external imports inside core packages, all `unsafe` isolated and named, continuous fuzzing of the Sentry, security advisories tracked at `SECURITY.md`.

**What gVisor does NOT defend against** (per its own documentation):

- Hardware side channels (Spectre, L1TF, Rowhammer, MDS) — gVisor relies on host kernel and microcode mitigations.
- Resource exhaustion / DoS — defers to host cgroups.
- Bugs in supported syscall implementations themselves; gVisor has had several CVEs (a public list lives at `SECURITY.md`) — most have been bugs in `pkg/sentry/...` syscall implementations rather than in netstack or the platform layer.
- Anything outside the syscall vector — e.g. an exploitable network service inside the sandbox is still exploitable; gVisor does not patch user code.

**netstack-specific security considerations** (relevant when used as a library):

- **Plaintext stack.** netstack carries cleartext IP. Anything sensitive must be encrypted at a higher layer.
- **Injection surface.** A VPN that pipes attacker-controlled bytes from a peer into a `channel` link endpoint is exposing the netstack parser. netstack's parsers (in `pkg/tcpip/header/`) are pure-Go bounds-checked code, but the attack surface is still that of a TCP/IP stack and has had its own CVEs.
- **No memory limit by default** on the per-flow segment heap and reassembly queues — caller must impose limits via stack options if exposed to untrusted peers.
- **Random secrets** for ISN and timestamp offsets read from `s.SecureRNG()`, which on Linux is `getrandom(2)`. If the consumer wires up an `InsecureRNG` or a deterministic test RNG, ISN predictability becomes a problem.
- **iptables/nftables compatibility code** (`pkg/sentry/socket/netfilter/`, `pkg/tcpip/nftables/`) is large and complex; it has historically been a source of bugs.

## Security Features Checklist

### Network Security
- [ ] **Rate limiting** - No connection-level rate limiting in netstack itself; SYN flood mitigated via SYN cookies (`alwaysUseSynCookies` option)
- [x] **Stateful packet filter** - iptables (conntrack) and nftables compatibility layers (`pkg/tcpip/transport/tcpconntrack/`, `pkg/sentry/socket/netfilter/`)
- [x] **Fine-grained ACLs** - Full iptables/nftables rule sets supported
- [ ] **Capability-based access** - N/A (kernel-style filtering, not capability tokens)

### Identity & Authentication
- [ ] **Identity validation** - N/A (no peer identity concept)
- [ ] **Signed configuration updates** - N/A
- [ ] **Certificate pinning** - N/A

### Implementation
- [x] **Memory-safe language** - Pure Go, no CGo, `unsafe` isolated to `*_unsafe.go` files
- [x] **Privilege separation** - Sentry / Gofer / runsc are separate processes with independent seccomp filters (runsc only)
- [x] **Sandboxing** - This is gVisor's whole purpose for runsc users
- [x] **Audit logging** - `seccheck` framework (`pkg/sentry/seccheck/`) emits structured audit events for syscalls, file ops, etc.

# NAT Traversal

gVisor's networking subsystem performs **NAT** in the iptables/nftables sense — it can SNAT/DNAT packets traversing the stack, and it has a connection-tracking module (`tcpconntrack`) — but it does **not** perform NAT *traversal*. There is no STUN, no UPnP, no hole-punching, no relay protocol. A VPN that uses netstack must layer all of that on top.

What is implemented:

- **iptables NAT targets** (`pkg/sentry/socket/netfilter/`): SNAT, DNAT, MASQUERADE, REDIRECT
- **nftables NAT** (`pkg/tcpip/nftables/`): equivalent NAT verdicts and chains
- **Connection tracking** (`pkg/tcpip/transport/tcpconntrack/`, `pkg/tcpip/stack/conntrack.go`): Tracks TCP/UDP/ICMP flows for stateful NAT and conntrack matches
- **Port allocation / rewriting** for NAT and `bind(2)` with `IP_BIND_ADDRESS_NO_PORT`

What is *not* implemented:

- STUN, TURN, ICE
- UPnP / NAT-PMP / PCP port mapping
- UDP hole punching helpers
- Any kind of relay protocol
- ALG (application-layer gateways) for NAT-traversed protocols

## NAT Traversal Checklist

### Discovery
- [ ] **STUN support** - Not implemented
- [ ] **Multiple STUN servers** - N/A
- [ ] **NAT type detection** - Not implemented

### Port Mapping
- [ ] **UPnP port mapping** - Not implemented
- [ ] **NAT-PMP support** - Not implemented
- [ ] **PCP support** - Not implemented

### Hole Punching
- [ ] **UDP hole punching** - Not implemented
- [ ] **Symmetric NAT handling** - N/A
- [ ] **Rendezvous coordination** - N/A

### Fallback
- [ ] **Relay fallback** - N/A
- [ ] **Multiple relay regions** - N/A
- [ ] **Automatic relay selection** - N/A
- [ ] **TCP relay support** - N/A

# Local Routing

netstack ships a **classical Linux-style routing table** rather than any peer-discovery mechanism. Routing decisions are made per packet by `Stack.FindRoute(...)`, which walks the route table installed via `Stack.SetRouteTable`.

**Routing model:**

- **Route table entries**: destination prefix, gateway, NIC, source hint, scope, MTU
- **Per-NIC primary addresses**: First-, secondary-, and never-primary endpoint promotion controlled by `tcpip.PrimaryEndpointBehavior`
- **Forwarding** can be enabled per NIC and per network protocol — gVisor netstack can act as an IP forwarder/router between link endpoints, which is how `runsc` networks containers and how Tailscale's exit-node functionality works
- **Multicast routing tables** (`pkg/tcpip/stack/iptables.go`, IPv4/IPv6 multicast forwarding) for IGMP/MLD-based group membership
- **Loopback** via `link/loopback` endpoint
- **Neighbor table** (`stack/neighbor_cache.go`): ARP/NDP-driven L2 resolution with the standard Linux state machine (`Incomplete` → `Reachable` → `Stale` → `Delay` → `Probe` → `Failed`)

There is no concept of peers, "subnets", LAN preference, exit nodes, or split tunneling at the netstack layer — those are all things the consuming VPN must implement and then encode as routes/NICs in the netstack instance it owns.

## Local Routing Checklist

### LAN Discovery
- [ ] **Broadcast/multicast discovery** - No service discovery; mDNS-style discovery would be an application above netstack
- [ ] **Direct path advertisement** - N/A
- [ ] **Same-subnet detection** - Only via the standard route-table prefix match

### LAN Optimization
- [ ] **Automatic LAN preference** - N/A; the route table determines the egress NIC
- [ ] **Trusted path mode** - N/A
- [ ] **LAN-only mode** - N/A

### Routing Features
- [x] **Subnet routes** - Standard CIDR prefix routes via `Stack.SetRouteTable`
- [x] **Full tunnel mode** - A `0.0.0.0/0` / `::/0` route is just another table entry
- [x] **Split tunneling** - Achieved by populating the route table selectively
- [ ] **Route priorities** - No metric/preference field; first matching route wins (unlike Linux)

# Central Point of Failure

gVisor has **no controller, no registry, no signaling server, and no infrastructure dependencies** of any kind. A `runsc`-launched sandbox is a single process tree (Sentry + Gofer + optional metric server) that runs entirely on one host. The only "control plane" is the OCI runtime invocation from the container manager (`containerd`, `dockerd`, `kubelet`), and once the sandbox is up it has no further dependence on that manager.

For netstack used as a library, the question is even simpler: it is a Go package linked into the consuming binary. It has no out-of-process state and no external dependencies.

**What can fail:**

- **Sentry crash** → entire sandbox dies; the container manager restarts it the same way it would restart a crashed `runc` container.
- **Gofer crash** → filesystem becomes inaccessible inside the sandbox; the Sentry detects the broken connection and shuts down.
- **Platform driver unavailable** (e.g. KVM disabled in BIOS while `--platform=kvm` is requested) → `runsc` fails to start the sandbox at all.

There is no analogue to a Headscale or NebulaCert to lose.

## Resilience Checklist

### Offline Operation
- [x] **Existing connections survive** - The stack has no upstream to lose contact with
- [x] **Local state caching** - All state is local
- [x] **Cached credentials** - N/A
- [x] **Graceful degradation** - N/A (nothing to degrade from)

### Redundancy
- [x] **Self-hosted controller** - N/A — there is no controller
- [x] **Controller redundancy** - N/A
- [x] **Relay redundancy** - N/A
- [x] **No single root of trust** - N/A; trust boundary is the host kernel

### Efficiency
- [ ] **Delta/incremental updates** - N/A
- [ ] **Long polling / push updates** - N/A
- [ ] **Configurable sync interval** - N/A

# Authentication

gVisor has **no notion of node identity, peer authentication, or enrollment**. It is a single-host runtime, and netstack is a single-process library. The closest things to "authentication" are:

- **Container identity** is whatever the OCI runtime / container manager passes in via the OCI bundle (`config.json`). gVisor inherits the container's user namespace, capabilities, and seccomp profile but does not authenticate them — that is the host container manager's job.
- **Sentry ↔ Gofer channel** is an inherited Unix socket pair created at fork time; there is no on-wire authentication because the channel is by construction private to the sandbox.
- **runsc CLI** trusts whoever can invoke it on the host, exactly like `runc`.
- **Inside the sandbox**, the Sentry implements the full Linux `setuid`/`setgid`/`capabilities`/`SELinux` story (`pkg/sentry/kernel/auth/`) and presents these to guest applications, but this is *guest-internal* authorization, not authentication of the sandbox to anyone else.

For VPN consumers of netstack, identity and authentication are wholly the surrounding VPN's responsibility — netstack has no peer concept to authenticate.

## Authentication Checklist

### Enrollment Methods
- [ ] **Pre-authentication keys** - N/A
- [ ] **OAuth/OIDC** - N/A
- [ ] **Interactive login** - N/A
- [ ] **CLI authentication** - N/A

### Authorization
- [ ] **Admin approval workflow** - N/A
- [ ] **Automated enrollment rules** - N/A
- [ ] **Ephemeral nodes** - N/A
- [ ] **Node expiry** - N/A

### Identity
- [ ] **Stable device identity** - N/A
- [ ] **Identity portability** - N/A
- [ ] **Multi-user support** - The Sentry implements full Linux multi-user semantics for guest processes, but no multi-tenant identity for the sandbox itself

# Platform Support

**runsc / Sentry:**

- **Linux x86_64** — Primary, fully supported, all platforms (`systrap`, `kvm`, `ptrace`)
- **Linux ARM64** — Supported, all platforms
- **macOS** — Build/test of some packages only via `brew install bazel@8`; not a runtime target — runsc cannot sandbox containers on macOS
- **Windows / FreeBSD / OpenBSD** — Not supported as runtime targets
- **Other architectures** — README notes "Other architectures may become available in the future"
- **Linux kernel requirement**: 4.14.77 or newer (older platforms link in the docs)
- **Container runtimes**: Docker (via `--runtime=runsc`), containerd (via `containerd-shim-runsc-v1` in `shim/`), Kubernetes (via the runtime class mechanism, used in production by GKE Sandbox)
- **Webhook**: `webhook/` ships a Kubernetes admission webhook that injects the `runsc` runtime class

**netstack as a library:**

- **Anywhere Go runs.** netstack is pure Go. The only platform-gated parts are link endpoints — `link/fdbased` and `link/xdp` are `//go:build linux`, but `link/channel`, `link/loopback`, `link/sharedmem`, and friends are portable. This is why Tailscale and Hyprspace can use it on Linux, macOS, Windows, and BSD without conditional code.

**Container support:**

- runsc is a container runtime. It is used by Docker, containerd, Kubernetes, and (notably) GKE Sandbox. The `images/` directory holds the canonical build environment.

**Filesystem implementations** (`pkg/sentry/fsimpl/`): ext4 (read-only via `pkg/erofs` and direct), tmpfs, proc, sys, devpts, devtmpfs, fuse, nsfs, overlay, host, gofer, lisafs, cgroupfs, mqueue, signalfd, eventfd, timerfd, pipefs, sockfs, verity (read-only authenticated FS), user, iouringfs.

**Platform implementations** (`pkg/sentry/platform/`):

- **systrap** — Default since mid-2023, uses `seccomp` `SECCOMP_RET_TRAP` + `SIGSYS`. Replaces ptrace.
- **kvm** — Uses host KVM virtualization extensions; best on bare-metal.
- **ptrace** — Legacy `PTRACE_SYSEMU`; works in nested VMs without virt extensions, but slow. Officially superseded.

## Platform Checklist

### Desktop/Server
- [x] **Linux** - Full runsc support on x86_64 and ARM64; netstack is platform-independent
- [ ] **macOS** - No runsc runtime; netstack itself builds and runs on macOS as a library
- [ ] **Windows** - No runsc; netstack builds as a library
- [ ] **FreeBSD/OpenBSD** - No runsc; netstack builds (link endpoints are Linux-gated)

### Mobile
- [ ] **iOS** - Not supported
- [ ] **Android** - Not officially supported as a runtime; netstack-as-library has been shipped on Android by Tailscale's iOS/Android apps

### Implementation
- [ ] **Kernel-mode datapath** - The whole point is *not* to be in-kernel
- [x] **Userspace implementation** - Pure-userspace Sentry and pure-Go netstack
- [x] **Container support** - runsc is an OCI container runtime; integrates with Docker, containerd, Kubernetes, and the containerd shim
