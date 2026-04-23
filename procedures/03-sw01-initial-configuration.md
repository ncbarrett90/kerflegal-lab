# sw01 Initial Configuration

## Purpose
Configure sw01 (Netgear GS728TPV2) from factory default to the target state defined in `network-design.md`. Admin hardening, VLAN creation, port plan applied, unused ports disabled, management interface moved to INFRA. All work is staged so current management access via flat LAN is preserved until the final step. Session loss is expected at the end of the procedure and is not recovered until network cutover runs.

## Prerequisites
- sw01 at factory default
- Firmware V6.0.10.26 (or current)
- 02-fw01-opnsense-vlan-configuration run for all required zones
- `network-design.md` section 4 (sw01 port plan) on hand

## Procedure

### 1. First login
Browse to DHCP'd IP. Log in `admin`/`password`. Forced password change on first login → set 20-char (platform max), store in pw manager.

### 2. Firmware check
Maintenance > Firmware. Confirm active image is V6.0.10.26 (or current). Update + reboot if behind. Image 2 holds prior version as rollback.

### 3. System identity
System > Management > System Information.
- System Name: `sw01`
- System Location: `<location>`
- System Contact: `<contact-or-blank>`

Save.

### 4. Management services
Security > Access.

- HTTPS: generate self-signed cert (interim until CA is up), enable, test login from workstation, then disable HTTP
- SSH: enabled (lockout backup)
- Telnet: not exposed in UI on V6.0.10.26; `nc -zv <sw01-ip> 23` returns refused or silent timeout

### 5. DNS resolver
System > Management > DNS Server. DNS Server 1: `10.0.10.1` (fw01 Unbound). Unreachable until cutover; persists through the IP move in step 11.

### 6. Time sync (SNTP)
System > Management > Time > SNTP. Enabled, server `10.0.10.1`. Unreachable until cutover.

### 7. Create VLANs
Switching > VLAN > Advanced > VLAN Configuration. VLAN 1 (default) and VLAN 2 (Netgear-reserved) cannot be removed; VLAN 2 is unused.

| VLAN ID | Name |
|---|---|
| 10 | INFRA |
| 20 | SECURITY |
| 30 | PAW |
| 40 | IDENTITY |
| 50 | SERVERS |
| 60 | CLIENTS |
| 100 | HOME |

### 8. Port configuration

Apply per-port settings per the matrix below. Configure in this order so the live mgmt session survives:

1. **Access ports** (no live traffic): 8, 9, 24
2. **Non-mgmt trunks**: 2, 4, 5, 6, 7
3. **Staged ports** (carry current flat-LAN session — keep VLAN 1 untagged + PVID 1 in place; cutover procedure retires them): 1 (fw01 trunk), 3 (workstation access)
4. **Disable unused**: 10–23

| Port | Device | PVID | Untagged Member | Tagged Member | Acceptable Frame Types | Ingress Filter | PoE | Admin |
|---|---|---|---|---|---|---|---|---|
| 1 | fw01 uplink (staged) | 1 | 1 | 10, 20, 30, 40, 50, 60, 100 | Admit All | on | – | up |
| 2 | ap01 | 10 | 10 | 100 | Admit All | on | on | down (until cabled) |
| 3 | workstation HOME (staged) | 1 | 1 | – | Admit All | on | – | up |
| 4 | pve01 | 10 | 10 | 30, 40, 50, 60 | Admit All | on | – | down |
| 5 | pve02 | 10 | 10 | 30, 40, 50, 60 | Admit All | on | – | down |
| 6 | pve03 | 10 | 10 | 30, 40, 50, 60 | Admit All | on | – | down |
| 7 | pve04 | 10 | 10 | 30 | Admit All | on | – | down |
| 8 | pbs01 | 10 | 10 | – | Untagged Only | on | – | down (until cabled) |
| 9 | wazuh01 | 20 | 20 | – | Untagged Only | on | – | down |
| 10–23 | unused | – | – | – | – | – | – | down |
| 24 | break-glass mgmt | 10 | 10 | – | Untagged Only | on | – | down |

After staging ports 1 and 3, confirm management access still works before continuing.

### 9. Config persistence
GS728TPV2 web-UI page Saves commit directly to startup — no separate running/startup distinction. Click `Maintenance > Save Config` if your firmware exposes it; otherwise nothing to do.

### 10. Export and back up the config
Maintenance > Download > Configuration. Download the text config file as `sw01-config-initial-YYYY-MM-DD.cfg` (or whatever extension the switch offers). Store off-box.

### 11. Move management IP to VLAN 10
**This is the point where session loss is expected.** Do not proceed unless steps 9 and 10 are complete.

System > Management > Management Interface, or equivalent.

- Management VLAN: 10
- IP assignment: static
- IP address: 10.0.10.15
- Subnet mask: 255.255.255.0
- Default gateway: 10.0.10.1

Apply.

Session drops. Switch is now waiting for VLAN 10 tagged traffic to reach it via the fw01 trunk, which happens during network cutover.

## Validation

Steps 1 through 10 are validated as the procedure proceeds. Step 11 cannot be validated until after cutover. The post-cutover checks are captured in the cutover procedure, not here.

Local-to-this-procedure validation checks:
- `admin` can log into the HTTPS UI with the new password; default `password` is rejected
- HTTP and Telnet return connection refused or no response from the workstation
- VLAN list shows 1, 2, 10, 20, 30, 40, 50, 60, 100
- Port configuration matches the port plan in `network-design.md` section 4
- Unused ports 10 through 23 are admin-down
- `sw01-config-initial-YYYY-MM-DD.cfg` is stored off-box

## Rollback

Four paths, in order of preference.

1. **Config restore via UI.** If still reachable, upload `sw01-config-initial-YYYY-MM-DD.cfg` via Maintenance > Upload. Returns the switch to the post-staging state. Use when a specific config change needs to be backed out.

2. **Boot to inactive firmware image.** Maintenance > Image Management, swap active image, reboot. Startup-config carries across. For firmware-related issues only. Inactive image may lag if not refreshed.

3. **Factory reset via console.** Serial cable to the switch, or web UI if still reachable, reset to factory defaults. All admin and VLAN state is cleared. Switch returns to DHCP on VLAN 1 with default credentials.

4. **Factory reset via physical button.** Hold the reset button for the duration specified in the GS728TPV2 hardware manual. Same outcome as path 3.

## Troubleshooting
*(Empty. Fill as issues arise.)*

## References
- `network-design.md` section 4 (sw01 port plan)
- ADR-0006 (network segmentation)
- GS728TPV2 user manual (Netgear support site, current firmware version)