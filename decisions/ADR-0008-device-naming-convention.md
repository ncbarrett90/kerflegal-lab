# ADR-0008: Device Naming Convention

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-04-20 |
| Supersedes | - |
| Superseded By | - |

## Context

A device naming convention needs to be in place before any device is provisioned. Many common schemes encode infrastructure details (site, VLAN, owner) that change over time and eventually leave hostnames misleading.

## Decision

Hostnames describe the device itself, not where it lives or who uses it. Format: `<role><sequence>`, lowercase, no separators, two-digit sequence. Zone, tier, and network information are tracked as attributes in the device registry rather than encoded in the hostname.

| Role Prefix | Meaning | Examples |
|---|---|---|
| `fw` | Firewall | `fw01` |
| `pve` | Proxmox VE hypervisor | `pve01`, `pve02` |
| `pbs` | Proxmox Backup Server | `pbs01` |
| `dc` | Domain controller | `dc01`, `dc02` |
| `ca` | Certificate authority | `ca01` |
| `fs` | File server | `fs01` |
| `srv` | Member server (general purpose) | `srv01` |
| `wazuh` | Wazuh SIEM | `wazuh01` |
| `paw` | Privileged access workstation | `paw01`, `paw02`, `paw03` |
| `ws` | Windows workstation | `ws01`, `ws02` |
| `sw` | Switch | `sw01`, `sw02` |
| `ap` | Access point | `ap01`, `ap02` |

## Alternatives Considered

- **OS plus user (e.g. `win-noah`)** - Role and ownership drift over the life of a device. Rejected.
- **VLAN or subnet encoded in hostname** - Devices move between subnets during the life of the lab. Rejected.
- **Theme-based (favorite bands, planets, etc.)** - Last lab used Meshuggah song titles and I could not tell what any device did from its name. Rejected.

## Consequences

- Log analysis and ad-hoc reference become easier: role is obvious from the name.
- Hostnames contain nothing that will change over time.
- Forces attribute tracking elsewhere (device registry) rather than relying on the name to carry metadata.
