# ADR-0004: Proxmox Backup Server Tier 0 Classification

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-04-15 |
| Supersedes | - |
| Superseded By | - |

## Context

Proxmox Backup Server (PBS) stores backups of all virtual machines running on the cluster, as well as configuration backups of the Proxmox nodes. This includes the domain controllers and other critical Tier 0 devices. Compromise of the backup server is equivalent to compromise of the data it holds, including the NTDS.dit.

## Decision

PBS is classified as a Tier 0 device under the three tier model ([ADR-0001](ADR-0001-three-tier-administrative-model.md)). Only Tier 0 admin accounts authenticate to it. It resides in a Tier 0 zone.

## Alternatives Considered

- **PBS as Tier 1** - PBS stores Tier 0 data. Classifying it as Tier 1 would grant Tier 1 administrators effective access to Tier 0 material and collapse the tier boundary. Rejected.

## Consequences

- Applying this classification throughout the lab adds overhead and complexity, but provides necessary context for access control decisions on the backup plane.
- Backup restore operations require Tier 0 credentials, which is the correct posture.

## References

- [Proxmox Backup Server documentation](https://pbs.proxmox.com/docs/)