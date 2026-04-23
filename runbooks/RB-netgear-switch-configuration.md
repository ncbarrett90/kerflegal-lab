# Runbook: Netgear Smart-Managed Switch Initial Configuration

## Purpose
Configure a Netgear ProSAFE smart-managed switch (GS7xxTPVx family and similar) from factory default to a hardened, VLAN-segmented baseline. Covers admin hardening, VLAN creation, per-port assignment, unused-port lockdown, and management IP move. Tested on firmware V6.0.10.x; menu paths vary slightly across firmware revisions.

The procedure is *additive* on the trunk that carries your current management session — VLAN 1 untagged passthrough is preserved until a separate cutover, so the session does not drop mid-runbook.

## Prerequisites
- Switch at factory default
- Firmware at the current vendor release
- Upstream router/firewall already configured for the target VLANs (subnets, DHCP if needed, gateway IPs)
- A port plan describing every port: device, access vs trunk, PVID, untagged/tagged VLAN membership, PoE, admin state
- Workstation on the same flat LAN as the switch's DHCP-assigned IP

## Procedure

### 1. First login
Browse to the switch's DHCP'd IP. Log in `admin` / `password`. The switch forces a password change on first login — set the longest password the platform allows (typically 20 chars), store in a password manager.

> Note: most Netgear smart-managed switches do not support additional local users. Multi-user auth requires TACACS+ or RADIUS.

### 2. Firmware check
Maintenance > Firmware. Confirm active image is at the current vendor release. Update and reboot if behind. The dual image bank holds the prior version as a rollback path — leave it in place unless you have a reason to promote the new firmware into both banks.

### 3. System identity
System > Management > System Information.
- System Name: `<hostname>`
- System Location: `<location>`
- System Contact: `<contact-or-blank>`

### 4. Management services
Security > Access.

- **HTTPS:** generate a self-signed certificate (Security > Access > HTTPS), enable HTTPS Admin Mode, log in via `https://<switch-ip>` to confirm, then disable HTTP. Replace the self-signed cert with a CA-issued one once a CA is available.
- **SSH:** enable. Provides a backup management path if HTTPS breaks.
- **Telnet:** if not exposed in the UI on your firmware, verify no listener: `nc -zv <switch-ip> 23` should return `Connection refused` or a silent timeout. Disable explicitly if exposed.

### 5. DNS resolver
System > Management > DNS Server. Set DNS Server 1 to `<dns-ip>`. If `<dns-ip>` is on the management VLAN that the switch can't reach yet, the value is still saved and becomes effective after step 12.

### 6. Time sync (SNTP)
System > Management > Time > SNTP. Enable SNTP, set Server 1 to `<ntp-ip>`, set timezone. Same reachability caveat as step 5.

Without working NTP, log timestamps and any cert validity checks are based on epoch and are unusable.

### 7. Create VLANs
Switching > VLAN > Advanced > VLAN Configuration. VLAN 1 (default) cannot be removed. VLAN 2 is Netgear-reserved on some models and cannot be removed; leave unused.

Add each VLAN required by the port plan:

| VLAN ID | Name |
|---|---|
| `<vlan-id>` | `<zone-name>` |
| `<vlan-id>` | `<zone-name>` |
| ... | ... |

### 8. Port configuration
Apply per-port settings from the port plan. Three Netgear-specific UI fields work together:

- **PVID** — VLAN tag applied to incoming untagged frames on this port
- **VLAN Member** — every VLAN this port participates in (tagged or untagged)
- **VLAN Tag** — subset of Member that is sent/accepted *tagged*

A VLAN that is in `Member` but not in `Tag` is untagged on the port (and should match the PVID for ingress consistency).

Two security knobs that default loose and need tightening:

- **Acceptable Frame Types** — defaults to `Admit All`. Set to `Untagged Only` on access ports so a malicious device can't VLAN-hop by sending tagged frames.
- **Ingress Filtering** — defaults to disabled. Enable everywhere so tagged frames for VLANs the port isn't a member of are dropped on ingress.

Suggested matrix layout for the port plan:

| Port | Role | PVID | Untagged Member | Tagged Member | Acceptable Frame Types | Ingress Filter | PoE | Admin |
|---|---|---|---|---|---|---|---|---|
| 1 | mgmt-trunk (staged) | 1 | 1 | `<vlan list>` | Admit All | on | – | up |
| 2 | trunk | `<pvid>` | `<pvid>` | `<vlan list>` | Admit All | on | as-needed | as-needed |
| ... | ... | ... | ... | ... | ... | ... | ... | ... |
| N | access | `<vlan-id>` | `<vlan-id>` | – | Untagged Only | on | – | down until cabled |

**Sequence:**

1. Configure access ports first (no live traffic at risk).
2. Configure trunk ports that are not currently carrying your management session.
3. Stage the trunk(s) in the live management path: add tagged membership for all required VLANs, but **leave** VLAN 1 as untagged member and PVID 1 alone. This lets your existing flat-LAN session survive.
4. Disable any port not in the plan: set Admin status to disabled. A disabled port refuses to come up on plug-in; a default-state port accepts a link and lands the device on VLAN 1.

### 9. Confirm config persistence
On Netgear smart-managed firmware, every page's `Save` button commits directly to startup config — there is no separate running-vs-startup save step. If `Maintenance > Save Config` (or `Maintenance > File Management > Save Configuration`) exists on your firmware, click it as a belt-and-suspenders confirmation.

### 10. Export and back up the config
Maintenance > Download > Configuration. Download as `<hostname>-config-initial-YYYY-MM-DD.cfg`. Store off-box.

### 11. Move management IP
**Session loss is expected at this step.** Do not proceed until step 9 confirms persistence and step 10 has produced a stored backup.

System > Management > Management Interface (or equivalent).

- Management VLAN: `<mgmt-vlan>`
- IP assignment: static
- IP address: `<mgmt-ip>`
- Subnet mask: `<mgmt-cidr>`
- Default gateway: `<mgmt-gateway>`

Apply. The session drops. The switch is now waiting for tagged `<mgmt-vlan>` traffic to reach it via the staged trunk; this happens once the upstream side completes its cutover.

## Validation

Steps 1 through 10 are validated inline as the procedure proceeds. Step 11 can only be validated after cutover.

- `admin` logs into the HTTPS UI with the new password; the default `password` is rejected
- HTTP and Telnet return refused or no response from the workstation
- VLAN list contains every VLAN from the plan (plus `1` and any vendor-reserved IDs)
- Port configuration matches the plan
- Unused ports are admin-down
- Backup config file is stored off-box

## Rollback

Four paths, in order of preference.

1. **Config restore via UI.** Maintenance > Upload, restore `<hostname>-config-initial-YYYY-MM-DD.cfg`. Use when a specific change needs to be backed out and the switch is still reachable.
2. **Boot to inactive firmware image.** Maintenance > Image Management, swap active image, reboot. Startup-config carries across. For firmware-related issues only. Inactive image may lag the active one.
3. **Factory reset via console.** Serial cable to the switch, or web UI if still reachable, reset to defaults. All admin and VLAN state cleared. Switch returns to DHCP on VLAN 1 with default credentials.
4. **Factory reset via physical button.** Hold reset for the duration specified in the hardware manual. Same outcome as path 3.

## Troubleshooting
*(Empty. Fill as issues arise.)*

## References
- Netgear ProSAFE smart-managed switch user manual (vendor support site, model-specific)
