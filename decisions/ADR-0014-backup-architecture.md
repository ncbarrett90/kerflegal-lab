# ADR-0014: Backup Architecture

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-04-28 |
| Supersedes | - |
| Superseded By | - |

## Context

The cluster will host the domain controllers and other Tier 0 services. A backup design needs to be in place before those workloads are built. pbs01 is classified Tier 0 ([ADR-0004](ADR-0004-pbs-tier0-classification.md)) and reserved for backups only ([ADR-0013](ADR-0013-cluster-storage-and-replication.md)).

## Decision

PBS is the backup target for all cluster VMs and Proxmox node configurations. It is not used for any other workload.

Each Proxmox node has its own PBS API token, scoped to a per-node namespace. Tokens have prune protection so a compromised node cannot delete its own backup history.

Backups run on a schedule with daily, weekly, and monthly retention. A scheduled verify job runs against the datastore to detect bit rot before a restore is needed. Specific numbers and cadences live in the backup procedures.

An off-site copy of the PBS datastore is part of the design. The mechanism and implementation are decided in a follow-up ADR after the cluster is operational.

## Alternatives Considered

- **No off-site copy** - Destruction or ransomware on pbs01 loses every backup along with the live data on the cluster. Rejected.
- **Single retention window** - All backups age out together. Latent corruption found later has nothing to restore from. Rejected.
- **Shared PBS token across all nodes** - A compromise of one node grants access to every node's backup history. Rejected.
- **Cloud backup as primary target** - Pulls Tier 0 data into a third-party service and adds latency, cost, and egress on every restore. Rejected.

## Consequences

- pbs01 keeps the Tier 0 isolation set by [ADR-0004](ADR-0004-pbs-tier0-classification.md). Live VM disks stay on cluster ZFS pools per [ADR-0013](ADR-0013-cluster-storage-and-replication.md).
- Prune protection prevents a compromised node from deleting its own backup history.
- Daily, weekly, and monthly retention covers both recent incidents and corruption that surfaces later.
- AD-specific recovery is not addressed here. It will be its own ADR alongside the AD build.
- Until off-site is in place, destroying pbs01 destroys every backup. Live VMs on the cluster are the only surviving copy in that window.
- **Deviation:** Production environments often run two PBS instances with replication or use immutable cloud storage. The lab uses a single PBS with off-site as a separate copy.

## References

- [Proxmox Backup Server documentation](https://pbs.proxmox.com/docs/)
- [Proxmox Backup Server access control](https://pbs.proxmox.com/docs/user-management.html)
