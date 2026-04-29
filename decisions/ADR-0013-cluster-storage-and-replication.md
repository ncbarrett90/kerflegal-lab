# ADR-0013: Cluster Storage Layout and Replication

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-04-28 |
| Supersedes | - |
| Superseded By | - |

## Context

The Proxmox cluster ([ADR-0003](ADR-0003-proxmox-ve-cluster.md)) needs a storage layout and HA strategy. Each cluster node has an NVMe and a secondary SSD. pve04 ([ADR-0010](ADR-0010-dedicated-paw-hypervisor.md)) has the same configuration. pbs01 ([ADR-0004](ADR-0004-pbs-tier0-classification.md)) has a 2x 1TB HDD ZFS mirror.

A hypervisor can read every guest's memory and disk, so the cluster control plane is uniformly Tier 0 regardless of which VM tiers it hosts. Tier separation is enforced above the hypervisor (network, accounts, PAWs, GPO). VM placement is therefore free to optimize for hardware fit and availability rather than tier.

## Decision

Storage layout applies uniformly to pve01-04. NVMe holds the Proxmox OS only. The secondary SSD is a ZFS pool registered under the same Proxmox storage ID on every host, so per-VM replication can target peers by ID. All VM disks live on this pool.

The cluster (pve01-03) uses Proxmox storage replication for HA. Replication is configured per VM. Replication targets are chosen so redundant pairs sit on different hosts. For example, dc01 replicates to the node hosting dc02, so losing either host leaves the pair intact. On host failure, HA restarts the VM from the latest replicated snapshot. RPO equals the replication interval.

pve04 uses the same disk layout but does not replicate (standalone per ADR-0010). pbs01 is reserved for backups and is not used as live VM storage.

## Alternatives Considered

- **NFS share off pbs01** - Collapses production and backup onto one host. Conflicts with the separation in [ADR-0004](ADR-0004-pbs-tier0-classification.md). The mirror is sized for sequential backup IO, not random live VM IO. Rejected.
- **Ceph across the cluster's secondary SSDs** - Provides shared storage with no data loss on node failure, but pve03's 16GB RAM is too tight for Ceph's baseline overhead, and asymmetric SSD sizes complicate the pool layout. Rejected.
- **Local SSD with no replication, restore from PBS on host failure** - Simplest, but RTO becomes the time of a full restore, and any writes since the last backup are lost. Rejected.
- **Dedicated NAS or SAN appliance** - The production answer. Hardware not available at lab scale. Rejected.

## Consequences

- Loss of any one cluster node leaves replicated VMs recoverable on a surviving peer. No shared storage box sits in the dependency chain.
- VM placement across pve01-03 is driven by hardware fit and keeping redundant pairs on different hosts, not tier. The cluster control plane is uniformly Tier 0.
- Replication consumes additional SSD on receiving nodes, roughly 2x the working set per replicated VM with one target and 3x with two.
- pbs01 capacity stays dedicated to backup growth.
- **Deviation:** Production environments typically use enterprise shared storage (FC SAN, NVMe-oF, hyperconverged) for no data loss on node failure. The lab accepts an RPO equal to the replication interval as the tradeoff against cost and complexity.

## References

- [Proxmox VE Storage Replication](https://pve.proxmox.com/wiki/Storage_Replication)
- [Proxmox VE ZFS on Linux](https://pve.proxmox.com/wiki/ZFS_on_Linux)
- [Proxmox VE High Availability](https://pve.proxmox.com/wiki/High_Availability)
