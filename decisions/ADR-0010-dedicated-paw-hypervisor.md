# ADR-0010: Dedicated Hypervisor for Tier 0 and Tier 1 PAWs

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-04-22 |
| Supersedes | - |
| Superseded By | - |

## Context

ADR-0002 placed all three PAWs as VMs on the Proxmox cluster. A cluster node compromise exposes the PAW running on it alongside the workloads it administers. The sharpest case is pve01, where paw01 shares a host with dc01 and ca01. An Intel NUC 10th Gen with 16GB RAM is available for reassignment.

## Decision

Build a dedicated single-node Proxmox host on the Intel NUC 10th Gen, named pve04, and run paw01 and paw02 there. paw03 remains on the cluster, pinned to pve03. Other elements of ADR-0002 are unchanged.

pve04 is an INFRA device. paw01 and paw02 attach to VLAN 30 through a trunk on the pve04 host NIC.

## Alternatives Considered

- **All three PAWs on the cluster (status quo)** - Leaves the hypervisor-plane exposure intact. Rejected.
- **All three PAWs on pve04** - 16GB across three Windows VMs plus Proxmox overhead is too tight. Rejected.
- **Physical workstation per tier** - Closest to production. Hardware not available at lab scale. Rejected.

## Consequences

- Tier 0 and Tier 1 admin credentials move behind hypervisor-level isolation. A cluster compromise no longer reaches paw01 or paw02.
- pve04 is a single point of failure for Tier 0 and Tier 1 PAW access. Break-glass accounts cover emergency paths.
- paw03 retains the same hypervisor-plane exposure as any other cluster VM. Accepted because the compromise path is scoped to Tier 2 workstation administration.
- Adds a fourth physical host to patch and maintain.
- Frees cluster RAM previously committed to paw01 and paw02.
- **Deviation:** Production PAW models use physical workstations per tier. paw03 additionally runs on the cluster rather than a dedicated host.

## References

- [Microsoft - Deploying a privileged access solution](https://learn.microsoft.com/en-us/security/privileged-access-workstations/privileged-access-deployment)