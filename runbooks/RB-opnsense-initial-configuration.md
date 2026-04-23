# Runbook: OPNsense Initial Configuration

## Purpose
Configure OPNsense from fresh install to a hardened single-LAN firewall. Default LAN allow-any rule is left in place. VLAN segmentation and zone firewall policy are handled in separate runbooks.

## Prerequisites
- OPNsense GUI reachable as `root`
- Console access to the firewall
- Password manager available for credential storage
- Authenticator app available on phone for TOTP enrollment
- Admin SSH keypair already generated for this environment; public key available to paste

## Procedure

### 1. Baseline backup
System > Configuration > Backups.
- Set Backup Count to `100` (default 60). More on-device history for rollback during and after this runbook.
- Download current config as `config-baseline-YYYY-MM-DD.xml`. Store off-box.

### 2. Apply pending updates
System > Firmware > Status. Check for updates, apply any available. Reboot if prompted.

### 3. System identity
System > Settings > General.
- Hostname: `<hostname>`
- Domain: `<domain>`
- Timezone: `<timezone>`

### 4. DNS via Unbound with DoT
System > Settings > General.
- Leave DNS server 1-4 blank. Firewall lookups will resolve via `127.0.0.1` → Unbound → DoT upstream.
- Uncheck "Allow DNS server list to be overridden by DHCP/PPP on WAN".

Services > Unbound DNS > General.
- Enable.
- Listen on interfaces that require Unbound resolution. For this baseline: LAN and Localhost.
- Enable DNSSEC.

Services > Unbound DNS > Query Forwarding. Leave empty — avoids catch-all collision with DoT.

Services > Unbound DNS > DNS over TLS. Add:
- Domain blank, Server IP `9.9.9.9`, Port `853`, Verify CN `dns.quad9.net`
- Domain blank, Server IP `149.112.112.112`, Port `853`, Verify CN `dns.quad9.net`

### 5. WAN hardening
Interfaces > WAN. Confirm "Block private networks" and "Block bogon networks" checked.

### 6. Web GUI hardening
System > Settings > Administration.
- HTTPS only
- Listen Interfaces: management-access interface(s) only — not WAN, not "All". For this baseline: LAN.
- TCP port: change from 443 to non-standard, store in password manager.
- Session timeout: 30 min

### 7. Primary admin account
System > Access > Users. Add:
- Username: `admin.username`
- Password: generated, 20+ chars, store in password manager
- Group: `admins`
- Shell: `/bin/sh`

Log out, log back in as `admin.username`. Confirm full GUI access.

### 8. Enable TOTP for primary admin
System > Access > Servers. Add a Local + Timebased One Time Password server.

System > Settings > Administration > Authentication. Enable both Local Database and the new TOTP server (both active during rollout prevents lockout).

Edit `admin.username`. Generate TOTP seed, store in password manager.

Log out, log back in as `admin.username` with password + TOTP code.

Once verified: Authentication section — remove Local Database, leaving only the TOTP server.

### 9. Root account (break-glass)
Edit `root`.
- Confirm password is strong, store in password manager.
- Do not generate a TOTP seed — this blocks root from the Step 8 GUI auth chain.

Root is break-glass only: physical console for config restore. GUI is gated by the missing TOTP seed.

### 10. SSH
System > Settings > Administration > Secure Shell.
- Enable
- Permit root login: off
- Permit password login: off (keys only)
- Listen on management-access interface(s) — not WAN. For this baseline: LAN.

System > Access > Users. Edit `admin.username`. Paste the admin public key into Authorized keys. Save.

Store the SSH key as a secure note in the password manager.

Test from workstation: `ssh admin.username@<lan-ip>` — expect key auth, no password prompt.

### 11. Verify DoT is in use
Interfaces > Diagnostics > Packet Capture on WAN. Run two captures while issuing test queries from a LAN client:
- Filter port `853`. Expected: traffic to/from `9.9.9.9` or `149.112.112.112`.
- Filter port `53`. Expected: no traffic.

### 12. Final backup
Download config as `config-post-initial-YYYY-MM-DD.xml`. Store off-box.

## Validation
- `admin.username` logs into GUI (new port) with password + TOTP code; reaches full menus
- `root` GUI login fails; console login works
- `nslookup google.com <lan-ip>` from workstation returns an answer
- Test queries show no port-53 traffic on WAN (853 only)
- `ssh admin.username@<lan-ip>` succeeds via key auth
- Password manager has entries for `admin.username`, `admin.username` SSH key, `root`

## Rollback
Three paths, in order of preference:

1. **GUI — History revert:** System > Configuration > History. Select a config version from before this runbook started. Revert. Firewall applies and may reboot.
2. **GUI — Restore from XML:** System > Configuration > Backups > Restore Configuration. Upload `config-baseline-YYYY-MM-DD.xml`. Apply. Firewall reboots.
3. **Console — boot-shell:** Physical console. Log in as `root`. Option 13 (Restore a backup) lists on-device history; pick a pre-runbook version. If on-device history is gone or inaccessible, USB the off-box XML and follow the [OPNsense config restore via boot shell](https://docs.opnsense.org/troubleshooting/config_reset.html) procedure.

**Rehearsal:** after the baseline backup, make a trivial change (host description) and revert via path 1. Untested paths are guesses — exercise path 3 at least once on new hardware.

## Troubleshooting
*(Empty. Fill as issues arise.)*
