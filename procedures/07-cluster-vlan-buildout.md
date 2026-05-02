# Cluster VLAN Buildout

## Purpose
Stand up VLAN 70 (CLUSTER) for corosync ring traffic on pve01-03 per [ADR-0016](../decisions/ADR-0016-dedicated-cluster-vlan.md). Run before [08-cluster-formation](08-cluster-formation.md).

## Prerequisites
- pve01-03 baselined per [06-proxmox-node-baseline](06-proxmox-node-baseline.md), reachable on INFRA at 10.0.10.5-7
- sw01 admin access at 10.0.10.15
- Workstation SSH key in place for `root` on pve01-03

## Procedure

### 1. Create VLAN 70 on sw01
Web UI > Switching > VLAN > Advanced > VLAN Configuration. Add VLAN 70, name `CLUSTER`. Save.

### 2. Tag VLAN 70 on ports 4-6
Switching > VLAN > Advanced > VLAN Membership. Select VLAN 70. Set ports 4, 5, and 6 to Tagged. Leave all other ports as Excluded. Save.

After save, ports 4-6 carry VLANs 10/30/40/50/60/70 tagged with PVID 1 unused, consistent with [ADR-0015](../decisions/ADR-0015-hypervisor-trunk-tagging-policy.md).

### 3. Add `vmbr0.70` on each cluster node
SSH to pve01 as `root`, append to `/etc/network/interfaces`:
```
auto vmbr0.70
iface vmbr0.70 inet static
    address 10.0.70.5/24
```
Repeat for pve02 (`address 10.0.70.6/24`) and pve03 (`address 10.0.70.7/24`).

No gateway, no DNS. This VLAN is L2 only.

Apply on each node:
```bash
ifreload -a
ip -br addr show vmbr0.70
```
Expect `vmbr0.70` UP with the configured IP.

### 4. Verify L2 reachability
From pve01:
```bash
ping -c 3 10.0.70.6
ping -c 3 10.0.70.7
```
Repeat from pve02 and pve03 to the other two IPs. All pings should succeed.

## Validation
- `ip -br addr` on each cluster node shows `vmbr0.70` carrying 10.0.70.5/6/7 respectively
- All three cluster nodes ping each other on the 10.0.70.x addresses
- Workstation on HOME cannot reach 10.0.70.x, confirming L2 isolation

## Rollback
- **Per host:** delete the `vmbr0.70` block from `/etc/network/interfaces`, `ifreload -a`.
- **Switch:** remove VLAN 70 membership from ports 4-6, delete VLAN 70.
