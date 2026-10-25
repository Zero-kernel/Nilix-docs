# PO-NET-01: Network Namespace Dataplane Isolation

**Status:** Claim Narrowing + Roadmap (2026-07-22)  
**Finding:** D1-ISO-NETNS-DATAPLANE  
**Executable Implementation:** TX-gate (fail-closed device ownership) — COMPLETE 2026-07-20  
**This Document:** Security posture + claim scope + path to full isolation  

---

## Executive Summary

**Current security posture (as of 2026-07-20):**
- ✅ **TX escape prevention:** CLONE_NEWNET children cannot transmit on parent devices (fail-closed EPERM via `resolve_authorized_tx_device`)
- ✅ **RX isolation:** Fully netns-aware (packets delivered to correct namespace sockets)
- ❌ **Per-ns config:** IP/MAC/ARP configuration is **global** (ns-0 only) — child namespaces have no isolated dataplane config

**Isolation claim (narrowed):**
> Zero-OS network namespaces provide **TX escape prevention** and **RX isolation** for container workloads. Child namespaces cannot egress on unauthorized devices (fail-closed). **Full container network isolation** (per-ns IP/MAC/ARP config, isolated routing tables, per-ns veth pairs) is **deferred** to Phase I.3+.

**D1 severity rationale:**  
The architectural requirement for container-native workload claims is **full network isolation** (isolated configs + routing). The current TX gate eliminates the acute security vulnerability (TX escapes) but does not provide the feature completeness required for "container network isolation" claims. D1 persists until either:
1. **Full per-ns config arc** is implemented (Phase I.3+), OR
2. **Container isolation claims** are explicitly scoped to exclude network isolation in external documentation

**This document:** Provides the security contract, documents residual scope gaps, and defines the path to full isolation.

---

## 1. Current Implementation (TX-Gate Leg)

### 1.1 Threat Model

**Prevented attack:**
A CLONE_NEWNET child process (which should be isolated) transmits packets on the root namespace's physical device (eth0) using the root namespace's IP/MAC, bypassing network isolation and potentially:
- Spoofing the host's identity
- Accessing networks the child should not reach
- Violating firewall/routing policies

**Implementation (2026-07-20):**
```rust
// kernel/net/src/stack.rs
fn resolve_authorized_tx_device(net_ns_id: u32, target_device_handle: u32) 
    -> Result<(Device, usize), TxError>
```

**Contract:**
1. **Single choke-point:** ALL physical TX paths call this resolver (verified: `build_frame_and_transmit`, `transmit_prepared_reply`)
2. **Authorization check:** `kernel_core::net_ns_owns_device(net_ns_id, device_handle)` — ns-0 owns all; child owns only `devices` set
3. **Fail-closed:** Non-owner → `FirewallDenied` (EPERM to userspace)
4. **Unregistered hook:** Admits only ns-0 (hook registration in `kernel_core::init` precedes userspace)

**Coverage:**
- ✅ Physical TX (eth0, any NIC driver)
- ✅ Root namespace (ns-0) unrestricted
- ✅ Child namespace (ns-1+) restricted to assigned devices (empty set by default → all TX denied)
- ✅ Loopback unaffected (127.0.0.1 delivery is socket-layer, never enters physical TX)
- ✅ Stale namespace (destroyed ns-id) → owns nothing (Weak registry cleanup)

**Test coverage:**
- `test_net_ns_isolation` (Test 11): root/child/add-device/remove-device/destroyed-ns predicate matrix
- Host unit test: unregistered hook arm verification

**Convergence:** MODE C, 2 iterations, SAFE (Codex SESSION_ID `019f7ec0-494f-7561-9abd-f834001adea8`)

### 1.2 Security Proof (TX Escape Prevention)

**Claim:** A CLONE_NEWNET child cannot transmit on devices it does not own.

**Proof obligations:**

| Obligation | Status | Evidence |
|------------|--------|----------|
| **O1:** All physical TX paths call the resolver | ✅ VERIFIED | Grep audit: 2 sinks (`build_frame_and_transmit`, `transmit_prepared_reply`); both call `resolve_authorized_tx_device` |
| **O2:** Resolver enforces ownership | ✅ VERIFIED | `net_ns_owns_device` implementation: ns-0 owns all; child owns `devices` set (empty by default) |
| **O3:** Non-owner → denial | ✅ VERIFIED | `!owns` → `Err(TxError::FirewallDenied)` → EPERM to userspace |
| **O4:** Device assignment is exclusive | ✅ VERIFIED | `move_device` transfers ownership (removed from source, added to dest); no shared ownership |
| **O5:** Destroyed namespace owns nothing | ✅ VERIFIED | `NET_NS_BY_ID` is Weak registry; stale id lookup fails → owns nothing |
| **O6:** Loopback unaffected | ✅ VERIFIED | 127.0.0.1 delivery is `deliver_to_local_sockets` (socket-layer), never enters `stack.rs` TX paths |

**Residual risks (bounded):**
- **R1-FUTURE-TX-SITES:** Convention (resolver call) is not type-enforced. A FUTURE physical TX path could bypass the resolver. **Mitigation:** `AuthorizedTxDevice` capability type (Phase I roadmap) makes bypass structurally impossible.
- **R2-DEVICE-MOVE-TOCTOU:** `move_device` checks ownership at call time but device could be moved during the operation. **Disposition:** Documented as check-time linearization; drain-before-return NOT claimed (acceptable for admin-only device moves).

**Conclusion:** TX escape prevention is **complete and sound** under the documented contract. Child namespaces fail closed (EPERM) when attempting unauthorized TX.

---

## 2. Residual Gap: Per-Namespace Dataplane Config

### 2.1 Current Limitation

**Global config:** IP addresses, MAC addresses, ARP tables, and routing tables are **namespace-0 only**. Child namespaces inherit the root namespace's config but cannot have isolated configs.

**Impact on container isolation:**

| Feature | Current Status | Required for Full Isolation |
|---------|---------------|----------------------------|
| TX escape prevention | ✅ COMPLETE | ✅ (delivered 2026-07-20) |
| RX isolation | ✅ COMPLETE | ✅ (pre-existing) |
| Per-ns IP/MAC config | ❌ MISSING | ✅ (child needs own IP/MAC) |
| Per-ns ARP table | ❌ MISSING | ✅ (child needs isolated ARP) |
| Per-ns routing table | ❌ MISSING | ✅ (child needs own routes) |
| veth pairs (inter-ns pipes) | ❌ MISSING | ✅ (ns-to-ns connectivity) |
| Bridge devices | ❌ MISSING | Optional (container orchestration) |

**Example limitation:**  
A container workload (CLONE_NEWNET child) with a veth device assigned to it:
- ✅ CAN receive packets addressed to the veth's IP (RX isolation works)
- ✅ CANNOT transmit on root's eth0 (TX gate works)
- ❌ CANNOT configure the veth's IP/MAC (no per-ns config API)
- ❌ CANNOT have its own routing table (global routes only)

**Workaround:** External orchestration (e.g., host-side `ip netns` + veth setup) can configure devices before assigning them to a namespace, but Zero-OS kernel does not expose per-ns config syscalls.

### 2.2 Named Residuals (Scope Gaps)

| ID | Gap | Impact | Phase |
|----|-----|--------|-------|
| **RN1-CONFIG-API** | No per-ns IP/MAC/ARP config syscalls | Child cannot self-configure network | I.3 |
| **RN2-ROUTING** | No per-ns routing table | Child cannot define own routes | I.4 |
| **RN3-VETH** | No veth device type (inter-ns pipe) | No ns-to-ns connectivity | I.3 |
| **RN4-BRIDGE** | No bridge device type | No multi-container L2 domain | I.5+ |
| **RN5-AUTHDEV-CAP** | Convention-based resolver call (not type-enforced) | Future TX sites could bypass | I.3 |
| **RN6-TX-PERF** | BTreeMap read-lock per child TX packet | Ns-0 fast path exempt; child TX has registry overhead | I.4 |

**Bounded-impact analysis:**
- **RN1-RN4:** Feature gaps, not security vulnerabilities. Child namespaces fail closed (EPERM on unauthorized TX) — the secure default.
- **RN5:** Convention risk (future code could bypass resolver). **Mitigation priority:** HIGH (Phase I.3). Introduce `AuthorizedTxDevice` capability type returned by resolver; TX sinks require it.
- **RN6:** Performance, not correctness. Ns-0 (host) unaffected; container TX has lock overhead. Generation-token cache candidate (Phase I.4).

---

## 3. Path to Full Isolation (Phase I Roadmap)

### 3.1 Phase I.3: Per-Namespace Config + veth

**Deliverables:**
1. **Per-ns config storage:**
   - `NetworkNamespace` gains `ip_addrs: BTreeSet<IpAddr>`, `mac_addrs: BTreeMap<DeviceHandle, MacAddr>`, `arp_table: BTreeMap<IpAddr, MacAddr>`
   - Config APIs: `SYS_netns_add_ip`, `SYS_netns_set_mac`, `SYS_arp_add` (scoped to calling ns)

2. **TX dataplane uses per-ns config:**
   - `resolve_src_ip(net_ns_id, device) -> Option<IpAddr>` — lookup in ns-specific config
   - `resolve_src_mac(net_ns_id, device) -> Option<MacAddr>`
   - Fallback: If ns has no config for device → EPERM (fail-closed)

3. **veth device type:**
   - `DeviceKind::Veth { peer_ns: u32, peer_handle: u32 }`
   - TX on veth → enqueue to peer's RX queue (inter-ns pipe)
   - Assignment: `SYS_netns_move_device` supports veth; both ends tracked

4. **`AuthorizedTxDevice` capability type (RN5 mitigation):**
   ```rust
   pub struct AuthorizedTxDevice {
       device: Device,
       stable_index: usize,
       _phantom: PhantomData<*const ()>, // !Send, !Sync — must consume in same call
   }
   ```
   - Resolver returns `AuthorizedTxDevice`; TX sinks require it
   - Future TX sites cannot bypass (type system enforces)

**Acceptance criteria:**
- Container can configure veth IP/MAC via syscalls
- Container can ARP/ping another container via veth pair
- Container TX on unauthorized device → EPERM (unchanged)
- Ns-0 config unaffected (backward compat)

### 3.2 Phase I.4: Per-Namespace Routing + TX Perf

**Deliverables:**
1. **Per-ns routing table:**
   - `routing_table: BTreeMap<IpPrefix, Route>` in `NetworkNamespace`
   - `SYS_route_add`, `SYS_route_del` (scoped to calling ns)
   - Longest-prefix-match for egress device selection

2. **TX performance (RN6 mitigation):**
   - Generation-token cache: `(net_ns_id, gen) -> DeviceSet`
   - On device move/destroy, increment `gen` (invalidates cache)
   - Child TX: check cache → fast path; miss → BTreeMap lookup + cache update

**Acceptance criteria:**
- Container can define routes (e.g., `10.0.0.0/24 via veth0`)
- TX dataplane selects device per ns-specific routes
- Child TX performance: <10% overhead vs ns-0 (cached path)

### 3.3 Phase I.5+: Bridge Devices (Optional)

**Scope:** L2 bridge for multi-container domains (e.g., Docker bridge network).  
**Complexity:** Medium (MAC learning, STP optional).  
**Priority:** LOW (veth pairs sufficient for most container use cases).

---

## 4. Claim Narrowing (Current Documentation)

### 4.1 Scoped Isolation Claim

**✅ Accurate claim (as of 2026-07-20):**
> Zero-OS network namespaces provide **TX escape prevention** and **RX packet isolation**. A CLONE_NEWNET child cannot transmit on devices it does not own (fail-closed EPERM). Packets are delivered to the correct namespace's sockets (RX isolation). Per-namespace IP/MAC/ARP configuration and routing tables are **not yet supported** (Phase I.3+ roadmap).

**❌ Inaccurate claims to avoid:**
- "Full container network isolation" (implies per-ns config, which is missing)
- "Docker/Kubernetes-compatible networking" (requires veth + bridge, which are deferred)
- "Production-ready container runtime" (network isolation is ONE of many container requirements)

### 4.2 External Documentation Updates

**If container isolation is mentioned in:**
- `README.md` (project overview)
- `docs/ARCHITECTURE.md` (system capabilities)
- Any blog posts / release notes

**Ensure the network isolation section states:**
1. TX escape prevention (✅ complete)
2. RX isolation (✅ complete)
3. Per-ns config (❌ deferred to Phase I.3+)
4. Link to this PO-NET-01 document for technical details

**Recommendation:** Defer "container isolation" claims entirely until Phase I.3 delivers per-ns config. Current capability is "network namespace primitive with TX escape prevention" — accurate but narrow.

---

## 5. D1 Severity Persistence

**Why D1 persists despite TX-gate completion:**

The D1 finding (`D1-ISO-NETNS-DATAPLANE`) was raised against the **architectural requirement** for container network isolation, not just the TX escape vulnerability. The threat model includes:

1. ✅ **Acute vulnerability (TX escape):** RESOLVED via TX-gate (2026-07-20)
2. ❌ **Feature completeness (isolated configs):** OPEN (Phase I.3+ roadmap)

**D1 severity is justified because:**
- Container isolation is a **stated project goal** (implied by CLONE_NEWNET support)
- Missing per-ns config prevents **production container workloads** (Docker/Kubernetes-style isolation)
- The gap is **architectural** (not a single-function bug) — requires design + multi-subsystem changes

**Path to D1 closure:**
- **Option A:** Implement Phase I.3 (per-ns config + veth) → full isolation → D1 RESOLVED
- **Option B:** Narrow container isolation claims in external docs → D1 DOWNGRADED to D2 (acknowledged limitation, not a blocker for current scope)

**Current stance (2026-07-22):** D1 persists. TX-gate provides the **security backstop** (fail-closed), which is the correct default. Per-ns config is the **feature arc** required for full isolation. Both are necessary; neither alone closes the finding.

---

## 6. Verification Strategy (Future Phases)

### 6.1 Phase I.3 Acceptance Tests

**Test coverage required before closing D1:**

| Test | Scenario | Pass Criteria |
|------|----------|---------------|
| **T1-NS-CONFIG** | Child configures veth IP/MAC via syscall | Config visible in ns, isolated from parent |
| **T2-VETH-PING** | Child pings another child via veth pair | ICMP echo/reply via inter-ns pipe |
| **T3-VETH-TX** | Child sends TCP via veth to peer ns | Packets delivered to peer ns sockets |
| **T4-UNAUTHORIZED-STILL-EPERM** | Child TX on root eth0 | EPERM (TX-gate unchanged) |
| **T5-ARP-ISOLATION** | Child ARP table independent of root | Separate ARP cache per ns |
| **T6-AUTHDEV-CAP** | Add new TX sink without `AuthorizedTxDevice` | Compile error (type enforcement) |

### 6.2 Phase I.4 Performance Validation

**Benchmark:** Child TX throughput vs ns-0 (1000-packet burst, veth loopback).  
**Target:** <10% overhead (cached path).  
**Fallback:** If generation-token cache insufficient, consider per-ns device array (O(1) lookup).

---

## 7. Summary

| Aspect | Status |
|--------|--------|
| **TX escape prevention** | ✅ COMPLETE (fail-closed TX-gate, 2026-07-20) |
| **RX isolation** | ✅ COMPLETE (pre-existing) |
| **Per-ns config (IP/MAC/ARP)** | ❌ OPEN (Phase I.3 roadmap) |
| **Per-ns routing** | ❌ OPEN (Phase I.4 roadmap) |
| **veth devices** | ❌ OPEN (Phase I.3 roadmap) |
| **PO-NET-01 (this document)** | ✅ DELIVERED (2026-07-22) |
| **D1-ISO finding closure** | ⏳ PENDING Phase I.3 delivery OR claim narrowing |

**Recommendation:** Accept the narrowed claim ("TX escape prevention + RX isolation") and defer "full container isolation" claims until Phase I.3 delivers per-ns config. The current implementation provides the **security primitive** (fail-closed); the **feature arc** is a documented roadmap item, not a vulnerability.

---

**Document Version:** 1.0 (2026-07-22)  
**Author:** kernel-implement (Claude Code, MODE autonomous)  
**Review:** Pending (no Codex session required for documentation-only PO)  
**Related Artifacts:** D1-ISO-NETNS-DATAPLANE finding (R180), TX-gate implementation (2026-07-20), Phase I roadmap
