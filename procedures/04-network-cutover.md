# network cutover

## Purpose
Retire the legacy flat LAN now that the workstation has been cut over to HOME. Removes the fw01 LAN interface + allow-any rule, the sw01 port-12 transit, and the VLAN 1 untagged membership on sw01 port 1 (fw01 trunk). After this procedure, sw01 port 1 is tagged-only and no host reaches fw01 via VLAN 1.

Per-device onboarding (pve01–04, pbs01, ap01, wazuh01) runs as separate procedures that follow.

## Prerequisites
- Workstation on HOME (VLAN 100, static 10.0.100.33) via sw01 port 3, reaching fw01 GUI, sw01 GUI, DNS, and internet
- fw01 VLAN interfaces live for all 7 zones (per `02-fw01-opnsense-vlan-configuration`)
- Kea DHCPv4 serving HOME, Unbound listening on INFRA + HOME
- HOME firewall rules live: admin → fw01 (scoped to workstation alias), DNS → fw01:53, egress → !rfc1918
- sw01 management IP on 10.0.10.15 (INFRA), port 1 staged with VLAN 1 untagged + PVID 1
- sw01 port 12 currently enabled as workstation cutover transit

## Procedure

### 1. Pre-work backups
- fw01: System > Configuration > Backups, download as `config-pre-network-cutover-YYYY-MM-DD.xml`
- sw01: Maintenance > Download > Configuration, download as `sw01-config-pre-network-cutover-YYYY-MM-DD.cfg`

### 2. Retire fw01 LAN interface
Firewall > Rules > LAN. Delete the default allow-any rule and any other LAN-specific rules. Apply.

Interfaces > Assignments. Unassign the LAN interface (the physical/parent port, not a VLAN). Apply.

### 3. Disable the sw01 transit port
Switching > Ports. Port 12 → Admin status: Disabled. Save.

### 4. Strip VLAN 1 untagged from sw01 port 1
Switching > VLAN > Port Membership, port 1:
- Remove VLAN 1 from untagged membership
- PVID: set to the planned trunk PVID
- Acceptable Frame Types: Tagged Only
- Ingress Filtering: enabled

Save.

### 5. Mgmt-plane hardening
Now that INFRA is the canonical mgmt zone, scope services down:

- System > Settings > Administration > Listen Interfaces: **INFRA**
- System > Settings > Administration > Secure Shell > Listen Interfaces: **INFRA**
- Services > Unbound DNS > General > Network Interfaces: remove LAN, keep INFRA + HOME + Localhost

Local Database stays in the auth chain throughout — do not remove it.

### 6. Post-work backups
- fw01: `config-post-network-cutover-YYYY-MM-DD.xml`
- sw01: `sw01-config-post-network-cutover-YYYY-MM-DD.cfg`

## Validation
- Workstation still reaches fw01 GUI, sw01 GUI, DNS, and internet from HOME
- sw01 port 1 shows no VLAN 1 membership and tagged-only on the planned VLANs
- A device plugged into an uncon­figured default port (VLAN 1 land) cannot reach fw01

## Rollback
- **fw01:** System > Configuration > History, revert to pre-cutover entry. Fallback: restore `config-pre-network-cutover-YYYY-MM-DD.xml`.
- **sw01:** Maintenance > Upload, restore `sw01-config-pre-network-cutover-YYYY-MM-DD.cfg`. Re-enable port 12 if transit is needed again.

## Troubleshooting

**Symptom:** Post-LAN-deletion, workstation loses fw01 GUI/SSH but internet and sw01 admin still work.
**Cause:** Admin rule destination was `LAN address` or a LAN-tied alias. Deletion invalidated the token; OPNsense silently deactivated the rule.
**Fix:** Before deletion, set destination to `INFRA address` (symbolic token, survives interface changes). If already locked out, see `01-fw01-opnsense-initial-configuration` troubleshooting.

**Symptom:** GUI and SSH unreachable from every interface after LAN deletion.
**Cause:** Listen Interfaces scoped to LAN only. With LAN gone, daemons bind to nothing.
**Fix:** Leave Listen blank during cutover. Step 5 tightens to INFRA at the safe end point.
