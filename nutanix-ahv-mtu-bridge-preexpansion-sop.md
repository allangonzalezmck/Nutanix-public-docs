# SOP: Pre-Expansion Bridge Split & Jumbo Frame (MTU 9000) Remediation for New AHV Hosts

| Field | Value |
|---|---|
| **Applies to** | AOS 7.3.1.5 / AHV 10.3.1.4 (new hosts imaged to the same versions) |
| **Owner** | Enterprise Architecture / Infrastructure Engineering |
| **Classification** | Internal |
| **Document type** | Standard Operating Procedure |
| **Last updated** | 2026-08-19 |

> **Validate before production use.** The `ovs-vsctl`/OVS-based commands in this SOP are standard Open vSwitch operations that have been stable across AHV releases for years and are low-risk to rely on as-is. The CVM/cluster-level commands (NCC check names, `ecli`/`ncli` output formatting, CLI-driven expand-cluster syntax) can shift between AOS releases — confirm exact syntax against the Nutanix Support Portal / release notes for 7.3.1.5 before running this in production, then this note can be removed from your internal copy.

---

## 1. Purpose & Scope

This SOP documents the workaround for a recurring Nutanix Foundation/cluster-expansion behavior: new hosts imaged via Foundation come up with a single bridge (`br0`) bonding all four physical NICs at the default MTU of 1500. When the production cluster's network is configured for jumbo frames (MTU 9000), the expand-cluster workflow does not reliably push the jumbo frame configuration to the new host's bridges/NICs. This leaves the new host at MTU 1500, which Prism Central reports as a network/MTU configuration mismatch after the host joins.

This document covers:
- Pre-expansion host preparation: splitting the default all-NIC bond into two bridges and setting MTU 9000
- Validation commands and representative expected output for each step
- Cluster expansion and post-expansion validation
- Rollback guidance

**Out of scope:** Foundation imaging itself, physical cabling, and upstream switch/VLAN/jumbo-frame configuration (these must already be in place — see Prerequisites).

---

## 2. Background: Why This Happens

- Foundation images every new AHV host with one bridge (`br0`) and one uplink bond (commonly `br0-up`) containing all discovered NICs (`eth0`–`eth3`), at MTU 1500.
- Since AOS 6.1, bridges are managed centrally through the **Virtual Switch (VS)** abstraction in Prism (e.g., `VS0` mapped to `br0`). When a host joins the cluster, Genesis is expected to reconcile the new host's bridge/uplink layout against the cluster's existing Virtual Switch definition.
- In practice, this reconciliation does not reliably carry over **MTU** (and in split-bridge topologies, the bridge/bond split itself) to the new host. The host joins with `br0` at MTU 1500 while the rest of the cluster runs MTU 9000, and Prism Central raises a network configuration-mismatch alert.
- The workaround is to bring the new host's bridge topology and MTU into line with production **before** running Expand Cluster, so there is nothing left for the (unreliable) automatic reconciliation to fix.

---

## 3. Prerequisites

- [ ] Host has been imaged via Foundation with AOS 7.3.1.5 / AHV 10.3.1.4 and is **not yet added** to the cluster.
- [ ] Root SSH access to the new AHV host's Foundation-assigned/temporary IP.
- [ ] Upstream physical switch ports for this host are **already** configured for jumbo frames (MTU ≥ 9000) and the correct VLANs/trunking. This SOP does not configure the switch side — confirm with the network team before proceeding. If the switch side isn't ready, stop here; setting host MTU without switch support will pass local checks but fail the jumbo-frame connectivity test in Section 6.6.
- [ ] Production cluster's actual bridge names, bond names, bond mode, and Virtual Switch names are confirmed (fill in the table in Section 4 — do not rely on the samples).
- [ ] Standard change window/approval obtained.
- [ ] IPMI/console access available as a fallback in case a network command locks out SSH.

---

## 4. Production Reference Configuration — Fill In Before Use

The values below are **samples only**. Bridge, bond, and Virtual Switch names are case-sensitive strings that Genesis/Prism compare literally across hosts — `BR0` and `br0` are not the same to the platform. Replace every sample value with what's actually configured on your production cluster before executing any command in Section 6.

| Field | Sample value used in this doc | Your production value |
|---|---|---|
| Bridge 1 name | `br0` | _______ |
| Bridge 1 NICs | `eth0`, `eth1` | _______ |
| Bridge 1 uplink bond name | `br0-up` | _______ |
| Bridge 1 bond mode | `active-backup` | _______ (active-backup / balance-slb / balance-tcp+LACP) |
| Bridge 1 MTU | `9000` | _______ |
| Bridge 1 typical role | Management / User VM / CVM external traffic | _______ |
| Bridge 2 name | `br1` | _______ |
| Bridge 2 NICs | `eth2`, `eth3` | _______ |
| Bridge 2 uplink bond name | `br1-up` | _______ |
| Bridge 2 bond mode | `active-backup` | _______ |
| Bridge 2 MTU | `9000` | _______ |
| Bridge 2 typical role | Storage / CVM backplane traffic | _______ |
| Virtual Switch name(s) | `VS0`, `VS1` | _______ |
| VLAN(s) | — | _______ |

---

## 5. Pre-Expansion Host Preparation — Overview

Perform all of Section 6 **before** running Expand Cluster. Nothing in this section touches the production cluster; it only configures the standalone new host.

## 6. Step-by-Step Procedure

### 6.1 Connect to the New AHV Host

```bash
ssh root@<new_host_foundation_ip>
```

### 6.2 Baseline — Inventory the Current OVS State

```bash
ovs-vsctl show
```

> **Expected output (representative — Foundation default):**
> ```
> 3c8d1234-abcd-46f1-9c21-abcdef123456
>     Bridge "br0"
>         Port "br0"
>             Interface "br0"
>                 type: internal
>         Port "br0-up"
>             Interface "eth3"
>             Interface "eth2"
>             Interface "eth1"
>             Interface "eth0"
>         Port "vnet0"
>             Interface "vnet0"
>     ovs_version: "2.17.9"
> ```
> One bridge, one bond, all four NICs — this is the Foundation default that needs to be split.

```bash
ovs-vsctl list-br
```
> **Expected:** `br0`

```bash
for i in eth0 eth1 eth2 eth3; do echo "== $i =="; ovs-vsctl get interface $i mtu; done
```
> **Expected:** `1500` for all four (this is the bug's symptom).

### 6.3 Split the Default Bond into `br0` (eth0/eth1) and `br1` (eth2/eth3)

**a) Remove the existing all-NIC uplink bond from `br0`:**
```bash
ovs-vsctl del-port br0 br0-up
```

**b) Recreate `br0`'s uplink bond with only eth0/eth1:**
```bash
ovs-vsctl add-bond br0 br0-up eth0 eth1 -- set port br0-up bond_mode=active-backup
```
Replace `active-backup` with your production bond mode (Section 4) if different — e.g. `balance-slb`, or for LACP: add `lacp=active` to the `set port` clause.

**c) Create the `br1` bridge:**
```bash
ovs-vsctl add-br br1
```

**d) Create `br1`'s uplink bond with eth2/eth3:**
```bash
ovs-vsctl add-bond br1 br1-up eth2 eth3 -- set port br1-up bond_mode=active-backup
```

**Validate the split:**
```bash
ovs-vsctl show
```
> **Expected output (representative):**
> ```
> 3c8d1234-abcd-46f1-9c21-abcdef123456
>     Bridge "br1"
>         Port "br1"
>             Interface "br1"
>                 type: internal
>         Port "br1-up"
>             Interface "eth2"
>             Interface "eth3"
>     Bridge "br0"
>         Port "br0"
>             Interface "br0"
>                 type: internal
>         Port "br0-up"
>             Interface "eth0"
>             Interface "eth1"
>         Port "vnet0"
>             Interface "vnet0"
>     ovs_version: "2.17.9"
> ```

### 6.4 Set MTU 9000

**Physical NICs:**
```bash
for i in eth0 eth1 eth2 eth3; do ovs-vsctl set interface $i mtu_request=9000; done
```

**Uplink bonds:**
```bash
ovs-vsctl set interface br0-up mtu_request=9000
ovs-vsctl set interface br1-up mtu_request=9000
```

**Bridge internal interfaces:**
```bash
ovs-vsctl set interface br0 mtu_request=9000
ovs-vsctl set interface br1 mtu_request=9000
```

`mtu_request` is the OVSDB column used to request an MTU on an OVS-managed interface; the kernel then reports the negotiated value back in the `mtu` column, which is what the next step checks.

### 6.5 Validate MTU Was Applied

```bash
for i in eth0 eth1 eth2 eth3 br0-up br1-up br0 br1; do echo "== $i =="; ovs-vsctl get interface $i mtu; done
```
> **Expected:** `9000` for every interface listed. If any interface still shows `1500` or errors out, re-check the `mtu_request` was set on that exact interface name (Section 6.4) before moving on.

Cross-check at the kernel/OS level:
```bash
ip link show eth0
ip link show br0
ip link show br1
```
> **Expected output (representative):**
> ```
> 4: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9000 qdisc mq master ovs-system state UP mode DEFAULT group default qlen 1000
>     link/ether 3c:ec:ef:aa:bb:cc brd ff:ff:ff:ff:ff:ff
> ```
> `mtu 9000` should appear in every interface's link line.

### 6.6 Validate Jumbo Frames End-to-End (Before Joining the Cluster)

Local MTU settings only prove the host will *accept* jumbo frames — not that the physical path supports them. Test against an existing cluster host on the same L2/VLAN as `br0`'s temporary Foundation IP:

```bash
ping -M do -s 8972 <existing_host_ip_same_vlan>
```
`8972` = 9000 MTU − 20 bytes IPv4 header − 8 bytes ICMP header. This forces the packet to the full jumbo size with the "don't fragment" bit set, so any MTU gap in the path surfaces immediately.

> **Expected output — success:**
> ```
> PING 10.10.20.15 (10.10.20.15) 8972(9000) bytes of data.
> 8980 bytes from 10.10.20.15: icmp_seq=1 ttl=64 time=0.412 ms
> 8980 bytes from 10.10.20.15: icmp_seq=2 ttl=64 time=0.398 ms
>
> --- 10.10.20.15 ping statistics ---
> 2 packets transmitted, 2 received, 0% packet loss, time 1001ms
> ```
>
> **Expected output — failure (host MTU is fine, path is not):**
> ```
> PING 10.10.20.15 (10.10.20.15) 8972(9000) bytes of data.
> ping: local error: Message too long, mtu=1500
> ```
> A failure here means an upstream switchport isn't passing jumbo frames — stop and engage the network team rather than proceeding to Section 7.

If `br1` carries no routable/management IP before the host joins the cluster (common when it's a dedicated backplane network with no host-level IP until the CVM comes up), defer full end-to-end validation on that path to Section 8.1, and use bond/link status as a pre-join proxy instead:

```bash
ovs-appctl bond/show br0-up
ovs-appctl bond/show br1-up
```
> **Expected output (representative):**
> ```
> ---- br0-up ----
> bond_mode: active-backup
> bond may include 2 slaves
> updelay: 0 ms
> downdelay: 0 ms
> lacp_status: off
>
> slave eth0: enabled
>     active slave
>     may_enable: true
>
> slave eth1: enabled
>     may_enable: true
> ```
> Both slaves should show `may_enable: true`. `may_enable: false` on a slave indicates a link/negotiation problem at the physical layer — resolve before joining the cluster.

### 6.7 Confirm Persistence

AHV persists OVS configuration in OVSDB across reboots by default — no separate "save" step is required. To validate during the same maintenance window:

```bash
reboot
```
After the host comes back:
```bash
ovs-vsctl show
for i in eth0 eth1 eth2 eth3; do ovs-vsctl get interface $i mtu; done
```
> **Expected:** identical to the state validated in 6.3–6.5 — the `br0`/`br1` split intact, MTU 9000 retained on all interfaces.

---

## 7. Cluster Expansion

### 7.1 Pre-Expansion Checklist

- [ ] Section 6 completed and validated on the new host
- [ ] Bridge names (`br0`/`br1`) and bond names match production exactly (Section 4)
- [ ] Jumbo-frame ping validated wherever a routable IP was available
- [ ] Change window approved
- [ ] Existing cluster is healthy before adding the node — from any existing CVM:
```bash
cluster status
ncc health_checks run_all
```
> **Expected:** `cluster status` reports all services `UP` on all existing CVMs; `ncc health_checks run_all` completes with no unresolved FAILs relevant to networking or cluster health.

### 7.2 Add the Host

Perform the actual node-join through Prism: **Prism Element → gear icon → Expand Cluster → select the discovered node → Expand Cluster.** The UI wizard runs live pre-checks (IP conflicts, AOS/AHV version match, factory configuration) that materially reduce the risk of a partial or failed join, and this workflow has been the Nutanix-recommended path across releases.

If your change process requires CLI-driven expansion instead, that is normally initiated from an existing CVM via the `cluster` add-node workflow. Exact flags for this have changed across AOS releases — confirm current syntax against the Nutanix Support Portal / release notes for 7.3.1.5 before using it rather than relying on a fixed command here.

### 7.3 Monitor the Expansion Task

From any existing CVM:
```bash
ecli task.list include_completed=false
```
> **Expected:** an in-progress expand-cluster task advancing toward 100% with no error status.

```bash
ncli host list
```
> **Expected:** the new host now listed with its host/CVM IP and a normal (`UP`/`NORMAL`) status once the task completes.

---

## 8. Post-Expansion Validation

### 8.1 Cluster-Wide Bridge & MTU Consistency

From any CVM (this fans the command out to every host in the cluster, including the one just added):
```bash
allssh "ovs-vsctl show"
allssh "for i in eth0 eth1 eth2 eth3; do ovs-vsctl get interface \$i mtu; done"
```
> **Expected:** every host — including the new one — returns `9000` for all four NICs, and the `br0`/`br1` bridge layout is identical across hosts.

### 8.2 NCC Health Check

```bash
ncc health_checks run_all
```
or, scoped to networking:
```bash
ncc health_checks network_checks run_all
```
> **Expected:** PASS on the network/bond/MTU-consistency checks. Confirm the exact check name in your NCC version's output — check naming has shifted across NCC releases, so don't hard-code a specific check name into automation without verifying it against this cluster's installed NCC version first.

### 8.3 Prism Central — Confirm the Mismatch Alert Clears

In Prism Central: **Operations → Alerts**, locate the network/MTU configuration-mismatch alert previously raised for this cluster. It should auto-resolve once 8.1 comes back clean; if not, resolve/acknowledge it manually per your standard alert-handling process only after 8.1 and 8.2 both confirm clean.

### 8.4 Virtual Switch Reconciliation (Belt-and-Suspenders)

Optionally force the cluster's central Virtual Switch definition to re-apply across all hosts (including the new one) so it's the authoritative source going forward:

- UI: **Prism Element → Settings → Network Configuration → Virtual Switch → edit → Save**
- CLI (read-only inspection, safe to run anytime):
```bash
acli net.list_virtual_switch
acli net.get_virtual_switch name=VS0
```
Before pushing an update via `acli net.update_virtual_switch`, run `acli help net.update_virtual_switch` to confirm the current parameter set for this build — the update command is state-changing across every host in the cluster, so don't run it from memory.

---

## 9. Rollback / Troubleshooting

| Symptom | Likely cause | Action |
|---|---|---|
| `ping -M do -s 8972` fails but local MTU shows 9000 | Upstream switchport not passing jumbo frames | Stop; engage network team. Do not proceed to Section 7. |
| `ovs-appctl bond/show` shows `may_enable: false` on a slave | Cabling, port-channel/LACP mismatch, or link negotiation issue | Verify physical cabling and switch-side bond/LACP config before proceeding. |
| Expand Cluster fails at network validation | Bridge/bond naming or MTU still doesn't match production | Re-verify Section 4 values were used exactly (case-sensitive) and re-run Section 6.5. |
| Need to fully revert the new host to Foundation defaults | Aborting the procedure | Run the revert commands below. |

**Revert to a single `br0` bridge with all four NICs (Foundation default):**
```bash
ovs-vsctl del-port br1 br1-up
ovs-vsctl del-br br1
ovs-vsctl del-port br0 br0-up
ovs-vsctl add-bond br0 br0-up eth0 eth1 eth2 eth3 -- set port br0-up bond_mode=active-backup
```

---

## 10. Appendix A — Command Reference Cheat Sheet

| Command | Purpose |
|---|---|
| `ovs-vsctl show` | Dump full bridge/port/interface topology |
| `ovs-vsctl list-br` | List bridge names |
| `ovs-vsctl del-port <bridge> <port>` | Remove a port (bond) from a bridge |
| `ovs-vsctl add-bond <bridge> <bond> <if1> <if2> -- set port <bond> bond_mode=<mode>` | Create a bridge uplink bond from two interfaces |
| `ovs-vsctl add-br <bridge>` | Create a new bridge |
| `ovs-vsctl set interface <if> mtu_request=<n>` | Request an MTU on an interface |
| `ovs-vsctl get interface <if> mtu` | Read the kernel-negotiated MTU |
| `ovs-appctl bond/show <bond>` | Bond member status (active/backup, may_enable) |
| `ip link show <if>` | OS-level confirmation of MTU/link state |
| `ping -M do -s 8972 <ip>` | End-to-end jumbo-frame validation (9000 MTU) |
| `cluster status` | Cluster-wide service health (from CVM) |
| `ncc health_checks run_all` | Full NCC health check sweep |
| `ecli task.list include_completed=false` | Monitor in-progress cluster tasks (e.g. expand-cluster) |
| `ncli host list` | List cluster hosts and status |
| `allssh "<cmd>"` | Fan a command out to every CVM/host in the cluster |
| `acli net.list_virtual_switch` / `net.get_virtual_switch name=<vs>` | Inspect Virtual Switch definitions |

---

## 11. Revision History

| Version | Date | Author | Change |
|---|---|---|---|
| 1.0 | 2026-08-19 | Enterprise Architecture | Initial SOP |
