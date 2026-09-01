# ADR-001: Anti-Affinity Enforcement for Flow Virtual Networking BGP Gateways

**Status:** Proposed
**Date:** 2026-09-01
**Deciders:** Enterprise Architecture / Network Engineering

## Context

A network architecture review of our Flow Virtual Networking (FVN) deployment found both BGP Gateway VMs servicing a production VPC placed on the same AHV host. This removes the redundancy the two-gateway design exists to provide. BGP is a control-plane function in FVN, not part of the data forwarding path, so co-location doesn't disrupt existing east/west traffic — but it does mean a single host failure would drop both BGP sessions to the ToR switches simultaneously, risking withdrawal of the VPC's dynamically-advertised routes until a gateway respawns on another host.

Nutanix's documented Gateway Fault Tolerance behavior explains how this happens: on a host failure, a displaced gateway is moved to another active node — but if every eligible node is already hosting a gateway, one node will temporarily host two. That is expected platform behavior, not a defect. What turned a transient reshuffle into a standing condition is that no anti-affinity policy was ever configured between the two gateway VMs at initial deployment, so nothing forced them back apart afterward — consistent with what we found when we traced back through the original build.

Cluster size compounds this. The Acropolis Leader is never eligible to host a gateway, so a 3-node cluster running the standard two-gateway design has zero slack: every non-leader host is already a gateway host, and any single maintenance or failure event triggers the co-location fallback by design.

## Decision

Configure an explicit VM-VM anti-affinity policy between the two BGP Gateway VMs per VPC — via Prism Central, Category-based (not available in Prism Element) — so the scheduler cannot place both on one host under normal placement or failover. Pair this with confirming each gateway is peered to both ToR switches and that gateway placement is VLAN-backed rather than overlay-backed.

## Options Considered

### Option A: VM-VM Anti-Affinity Policy (Prism Central Categories)
| Dimension | Assessment |
|---|---|
| Complexity | Low |
| Cost | None — native platform feature |
| Effectiveness | Structural — prevents recurrence under normal operation |
| Team familiarity | Low — new to this team, standard AHV feature |

**Pros:** Enforced at placement time, not just detected afterward; no new tooling required.
**Cons:** Documented as best-effort at the extreme edge — if every eligible host is down simultaneously, the platform's own fallback can still co-locate the gateways; anti-affinity persistence through DR failover/failback isn't guaranteed and needs validation.

### Option B: Manual Placement Monitoring Only
| Dimension | Assessment |
|---|---|
| Complexity | Low |
| Cost | Ongoing engineering time |
| Effectiveness | Reactive only |
| Team familiarity | High |

**Pros:** No configuration change required.
**Cons:** Doesn't prevent recurrence, only detects it after the fact — the same gap that let this go unnoticed the first time.

### Option C: Increase Cluster Node Headroom
| Dimension | Assessment |
|---|---|
| Complexity | Medium — capacity planning, possibly new hardware |
| Cost | Higher — additional node(s) |
| Effectiveness | Reduces likelihood, doesn't eliminate it |
| Team familiarity | High |

**Pros:** Also improves general cluster fault tolerance, not just gateway placement.
**Cons:** Doesn't structurally prevent co-location on its own — addresses probability, not the missing control.

## Trade-off Analysis

Option A is the only option that changes what the scheduler is allowed to do, rather than relying on someone noticing or on having enough spare capacity on hand. It's also the cheapest and fastest to implement. Option C is worth pursuing as a complementary improvement on undersized clusters, but shouldn't substitute for A. Option B is not a fix by itself — only a detector.

## Consequences

- Removes single-host blast radius for BGP control-plane availability on covered VPCs.
- Adds Prism Central Category management as an ongoing part of network configuration, not a one-time setting.
- Anti-affinity enforcement is not proven to survive DR failover/failback automatically — needs a validation test, not an assumption.
- Does not eliminate co-location risk in the extreme case where every eligible host is simultaneously down — that residual risk should be explicitly accepted, not silently assumed away.

## Action Items
1. [ ] Confirm current BGP Gateway VM placement across all production clusters running FVN
2. [ ] Create a Prism Central Category-based anti-affinity policy for each VPC's gateway pair
3. [ ] Confirm gateway placement is VLAN-backed, not overlay-backed (required for MD5 BGP session authentication)
4. [ ] Confirm each gateway is peered to both ToR switches
5. [ ] Check eligible-host headroom against cluster size for each production cluster (Acropolis Leader exclusion + gateway count)
6. [ ] Add a periodic check for gateway placement drift
7. [ ] Test anti-affinity persistence through a DR failover/failback cycle

## References
- Nutanix Bible — Flow Virtual Networking: https://www.nutanixbible.com/12c-book-of-network-services-flow-virtual-networking.html
