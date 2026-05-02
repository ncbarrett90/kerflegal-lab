# ADR-0016: Dedicated Cluster Communication VLAN

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-05-02 |
| Supersedes | - |
| Superseded By | - |

## Context

The Proxmox cluster ([ADR-0003](ADR-0003-proxmox-ve-cluster.md)) relies on corosync to track cluster membership and quorum. Corosync is sensitive to network delay and to other traffic competing for bandwidth on the network it shares. Running it on the management VLAN (INFRA) means cluster heartbeats compete with everyday admin traffic on the same segment. The cluster only spans pve01-03. pve04 is standalone per [ADR-0010](ADR-0010-dedicated-paw-hypervisor.md) and stays off this VLAN.

## Decision

Give corosync its own VLAN, 70 (CLUSTER, 10.0.70.0/24). The VLAN exists only at the switch and on the cluster hosts, with no fw01 interface and no gateway. Static IPs on pve01-03 mirror the INFRA octet (10.0.70.5-7). sw01 ports 4-6 add VLAN 70 to the existing tagged trunk per [ADR-0015](ADR-0015-hypervisor-trunk-tagging-policy.md). Hosts attach via `vmbr0.70`. One corosync link (link0) runs on this VLAN.

## Alternatives Considered

- **Share the INFRA VLAN for corosync** - Simplest, but cluster heartbeats and admin traffic end up on the same segment. Rejected for the small cost of standing up a dedicated VLAN.
- **Routable cluster VLAN with an fw01 interface** - Adds an OPNsense interface and a default-deny rule set with no functional gain, since corosync never needs to leave the cluster nodes. Rejected.
- **Two corosync links over a second physical NIC** - Gives the cluster a physically redundant path. The current hardware is single-NIC. Rejected for now, revisit if a second NIC becomes available.

## Consequences

- Corosync traffic is separated from management traffic at layer 2.
- The cluster VLAN is unreachable from any other zone, which limits its exposure.
- pve04 is unaffected and stays out of cluster networking entirely.
- **Deviation:** Single-NIC hardware means cluster traffic still travels over the same physical link and switch port as everything else. The separation is logical, not physical. Adding a second NIC is the path to true physical isolation.

## References

- [Proxmox VE Cluster Manager - Network Requirements](https://pve.proxmox.com/wiki/Cluster_Manager#_cluster_network)
- [ADR-0003](ADR-0003-proxmox-ve-cluster.md)
- [ADR-0006](ADR-0006-network-segmentation-and-ip-vlan-scheme.md)
- [ADR-0015](ADR-0015-hypervisor-trunk-tagging-policy.md)
