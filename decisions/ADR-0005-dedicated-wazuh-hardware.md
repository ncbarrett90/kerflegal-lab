# ADR-0005: Dedicated Hardware for Wazuh SIEM

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-04-16 |
| Supersedes | - |
| Superseded By | - |

## Context

Wazuh's resource requirements strain the Proxmox cluster, and security monitoring should be isolated from the systems it observes. If the cluster, domain, or hypervisor plane is compromised, a SIEM running inside that same environment cannot be trusted to report on the incident.

## Decision

Deploy wazuh01 on dedicated physical hardware, not as a VM on the Proxmox cluster. wazuh01 resides in the SECURITY zone (see [ADR-0006](ADR-0006-network-segmentation-and-ip-vlan-scheme.md)) alongside future security tooling. Agent and syslog ingestion from every zone into wazuh01 is permitted by firewall policy.

## Alternatives Considered

- **VM on pve01** - Resource contention and shared management plane with the workloads being monitored. Rejected.
- **VM on the cluster with dedicated node** - Still shares the Proxmox control plane, so a cluster or hypervisor compromise would reach the SIEM. Rejected.

## Consequences

- Additional physical host to manage and patch.
- Cleaner failure isolation between the monitored environment and the monitor.
- Production-aligned SIEM deployment.
- Frees cluster RAM for domain workloads.

## References

- [Wazuh - Installation guide](https://documentation.wazuh.com/current/installation-guide/index.html)
