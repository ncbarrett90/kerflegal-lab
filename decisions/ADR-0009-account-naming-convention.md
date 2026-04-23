# ADR-0009: Tiered Domain Account Naming Convention

| Field | Value |
|---|---|
| Status | Accepted |
| Date | 2026-04-20 |
| Supersedes | - |
| Superseded By | - |

## Context

Account naming needs to make administrative rights and purpose visible at first glance, in log output, and during audits. A flat `firstname.lastname` scheme for both regular users and admins gives no way to tell a compromised daily-driver account from a compromised Tier 0 credential in a log line.

## Decision

Accounts follow a prefix-based convention that makes type and tier readable from the account name.

| Type | Pattern | Examples |
|---|---|---|
| Regular user | `firstname.lastname` | `noah.barrett` |
| Tiered admin | `t<N>-firstname.lastname` | `t0-noah.barrett`, `t1-noah.barrett` |
| Service account | `svc-servicename` | `svc-wazuh`, `svc-pbs-backup` |
| gMSA | `gmsa-servicename$` | `gmsa-dhcp$`, `gmsa-sql$` |
| Break-glass | `bg-t<N>-<role>` | `bg-t0-domainadmin`, `bg-t1-serveradmin`, `bg-t2-workstationadmin` |

One break-glass account per tier. All admin accounts use the `@corp.kerflegal.com` suffix and never authenticate against anything outside the internal domain. The tier prefix on admin accounts makes credential misuse visible in logs: a `t0-` account authenticating to a Tier 2 system is an immediate detection signal.

## Alternatives Considered

- **First initial plus last name (e.g. `nbarrett`)** - Common in production. Collides easily in larger directories and carries no role or tier information. Rejected.
- **Company-code-prefixed usernames** - Seen at MSP-managed clients. Without the master mapping list the prefix carries no context, which is worse than no prefix. Rejected.
- **Flat `firstname.lastname` for admins too** - Cannot distinguish tier or administrative rights from the name. Rejected.

## Consequences

- Log analysis and audit work become easier; tier and account type are readable at a glance.
- Administrators hold multiple accounts (regular plus one admin account per tier they operate in). Operational overhead is intentional.
- **Deviation:** Production environments I have worked in commonly use a first-initial-plus-last-name format, or embed a company code. Both are in active use and both are harder to work with than the convention adopted here.

## Amendment 2026-04-22: Local accounts on non-domain-joined devices

ADR-0009 governs domain accounts only. Local accounts on non-domain-joined infrastructure (fw01, Proxmox hosts, PBS, Wazuh, switches, AP, PAWs) use plain `firstname.lastname` for admin accounts and `bg-<device>` for per-device break-glass. Tier context for these devices is carried in the device registry, not in the account name. This keeps local and domain credentials visually distinct in aggregated logs.
