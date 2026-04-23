# ADR-0002: Per-Tier Privileged Access Workstation Model

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-04-14 |
| Supersedes | - |
| Superseded By | - |

> **Note:** The hypervisor-placement element of this decision is revised by ADR-0010. paw01 and paw02 run on the dedicated pve04 host. paw03 remains on the cluster. All other elements of this ADR stay in force.

## Context

The three tier model requires that credentials never authenticate to a system at a lower tier. That forces dedicated admin workstations. The daily driver used for email and browsing cannot be the same device that types Tier 0 credentials. The PAW model also needs to define how many PAWs exist and what each is permitted to administer.

## Decision

Deploy one dedicated Privileged Access Workstation per tier. Each PAW administers only its assigned tier.

| Hostname | Tier Administered | Purpose |
|---|---|---|
| paw01 | Tier 0 | DCs, CA, hypervisors, PBS, network gear, SIEM |
| paw02 | Tier 1 | Member servers, file servers |
| paw03 | Tier 2 | Windows workstation VMs |

PAWs run as VMs on the Proxmox cluster and are reached from the personal workstation through the Proxmox console, not through direct remote protocols. PAWs are not domain-joined. PAWs have no general internet egress. Outbound traffic is limited to the specific admin endpoints each PAW requires (Windows Update, package repositories, management consoles).

Network placement of PAWs is defined in [ADR-0006](ADR-0006-network-segmentation-and-ip-vlan-scheme.md).

## Alternatives Considered

- **Single PAW for all tiers** - Collapses the tier model entirely. Rejected.
- **RDP from the daily driver** - Administration should never login from the same device used for internet browsing and email. Rejected.
- **Domain-joined PAWs** - Creates a path where domain compromise reaches the admin workstations. Rejected.
- **Physical PAW per tier** - Dedicated physical machines. Hardware limitations prevent this at lab scale. Rejected.

## Consequences

- Three PAW machines to build, maintain, and patch.
- Administration is slower than direct remote protocols would allow, since all sessions route through the Proxmox console.
- Firewall rules from the PAW zone must be scoped per workstation so each PAW can only reach its assigned tier.
- **Deviation:** Production environments use physical PAW machines. This lab uses VMs due to hardware availability.

## References

- [Microsoft - Deploying a privileged access solution](https://learn.microsoft.com/en-us/security/privileged-access-workstations/privileged-access-deployment)