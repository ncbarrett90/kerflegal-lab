# Architecture Decision Record (ADR) Index

> Single-page register of every architecture decision in the lab. Each row links to the full ADR. Use this page to answer "what has been decided" without opening individual files.
>
> **Process for a new decision:**
> 1. Copy `adr-template.md` to `ADR-NNNN-short-kebab-case-title.md` using the next number
> 2. Fill in the ADR
> 3. Add a row to the index below
> 4. Link the ADR from any affected device records, procedures, or other ADRs
>
> **Status values:**
> - **Proposed** - Drafted but not yet in force
> - **Accepted** - Current and in force
> - **Superseded** - Replaced by a newer ADR (linked in the Superseded By column)
> - **Deprecated** - No longer applies, not replaced

---

## Index

| # | Title | Status | Date | Superseded By |
|---|---|---|---|---|
| [0001](ADR-0001-three-tier-administrative-model.md) | Adopt Microsoft Three Tier Administrative Model | Accepted | 2026-04-14 | - |
| [0002](ADR-0002-per-tier-paw-model.md) | Per-Tier Privileged Access Workstation Model | Accepted | 2026-04-14 | - |
| [0003](ADR-0003-proxmox-ve-cluster.md) | Proxmox VE Cluster for Lab Virtualization | Accepted | 2026-04-15 | - |
| [0004](ADR-0004-pbs-tier0-classification.md) | Proxmox Backup Server Tier 0 Classification | Accepted | 2026-04-15 | - |
| [0005](ADR-0005-dedicated-wazuh-hardware.md) | Dedicated Hardware for Wazuh SIEM | Accepted | 2026-04-16 | - |
| [0006](ADR-0006-network-segmentation-and-ip-vlan-scheme.md) | Network Segmentation and IP/VLAN Scheme | Accepted | 2026-04-17 | - |
| [0007](ADR-0007-dns-architecture.md) | DNS Architecture | Superseded | 2026-04-18 | [0011](ADR-0011-infrastructure-dns-zone.md) |
| [0008](ADR-0008-device-naming-convention.md) | Device Naming Convention | Accepted | 2026-04-20 | - |
| [0009](ADR-0009-account-naming-convention.md) | Tiered Domain Account Naming Convention | Accepted | 2026-04-20 | - |
| [0010](ADR-0010-dedicated-paw-hypervisor.md) | Dedicated Hypervisor for Tier 0 and Tier 1 PAWs | Accepted | 2026-04-22 | - |
| [0011](ADR-0011-infrastructure-dns-zone.md) | Infrastructure DNS Zone | Accepted | 2026-04-22 | - |
| [0012](ADR-0012-no-dedicated-home-access-switch.md) | No Dedicated HOME Access Switch | Accepted | 2026-04-23 | - |
| [0013](ADR-0013-cluster-storage-and-replication.md) | Cluster Storage Layout and Replication | Accepted | 2026-04-28 | - |

---
