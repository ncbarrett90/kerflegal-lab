# Proxmox Cluster Formation

## Purpose
Form a three-node Proxmox VE cluster on pve01-03 with corosync running on VLAN 70 per [ADR-0003](../decisions/ADR-0003-proxmox-ve-cluster.md) and [ADR-0016](../decisions/ADR-0016-dedicated-cluster-vlan.md). pve04 is excluded by design per [ADR-0010](../decisions/ADR-0010-dedicated-paw-hypervisor.md).

## Prerequisites
- [07-cluster-vlan-buildout](07-cluster-vlan-buildout.md) complete and validated
- pve01-03 host no VMs or containers
- Cluster name `kerflegal` (override at step 1 if different)
- Root SSH access to all three nodes via key auth

## Procedure

### 1. Create the cluster on pve01
SSH to pve01 as `root`:
```bash
pvecm create kerflegal --link0 10.0.70.5
pvecm status
```
Expect a `Quorum information` block reporting one node, Quorate Yes. The link0 address must be 10.0.70.5.

### 2. Join pve02
SSH to pve02 as `root`:
```bash
pvecm add 10.0.10.5 --link0 10.0.70.6
```
The peer IP is the management IP (10.0.10.5), not the cluster VLAN IP, because the pveproxy cert on pve01 is generated for the hostname and primary IP. Connecting to 10.0.70.5 fails hostname verification on the bootstrap API call. The `--link0` flag still pins corosync itself to the cluster VLAN.

Enter the pve01 root password when prompted. The local web UI may drop briefly while `/etc/pve` re-syncs.

Verify from pve01:
```bash
pvecm status
```
Expect two nodes, Quorate Yes.

### 3. Join pve03
SSH to pve03 as `root`:
```bash
pvecm add 10.0.10.5 --link0 10.0.70.7
```
Enter the pve01 root password.

Verify from any node:
```bash
pvecm status
pvecm nodes
```
Expect three nodes, Quorate Yes, Expected votes 3.

### 4. Confirm corosync on the cluster VLAN
On each node:
```bash
corosync-cfgtool -s
```
Expect a single ring with `LINK ID 0` bound to the local node's 10.0.70.x address.

## Validation
- `pvecm status` reports three nodes, Quorate Yes, Expected votes 3
- `corosync-cfgtool -s` shows link0 on 10.0.70.x for each node
- Web UI on any node lists all three under Datacenter, all green
- `noah.barrett@pam` retains Administrator on the cluster scope

## Rollback
Cluster teardown is destructive. If a single node failed to join cleanly, run `pvecm delnode <node>` from a remaining quorate peer, fix the underlying issue, then retry the join. If the cluster as a whole is wedged at this stage, reinstall pve01-03 from ISO and start over. There are no VMs to preserve yet.

## Troubleshooting

**Symptom:** `pvecm add` returns `hostname verification failed`.
**Cause:** Cert is bound to the management IP, not the cluster VLAN IP.
**Fix:** Use the management IP as the `pvecm add` peer. `--link0` still pins corosync to the cluster VLAN.

**Symptom:** Web UI seems to accept only pve01's password across all nodes.
**Cause:** Cluster login session is shared, navigating between nodes rides the existing login.
**Fix:** Test each node's password in a fresh incognito tab pointed at that node's IP.
