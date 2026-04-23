# ADR-0003: Proxmox VE Cluster for Lab Virtualization

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-04-15 |
| Supersedes | - |
| Superseded By | - |

## Context

The lab needs a virtualization platform to host the bulk of its workloads (DCs, member servers, PAWs, clients). The hardware inventory includes three Optiplex nodes suitable for clustering. The choice is between standalone hosts and a cluster, and between distributing workloads across nodes or pinning by tier.

## Decision

Run lab virtualization on a three-node Proxmox VE cluster (pve01, pve02, pve03) with HA enabled. Workloads are distributed across nodes for availability and resource utilization rather than pinned to specific nodes by tier. Three nodes satisfy quorum.

## Alternatives Considered

- **Standalone hosts** - No HA, no live migration, no shared management plane. Rejected.
- **Pin workloads by tier across the cluster** - Shared management plane means pinning does not produce real tier isolation. HA either cannot cross tier boundaries (defeats HA) or violates the pinning (defeats the pinning). Rejected.

## Consequences

- HA and live migration available across all workloads.
- Shared management plane across tiers.
- **Deviation:** Production Tier 0 typically runs on separate hypervisor infrastructure. This lab shares the hypervisor plane across tiers. Tier separation is enforced at the network, firewall, credential, and identity layers, not at the hypervisor.

## References

- [Proxmox VE - Cluster Manager](https://pve.proxmox.com/wiki/Cluster_Manager)