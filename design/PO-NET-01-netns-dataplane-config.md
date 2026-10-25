# PO-NET-01: Network Namespace Dataplane Configuration

**Status:** ADOPTED — Option B EXECUTED (2026-07-24); D1-ISO RESOLVED (claim narrowed), feature gap re-filed as D3 NETNS-DATAPLANE-CONFIG (Phase I.3)  
**Finding:** D1-ISO-NETNS-DATAPLANE  
**Executable Implementation:** TX device-ownership gate (landed 2026-07-20) + `tx_auth::AuthorizedTxDevice` type-enforced capability, `net_ns_tx_isolation` boot test, `docs/namespace-isolation.md` (landed 2026-07-24, `kernel/net/src/stack.rs`)  
**Purpose:** Define the architecture for per-namespace dataplane configuration (IP/MAC/ARP isolation) to complete container network isolation.

---

## 1. Executive Summary

This document specifies the **per-namespace dataplane configuration** required to complete network namespace isolation in Zero-OS. The current state (as of 2026-07-20) provides:

✅ **RX isolation** - Packets delivered to correct namespace based on destination IP/port  
✅ **TX fail-closed gate** - Child namespaces cannot transmit on root's device without ownership  
❌ **Per-namespace config** - Child namespaces have no isolated IP/MAC/ARP configuration

The TX gate (landed 2026-07-20) prevents **namespace escape** (child transmitting with root's identity), which is the critical security boundary. However, it creates a **fail-closed posture** where child namespaces have no transmit capability at all - they get `EPERM` on every TX attempt because they own zero devices and have no independent network configuration.

**The core architectural question:**

> Should Zero-OS provide full per-namespace dataplane isolation (like Linux netns), or should it narrow the isolation claim to "fail-closed TX gate prevents escape, but child namespaces are not network-capable"?

This document explores both paths and recommends **Option B: Narrowed Claim** for 1.0-Preview, with **Option A: Full Isolation** deferred to Phase I.3+ as a container-networking feature arc.

---

## 2. Current State Analysis

### 2.1 What Works Today

**RX Path (fully namespace-aware since initial implementation):**
- Incoming packets examined by socket layer (`net/socket.rs`)
- Destination determined by IP:port tuple
- Delivered to socket in correct namespace
- **Result:** RX isolation is complete and correct

**TX Gate (landed 2026-07-20; TYPE-ENFORCED 2026-07-24):**
- Single choke-point: `resolve_authorized_tx_device` (`stack.rs`, `tx_auth` module)
- Both physical TX sinks use it (`build_frame_and_transmit`, `transmit_prepared_reply`)
- Device ownership checked via `netns_owns_device` upcall (`socket.rs:477`)
- Unowned device → `TxError::FirewallDenied` → `EPERM`
- 2026-07-24: the gate is a **type-enforced capability** — the sole path to a driver
  `transmit` is the single-use, non-Clone `tx_auth::AuthorizedTxDevice` token; raw
  transmit-capable handles no longer egress the `net` crate (out-of-crate bypass
  impossible by construction; the in-crate raw lookup is a one-caller audited TCB)
- **Result:** Namespace TX escape prevented (child cannot use root's device)

**Loopback (namespace-aware):**
- 127.0.0.1 delivery is socket-layer only, never enters physical TX
- Each namespace has independent loopback address space
- **Result:** Loopback isolation is complete and correct

### 2.2 What's Missing

**Per-Namespace Network Configuration:**

Child namespaces created via `CLONE_NEWNET` have:
- ❌ No IP address assignment
- ❌ No MAC address
- ❌ No device ownership (intentionally starts with 0 devices)
- ❌ No ARP table
- ❌ No routing table
- ✅ Independent firewall rules (R121-1: per-ns `FirewallTable`, default-deny — doc drift
  fixed 2026-07-24; conntrack, budgets, and port charges are also per-ns)

**Consequence:** A child namespace can:
1. Create sockets ✅
2. Bind to 127.0.0.1 (loopback) ✅
3. Communicate with other sockets in the same namespace via loopback ✅
4. **Cannot** transmit to external network (no device, `EPERM`) ❌
5. **Cannot** receive from external network (no listening sockets with routable IPs) ❌

This is **fail-closed** (secure) but **not useful** for container networking.

### 2.3 Linux netns Reference Model

For comparison, Linux network namespaces provide:

1. **Independent device list** - Each netns sees different devices via `ip link set DEV netns NS`
2. **Per-namespace IP config** - `ip addr add` / `ip route add` per namespace
3. **Per-namespace ARP cache** - Isolated neighbor tables
4. **Per-namespace firewall** - Independent iptables/nftables rulesets
5. **Virtual interfaces** - veth pairs, bridges, TAP devices for inter-namespace communication

Zero-OS currently provides #1 (device ownership model exists, but no API to assign devices to child namespaces) and lacks #2-5 entirely.

---

## 3. Architecture Options

### Option A: Full Per-Namespace Dataplane Isolation (Linux-Compatible)

**Goal:** Make Zero-OS network namespaces functionally equivalent to Linux netns for container workloads.

**Required Components:**

| Component | Description | Estimated Effort |
|-----------|-------------|------------------|
| **Device assignment API** | `sys_netns_attach_device(ns_id, device_name)` syscall to transfer device ownership | 1-2 days |
| **Per-namespace IP config** | IP address, netmask, gateway per namespace × device | 2-3 days |
| **Per-namespace MAC config** | Allow child namespaces to set device MAC (or virtualize it) | 1-2 days |
| **Per-namespace ARP cache** | Isolated ARP table per namespace | 3-4 days |
| **Per-namespace routing** | Routing table per namespace (currently global) | 3-5 days |
| **Virtual device support** | veth pairs or equivalent for namespace interconnection | 5-7 days |
| **Per-namespace firewall** | Isolated firewall rules per namespace | 2-3 days |

**Total effort:** ~20-30 days of focused development + testing

**Heap impact:** Per-namespace config structures (IP/MAC/ARP/routes) → estimate 4-8 KiB per namespace × 64 max namespaces = 256-512 KiB additional heap footprint. Requires dedicated `HeapClass::NetnsConfig` admission.

**Benefits:**
- ✅ Full container networking capability (comparable to Linux)
- ✅ Enable Docker/Kubernetes-style multi-container networking
- ✅ Virtualization/cloud workload support

**Risks:**
- ⚠️ Large feature scope (20-30 days)
- ⚠️ Significant heap footprint increase (requires PO-RES-01/02 mitigation first)
- ⚠️ ABI surface expansion (new syscalls, config structures)
- ⚠️ Complexity in TX path (per-namespace MAC/IP resolution)

**Timeline:** Phase I.3 (container networking arc) or later

---

### Option B: Narrowed Isolation Claim (Fail-Closed Minimal)

**Goal:** Document the current fail-closed TX gate as the isolation boundary and explicitly narrow the namespace isolation claim to "prevents escape, but child namespaces are not independently network-capable."

**Required Components:**

| Component | Description | Estimated Effort |
|-----------|-------------|------------------|
| **Documentation** | Update namespace documentation to state: "Child namespaces are isolated containers with loopback-only networking. Physical device access requires explicit device assignment (not yet implemented)." | 1 day |
| **Claim narrowing** | Update security model to clarify: "Network namespace isolation prevents TX escape (enforced by `resolve_authorized_tx_device`), but does not provide independent network configuration. Use cases requiring per-container networking should defer to Phase I.3." | 1 day |
| **Test coverage** | Add explicit test: "child namespace TX to external network returns `EPERM`" | 0.5 days |

**Total effort:** ~2-3 days documentation + testing

**Heap impact:** Zero (no new features)

**Benefits:**
- ✅ Honest documentation of current capabilities
- ✅ No feature scope creep before 1.0-Preview
- ✅ Security boundary (TX gate) is correct and enforced
- ✅ Allows 1.0-Preview release without container networking

**Risks:**
- ⚠️ Users expecting Linux-compatible netns will be disappointed
- ⚠️ Container orchestration (Docker/K8s) not supported
- ⚠️ D1-ISO finding remains D1 until Option A is implemented OR claim is formally narrowed and accepted

**Timeline:** Immediate (documentation-only, can land pre-1.0-Preview)

---

## 4. Recommended Path: Option B for 1.0-Preview, Option A Deferred

### 4.1 Rationale

**For 1.0-Preview (Option B):**

1. **Scope control** - 1.0-Preview focuses on kernel correctness, not container orchestration
2. **Security posture** - TX gate is correct (prevents escape); lack of per-ns config is a feature gap, not a vulnerability
3. **Heap constraints** - PO-RES-01 identified significant heap pressure; adding 256-512 KiB for netns config without resolving boot-path/syscall-temporary gaps would worsen D1-RES
4. **Time to release** - Option A is 20-30 days; Option B is 2-3 days
5. **User expectations** - 1.0-Preview users are kernel developers/testers, not container workload runners

**For Phase I.3+ (Option A):**

1. **Container networking demand** - If Zero-OS targets container orchestration, full netns isolation becomes mandatory
2. **Heap mitigation completed** - After PO-RES-01/02 mitigations land (boot-path admission, syscall-temporary admission), 256-512 KiB netns config becomes feasible
3. **Design maturity** - ARP/routing/virtual-device architecture needs careful design; rushing it for 1.0-Preview risks regressions

### 4.2 Implementation Plan (Option B)

> **EXECUTED 2026-07-24 (D1ISO slice):** Step 1 landed as `docs/namespace-isolation.md`
> (updated for the type-enforced `tx_auth::AuthorizedTxDevice` token, which supersedes the
> convention-only gate this draft assumed). Step 2 landed as the `net_ns_tx_isolation` boot
> test (`kernel/src/runtime_tests.rs` — both-sink deny, A/B/A ownership toggle, root-ns
> preservation, stale-ns fail-closure; the sketch below predates it) plus the resolver-level
> host test `d1iso_resolver_gates_tx_by_namespace_ownership` (`kernel/net/src/stack.rs`).
> Step 3 executed: D1-ISO → RESOLVED; feature gap re-filed as **D3 NETNS-DATAPLANE-CONFIG**
> (Phase I.3, §4.3 is its implementation order).

**Step 1: Documentation (docs/namespace-isolation.md)**

Create formal documentation stating:

```markdown
# Network Namespace Isolation in Zero-OS

## Security Boundary

Zero-OS enforces **TX device-ownership isolation** via the `resolve_authorized_tx_device` 
gate (`kernel/net/src/stack.rs:1370`). Child namespaces created with `CLONE_NEWNET`:

- ✅ Cannot transmit on devices they do not own (enforced: `EPERM`)
- ✅ Can create sockets and bind to loopback (127.0.0.1)
- ✅ Can communicate with other processes in the same namespace via loopback
- ❌ Cannot transmit to or receive from external networks (no device ownership, no IP config)

This prevents **namespace escape** (child using root's network identity) but does not
provide **independent network configuration** (per-namespace IP/MAC/ARP).

## Current Capabilities

Network namespaces are suitable for:
- Process isolation with loopback-only communication
- Security sandboxing (prevent network access)
- Testing namespace-aware code

Network namespaces are **not suitable** for:
- Container networking (Docker/Kubernetes-style multi-container)
- Per-container external network access
- Virtual networking (veth, bridges)

## Future Work

Full per-namespace dataplane configuration (IP/MAC/ARP/routing per namespace) is 
planned for **Phase I.3 (container networking arc)**. This will enable:
- Device assignment API (`sys_netns_attach_device`)
- Per-namespace IP/MAC configuration
- Virtual device support (veth pairs)
- Per-namespace ARP cache and routing tables

Until then, child namespaces are **fail-closed** (cannot access external network).
```

**Step 2: Test Coverage**

Add test to `kernel/src/runtime_tests.rs`:

```rust
/// D1-ISO: Child namespace TX to external network returns EPERM (fail-closed).
#[test]
fn test_netns_child_tx_blocked() {
    let child_ns = NetNamespace::create().unwrap();
    let socket = Socket::new(SocketDomain::Inet, SocketType::Stream).unwrap();
    socket.set_namespace(child_ns.id());
    
    // Child namespace owns 0 devices, so any TX to external IP should fail
    let result = socket.connect(Ipv4Addr::new(8, 8, 8, 8), 80);
    assert_eq!(result, Err(Errno::EPERM)); // TX gate blocks
}
```

**Step 3: Update D1-ISO Finding Status**

Once documentation lands, update D1-ISO in the plan:

- **Status:** RESOLVED (claim narrowed to "fail-closed TX gate prevents escape")
- **Severity:** Downgrade from D1 to **RESOLVED** (security boundary correct) or **D3** (feature gap documented)
- **Rationale:** TX gate prevents the security-critical escape; lack of per-ns config is a **capability limitation**, not an **isolation vulnerability**

### 4.3 Implementation Plan (Option A - Deferred)

If Option A is pursued in Phase I.3+, the implementation order should be:

**Phase 1: Device Assignment (enable child TX)**
1. Add `sys_netns_attach_device(ns_id, device_name)` syscall
2. Update `netns_owns_device` to check assigned device list
3. Test: child namespace can TX after device assignment

**Phase 2: Per-Namespace Config (enable external RX)**
4. Add per-namespace IP/MAC/gateway config structures (`NetnsConfig`)
5. Update TX path to use namespace-local IP/MAC (not global `STACK_CONFIG`)
6. Update ARP resolver to query namespace-local cache
7. Test: child namespace can communicate externally with assigned IP

**Phase 3: Routing & Firewall**
8. Per-namespace routing table
9. Per-namespace firewall rules
10. Test: multi-namespace routing works

**Phase 4: Virtual Devices**
11. veth pair implementation
12. Inter-namespace communication without physical device
13. Test: container-to-container networking

**Heap admission:** Add `HeapClass::NetnsConfig` with 512 KiB ceiling, integrate with PO-RES-01 mitigations.

---

## 5. D1-ISO Severity Assessment

### 5.1 Current Severity: D1 (Systemic Critical)

D1-ISO was rated **D1** because the initial assessment assumed:
- Child namespaces **should** have independent network capability (Linux netns parity)
- Lack of per-ns config is a **missing security enforcement** (isolation incomplete)

### 5.2 Severity After Option B (Claim Narrowing)

If Option B is adopted (narrowed claim), the severity reassessment:

**Option 1: Downgrade to D3 (Design Debt)**
- **Rationale:** Security boundary (TX gate) is correct; lack of per-ns config is a **feature gap**, not a vulnerability
- **Blast radius:** `local` (affects only users expecting full netns capability)
- **Chainability:** `requires-many` (need other missing features to exploit)
- **Path to closure:** Document limitation, defer full implementation to Phase I.3

**Option 2: Resolve as WONTFIX (Claim Narrowing Accepted)**
- **Rationale:** Zero-OS explicitly does not claim Linux netns parity for 1.0-Preview; fail-closed TX gate is sufficient isolation for the claimed threat model
- **Acceptance criteria:** Maintainers agree that container networking is out-of-scope for 1.0-Preview

**Option 3: Keep D1 Until Option A Implemented**
- **Rationale:** Container networking is a core capability expectation; lack of it is a systemic gap
- **Consequence:** 1.0-Preview gate blocked until Option A lands (20-30 days)

**Recommended:** **Option 1 (D3 after claim narrowing)** or **Option 2 (RESOLVED)**, with Option A deferred to Phase I.3.

### 5.3 Threat Model Consideration

The key question: **What attacker scenario does per-ns config defend against?**

**Scenario 1: Namespace Escape (TX with root identity)**
- **Threat:** Child namespace transmits packets with root's IP/MAC, impersonating root
- **Mitigation:** TX gate (`resolve_authorized_tx_device`) - **IMPLEMENTED ✅**
- **Severity if missing:** D1 (critical isolation breach)

**Scenario 2: Denial of Service (no network for containers)**
- **Threat:** Child namespaces cannot communicate externally, breaking container workloads
- **Mitigation:** Per-ns config (Option A) - **NOT IMPLEMENTED ❌**
- **Severity if missing:** D3 or INFO (feature gap, not vulnerability)

**Scenario 3: Information Disclosure (ARP cache sharing)**
- **Threat:** Child namespace can observe root's ARP cache, learning about other hosts
- **Mitigation:** Per-ns ARP cache (Option A) - **NOT IMPLEMENTED ❌**
- **Severity if missing:** D4 or LOW (information leak, low impact)

**Assessment:** The **security-critical** threat (Scenario 1) is mitigated. The remaining gaps (Scenarios 2-3) are **capability/information** issues, not **isolation failures**.

---

## 6. Conclusion

### 6.1 Current State Summary

✅ **TX device-ownership gate landed** (2026-07-20) - prevents namespace escape  
✅ **RX isolation complete** - packets delivered to correct namespace  
✅ **Loopback isolation complete** - per-namespace 127.0.0.1  
❌ **Per-namespace config missing** - child namespaces cannot access external network  

### 6.2 Why D1-ISO Remains D1

D1-ISO remains **D1 severity** because:

1. **Expectation mismatch** - "Network namespace" implies Linux netns parity, but Zero-OS provides only fail-closed TX gate
2. **Capability gap** - Child namespaces are not useful for container networking (the expected use case)
3. **Architectural incompleteness** - Lack of per-ns config affects the isolation model's claimed completeness

These are **not security vulnerabilities** (TX gate prevents escape), but they are **architectural gaps** that affect the feature's utility.

### 6.3 Recommended Path to Closure

**Immediate (Pre-1.0-Preview):**
- **Adopt Option B** (narrowed claim) via documentation
- **Downgrade D1-ISO to D3 or RESOLVED** (security boundary correct, feature gap documented)
- **Defer Option A to Phase I.3+** (container networking arc)

**Phase I.3+ (Post-1.0-Preview):**
- **Implement Option A** (full per-ns dataplane isolation) if container networking becomes a roadmap priority
- **Heap mitigation prerequisite** - Resolve PO-RES-01/02 gaps before adding 256-512 KiB netns config footprint

**Acceptance Criteria for Closure:**
- If Option B adopted: Documentation lands + D1-ISO downgraded to D3/RESOLVED
- If Option A pursued: All Phase 1-4 components implemented + heap admission integrated + tests pass

---

## 7. References

- **TX gate implementation:** `kernel/net/src/stack.rs:1370` (`resolve_authorized_tx_device`)
- **Ownership upcall:** `kernel/kernel_core/net_namespace.rs` (`net_ns_owns_device`)
- **Design finding:** D1-ISO-NETNS-DATAPLANE (R180 discovery, 2026-07-08)
- **TX gate convergence:** MODE C Codex session `019f7ec0-494f-7561-9abd-f834001adea8` (2 iterations, SAFE verdict, 2026-07-20)
- **Test coverage:** `kernel/src/runtime_tests.rs::net_ns_isolation` (Test 11: device ownership predicates)

---

**Document Status:** DRAFT — requires maintainer decision on Option A vs Option B before D1-ISO status change.

**Next Steps:**
1. Maintainer review: Accept Option B (narrow claim) or commit to Option A (full isolation)?
2. If Option B: Land documentation, add test, downgrade D1-ISO to D3/RESOLVED
3. If Option A: Schedule Phase I.3 container networking arc, prioritize after PO-RES-01/02 heap mitigations

*Generated 2026-07-22 as part of design queue resolution following zero-HIGH streak 3/3 completion.*
