# Device Information Template

> **Usage:** Every device in the lab registry uses the Core section. Attach class modules based on what the device is. A Proxmox host gets Core + Compute + Hypervisor. A domain controller gets Core + Compute + Service. A managed switch gets Core + Network. Fill with `N/A` where a field doesn't apply - blank means "not yet filled," `N/A` means "considered and not applicable."
>
> **Module selection:**
> - **Compute** - anything running a general-purpose OS (physical hosts, VMs, containers)
> - **Network** - switches, firewalls, access points, routers
> - **Hypervisor** - Proxmox hosts, or any device running a hypervisor role
> - **Service** - any device hosting a defined service (AD DS, DNS, DHCP, SMB, SIEM, backup target, etc.). One copy per distinct service.

---

## Core (required for every device)

### Identity
| Field | Value |
|---|---|
| Asset ID | |
| Hostname | |
| Device Class (Physical / VM / Container / Network) | |
| Role / Purpose | |
| Criticality Tier (0 / 1 / 2) | |
| Status (Online / Offline / Maintenance / Decommissioned) | |

### Placement & Access
| Field | Value |
|---|---|
| Network Zone (INFRA / SECURITY / PAW / IDENTITY / SERVERS / CLIENTS / HOME) | |
| Primary IP / CIDR | |
| Management Interface / Access Method | |

### Ownership & Lifecycle
| Field | Value |
|---|---|
| Technical Owner | |
| Deployed Date | |
| Last Reviewed | |
| Planned Decommission | |

### Dependencies
| Field | Value |
|---|---|
| Depends On (what this device requires - parent hypervisor, upstream switch, storage, etc.) | |
| Depended On By (what requires this device) | |

**Criticality Tier reference:**
- **Tier 0** - Identity and core infrastructure. Compromise = environment compromise. (DCs, PKI, LAPS infrastructure, backup servers, hypervisors hosting Tier 0 workloads)
- **Tier 1** - Servers. Member servers, file servers, application servers, SIEM.
- **Tier 2** - Workstations and endpoints.

**Asset ID convention:**
- **PH-NNN** - Physical host (PH-001, PH-002, ...)
- **VM-NNN** - Virtual machine (VM-001, VM-002, ...)
- **CT-NNN** - Container (CT-001, CT-002, ...)
- **NET-NNN** - Network device - switch, firewall, AP (NET-001, NET-002, ...)

Numbers are sequential per prefix and never reused. If a device is decommissioned, its ID is retired.


**Network Zone reference** (canonical set per [ADR-0006](../../decisions/ADR-0006-network-segmentation-and-ip-vlan-scheme.md)):
- **INFRA** - Tier 0. Hypervisor admin, PBS, network gear management.
- **SECURITY** - Tier 0. Wazuh SIEM and future security tooling. Accepts log and agent traffic from all other zones.
- **PAW** - Tier 0. Privileged Access Workstations. Originates administrative access to other zones.
- **IDENTITY** - Tier 0. Domain controllers and certificate authority.
- **SERVERS** - Tier 1. Member servers, file servers, application servers.
- **CLIENTS** - Tier 2. Domain-joined Windows workstations.
- **HOME** - Out of scope. Personal workstation and home network. Appears in firewall policy as a source or destination but is not a lab asset.

---

## Compute Module
*Attach for: physical hosts, VMs, containers.*

### System
| Field | Value |
|---|---|
| Operating System | |
| OS Version / Build | |
| Architecture | |
| Kernel / Image Version | |
| Hardening Baseline Applied (e.g., CIS Win11 v3.0.0 L1) | |

### Hardware / Resources
| Field | Value |
|---|---|
| Physical Model (if physical) | |
| CPU (model or vCPU count) | |
| RAM | |
| Storage (type, capacity, layout) | |
| Primary MAC Address | |
| Parent Hypervisor / Host (if VM) | |

### Patching
| Field | Value |
|---|---|
| Patching Method (WSUS / Intune / unattended-upgrades / manual) | |
| Automatic Updates (Yes / No) | |
| Last Patch Date | |
| Next Scheduled Patch Window | |
| Reboot Policy | |

### Backup
| Field | Value |
|---|---|
| Backup Method (PBS / image / file-level / N/A) | |
| Backup Schedule | |
| Backup Target | |
| Retention Policy | |
| Last Verified Restore | |
| Snapshot Strategy | |

### Local Accounts
*Accounts that exist on this device itself - local admin, local service accounts, break-glass. Directory accounts (domain users, domain admins, gMSAs) are NOT listed here - they live in the directory account registry and are referenced by name in the Service module.*

| Account Name | Type (Local Admin / Local Service / Break-Glass) | Auth Method | Credential Storage | Last Rotated | Notes |
|---|---|---|---|---|---|
| | | | | | |

---

## Network Module
*Attach for: switches, firewalls, APs, routers. Also attach to any host with non-trivial network config (trunk ports, multiple NICs, etc.).*

### Interfaces
| Interface | Purpose | VLAN / Tag Mode | IP / CIDR | Notes |
|---|---|---|---|---|
| | | | | |

### Physical Connectivity
| Field | Value |
|---|---|
| Upstream Device / Port | |
| Downstream Devices | |
| Console / Out-of-Band Access | |

### Firmware / OS
| Field | Value |
|---|---|
| Firmware Version | |
| Last Firmware Update | |
| Update Method | |

### Management
| Field | Value |
|---|---|
| Management IP | |
| Management Protocol (HTTPS / SSH / Serial) | |
| Allowed Management Source(s) | |

---

## Hypervisor Module
*Attach to: any device running a hypervisor role (Proxmox nodes).*

| Field | Value |
|---|---|
| Hypervisor Product / Version | |
| Cluster Membership | |
| Storage Pools (name, type, capacity, purpose) | |
| Network Bridges / vSwitches | |
| Shared Storage References (NFS, iSCSI, Ceph) | |
| Guest Inventory (link to separate list if long) | |
| High Availability Configuration | |
| Backup Integration (PBS datastore, credentials method) | |

---

## Service Module
*Attach one copy per distinct service hosted on this device. A DC running AD DS, DNS, and DHCP gets three Service modules.*

### Service Identity
| Field | Value |
|---|---|
| Service Name | |
| Service Type (AD DS / DNS / DHCP / SMB / SIEM / Backup / Web / etc.) | |
| Version | |
| Startup Type (Automatic / Manual / Disabled) | |

### Exposure
| Field | Value |
|---|---|
| Listening Ports (port/proto) | |
| Bind Interface(s) | |
| Exposed To (internal-only / specific zones / external) | |
| Authentication Method | |
| TLS / Certificate Status (issuer, expiration) | |

### Dependencies
| Field | Value |
|---|---|
| Depends On (upstream services this requires) | |
| Depended On By (downstream services that require this) | |
| Tolerable Downtime | |

### Logging & Monitoring
| Field | Value |
|---|---|
| Log Sources (channels, files, etc.) | |
| Log Destination (local / forwarded to SIEM) | |
| SIEM Integration (Yes / No) | |
| Alerts Configured | |

### Directory Accounts Referenced
*Service accounts and admin accounts used by this service that live in the directory (AD, local domain). List by name - authoritative record is in the directory account registry.*

| Account Name | Purpose | Privilege Level | Reference |
|---|---|---|---|
| | | | |

---

## Security (applies across the device)

### Hardening & Compliance
| Field | Value |
|---|---|
| Hardening Baseline Applied (reference) | |
| Compliance Mapping Reference (link to control-mapping artifact) | |
| Known Deviations from Baseline | |
| Last Vulnerability Scan | |

### Host Firewall & Exposure
| Field | Value |
|---|---|
| Host Firewall Enabled (Yes / No) | |
| Firewall Open Ports/Services (summary) | |

### Administrative Access
| Field | Value |
|---|---|
| Admin Access Method (direct / jump host / PAW) | |
| MFA Enforced for Admin Access (Yes / No / N/A) | |
| Allowed Admin Source(s) | |

---

## Change Control

| Field | Value |
|---|---|
| Last Change Date | |
| Last Change Reference | |
| Pending / Planned Changes | |
