# Chapter 17 — Linux and Networking for SRE

## 17.1 Concept Explanation

Every production system runs on Linux. Every distributed system runs on networks. An SRE who is not comfortable with Linux and networking is an SRE who calls someone else when things break. This chapter covers what you must know.

We will not turn you into a kernel developer. We will turn you into someone who can debug a production node confidently, read system metrics, understand the network layer, and not be surprised by the difference between "CPU usage" and "load average."

## 17.2 Linux Process Model

Every running program is a process with:
- A unique PID.
- A parent process (everything descends from PID 1, init or systemd).
- An open file table.
- A virtual memory map.
- Permissions (user, group).
- A state (running, sleeping, zombie, stopped).

The kernel manages processes. The kernel itself is not a process — it lives in kernel space, separate from user space. System calls cross from user space to kernel space (open, read, write, fork, exec).

Understanding this matters because most production problems are process problems: too many, leaking memory, deadlocked, getting OOM-killed.

## 17.3 Memory in Linux

Linux uses **virtual memory**. Each process sees its own address space; the kernel maps virtual addresses to physical pages.

Key concepts:
- **Resident Set Size (RSS)** — physical memory in use.
- **Virtual size (VSZ)** — total address space (mostly meaningless).
- **Page cache** — file data cached by the kernel.
- **Swap** — disk-backed memory when RAM is full.
- **OOM Killer** — kernel mechanism that kills processes when memory is exhausted.

A common surprise: "free memory" is rarely 0 on a healthy Linux system because page cache uses any spare RAM. That memory is reclaimable. `free -h` shows both.

When memory is truly exhausted, the OOM killer chooses victims based on heuristics. Configure your important processes with `oom_score_adj` to control selection.

## 17.4 CPU and Load Average

Linux load average is the average number of processes runnable or in uninterruptible sleep over 1, 5, and 15 minutes.

A common confusion: load average is not CPU percent.
- Load average 1.0 on a 4-core machine ≈ 25% busy.
- Load average 4.0 on a 4-core machine ≈ 100% busy.
- Load average 8.0 on a 4-core machine = 100% busy + queue of 4 waiting.

Sustained load > core count means overload. Tools:
- `uptime` shows load averages.
- `top` / `htop` show per-process CPU and memory.
- `vmstat 1` shows aggregate CPU breakdown over time.

## 17.5 Disk and I/O

Disk I/O is often the bottleneck. Tools:
- `df -h` — disk space.
- `du -h` — directory size.
- `iostat 1` — disk read/write throughput.
- `iotop` — per-process I/O.
- `lsof` — open files (and sockets).

A full disk causes mysterious failures (logs cannot write, databases corrupt). Monitor disk space proactively.

## 17.6 Networking Fundamentals

Linux networking sits on top of the TCP/IP stack. SREs need to know:

**OSI / TCP-IP layers in practice:**
- Layer 2 — Ethernet, MAC addresses, ARP.
- Layer 3 — IP, routing.
- Layer 4 — TCP, UDP, ports.
- Layer 7 — HTTP, gRPC, application protocols.

**IP addressing:**
- IPv4 vs IPv6.
- Private ranges (10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16).
- CIDR notation.
- NAT for outbound from private networks.

**Ports:**
- Well-known (<1024) require privilege.
- Ephemeral (typically 32768-60999) for outbound connections.
- TIME_WAIT state — closed connections held for cleanup.

**Sockets:**
- TCP for reliable, ordered streams.
- UDP for fast, lossy, no-handshake.
- TLS layered on TCP.

## 17.7 TCP Lifecycle

A TCP connection lifecycle:
1. **SYN** — client requests connection.
2. **SYN-ACK** — server acknowledges.
3. **ACK** — client confirms (3-way handshake complete).
4. Data exchange.
5. **FIN** from either side.
6. **FIN-ACK**.
7. **TIME_WAIT** — keep socket around briefly.
8. **CLOSED**.

When connections fail, knowing where in this cycle helps diagnose. `tcpdump` captures, `Wireshark` analyzes.

Common production issues:
- **Too many TIME_WAIT sockets** — exhausted ephemeral ports on outbound-heavy hosts. Fix with connection pooling.
- **SYN floods** — DDoS pattern. Defended by SYN cookies, kernel tunables.
- **Retransmit storms** — flaky network, latency spikes.

## 17.8 DNS

DNS resolves names to IP addresses. Production issues from DNS:

- **Slow DNS** — every connection waits for resolution. Cache aggressively.
- **DNS outages** — your service goes down when nothing else has changed.
- **TTL issues** — caches hold stale results past intended changes.
- **Wildcard pitfalls** — wildcards match more than expected.

Tools:
- `dig` — query DNS records.
- `nslookup` — older alternative.
- `host` — simple lookup.

Always check DNS when debugging connection issues. "It's never DNS" but it often is.

## 17.9 HTTP and TLS Fundamentals

**HTTP/1.1** — request/response, one in flight per connection (or pipelined).

**HTTP/2** — multiplexed, binary, header compression, server push.

**HTTP/3** — over QUIC (UDP-based), better on lossy networks.

**TLS** — encryption layer. Important details:
- Certificate validation (chain of trust, expiry).
- Cipher suites (some are weak; tune).
- SNI (Server Name Indication) — multiple certs per IP.
- mTLS — mutual authentication, both sides have certs.
- Session resumption — avoid full handshakes.

Certificate expiry is a recurring outage cause. Automate renewal (Let's Encrypt + cert-manager).

## 17.10 Load Balancers

Two main types:

**L4 (TCP) load balancer.** Routes connections by IP/port. Cheap, fast, protocol-agnostic. AWS NLB, HAProxy at L4.

**L7 (HTTP) load balancer.** Routes by HTTP attributes (path, headers, host). Can do TLS termination, request routing, retries. AWS ALB, nginx, Envoy.

Choose L7 when you need HTTP-aware features. Choose L4 for raw TCP performance.

Load balancing algorithms:
- Round-robin.
- Least connections.
- Consistent hashing (for session affinity or cache locality).
- Weighted (for canary traffic shifting).

## 17.11 Network Debugging Tools

Master these:
- `ping` — basic reachability.
- `traceroute` / `mtr` — path between hops.
- `dig` — DNS.
- `curl -v` — full HTTP request lifecycle.
- `netstat -tlnp` / `ss -tlnp` — listening ports.
- `tcpdump` — packet capture.
- `iptables -L` / `nft list ruleset` — firewall rules.
- `nc` (netcat) — quick TCP/UDP testing.
- `iperf3` — bandwidth measurement.

Knowing which tool to reach for instantly is the mark of an experienced SRE.

## 17.12 Systemd Basics

Systemd is the init system on modern Linux. SREs interact with it via:
- `systemctl status / start / stop / restart` — service management.
- `journalctl -u <service>` — service logs.
- Unit files in `/etc/systemd/system/` — service definitions.

Many production debugging sessions start with `systemctl status` and `journalctl`.

## 17.13 cgroups and Namespaces

The primitives that enable containers:

**Namespaces** isolate what a process can see (PIDs, mounts, network, users).
**cgroups** limit what a process can use (CPU, memory, I/O).

Docker, Kubernetes, and other container runtimes are user-space wrappers over these kernel features.

SRE relevance: when a container is OOM-killed, the cgroup hit its limit. When a container is CPU-throttled, the cgroup CPU quota is exhausted. Knowing this connects symptoms to the underlying mechanism.

## 17.14 File Descriptors

Linux assigns file descriptors (FDs) to every open file, socket, and pipe. Limits are per-process and per-system.

When a process runs out of FDs:
- `accept()` fails.
- File opens fail.
- Logs may stop writing.

Default ulimit is usually too low for production servers. Raise to 65535+ for serious services.

Tool: `lsof -p <pid>` shows all FDs for a process.

## 17.15 Production Architecture (Linux + Network)

```
   Client
     |
   DNS resolution
     |
   TCP handshake to load balancer
     |
   TLS termination (if HTTPS)
     |
   Load balancer routes to backend
     |
   Backend process (Linux node)
        |
        +-- cgroup limits CPU/memory
        +-- namespace isolation
        +-- filesystem in mount namespace
        +-- network in network namespace
        |
   Process accepts connection, reads, writes
     |
   Response back through the chain
```

## 17.16 Tradeoffs

| Choice | Win | Cost |
|---|---|---|
| L4 LB | Fast, simple | Less feature-rich |
| L7 LB | HTTP-aware | More compute |
| HTTP/1.1 | Universal | Slower |
| HTTP/2 | Multiplexed | Implementation complexity |
| TLS everywhere | Secure | CPU overhead |
| Long TTLs | Less DNS load | Stale records |
| Short TTLs | Fast changes | More DNS load |

## 17.17 Scaling Challenges

- Ephemeral port exhaustion on outbound-heavy hosts.
- TIME_WAIT accumulation.
- DNS as a single point of failure.
- Certificate expiry storms (many certs expire same week).
- File descriptor limits at high concurrency.

## 17.18 Security

- Firewall (iptables, nftables, Security Groups).
- TLS for all internal traffic where feasible.
- mTLS for service-to-service in zero-trust.
- DNSSEC where supported.
- Rate limiting at the load balancer.

## 17.19 Deployment Guide

A production Linux node baseline:
- Modern kernel.
- ulimit -n 65535+.
- TCP tuning for high concurrency (somaxconn, tcp_tw_reuse).
- Systemd-managed services with proper restart policies.
- Log rotation configured.
- Time sync (NTP / chrony).
- Monitoring agent installed (node_exporter or equivalent).

## 17.20 Monitoring Strategy

Use USE method for nodes:
- CPU utilization and saturation.
- Memory utilization (RSS, page cache) and saturation (swap, OOM).
- Disk utilization (space, I/O) and saturation.
- Network utilization (bandwidth, packet rate) and errors.

Plus:
- Load average.
- File descriptor count.
- Process count.

## 17.21 Cost Optimization

- Right-size nodes (most are over-provisioned).
- Use spot instances where workloads tolerate.
- Newer instance generations are usually better price/performance.

## 17.22 Interview Questions

- *Explain load average.*
- *Walk through what happens when you type a URL.*
- *Difference between L4 and L7 load balancing.*
- *How does TLS handshake work?*
- *Debug a slow service: what tools do you reach for first?*

## 17.23 Hands-on Exercises

1. On a Linux box, run `top`, `vmstat 1`, `iostat 1`. Read what they tell you about the system.
2. Use `tcpdump` to capture a curl to a web service. Walk through the packets.
3. Use `dig` to look up a domain's A, AAAA, MX, NS records.
4. Simulate file descriptor exhaustion in a test environment. Observe the failure mode.

## 17.24 Common Mistakes

- Confusing load average with CPU percent.
- Thinking "free" memory is what's actually available.
- Ignoring TIME_WAIT accumulation.
- Long DNS TTLs that prevent quick changes.
- Default ulimit in production.
- Not monitoring certificate expiry.

## 17.25 Enterprise Best Practices

Linux distribution standardized (Ubuntu LTS or RHEL). Image hardening baseline. Automated patching. Cert-manager for TLS. DNS managed by a tool (Route53, Cloudflare). Network policies enforced in Kubernetes. Logs from journalctl shipped centrally. SRE training includes Linux fundamentals.
