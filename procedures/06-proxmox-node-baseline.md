# Proxmox Node Baseline

## Purpose
Bring a freshly installed Proxmox VE host (pve01-04) to the minimum state required to join a cluster and host VMs. Hardening beyond SSH and the local admin account is deferred to a post-build pass after the cluster and AD lab are standing.

## Prerequisites
- PVE 9 installed on NVMe with hostname set per [ADR-0008](../decisions/ADR-0008-device-naming-convention.md)
- Static IP planned per [network-design.md](../design/network-design.md) (10.0.10.5-8)
- Switch port configured as a tagged-only trunk per [ADR-0015](../decisions/ADR-0015-hypervisor-trunk-tagging-policy.md)
- fw01 INFRA allow rules in place per `04-network-cutover` step 6 (ICMP, DNS, NTP, internet egress)
- Workstation SSH public key on hand and `ssh-copy-id` available
- Password manager and console access (IPMI/KVM or physical)

## If PVE was installed previously

Skip this section on hardware that has not run PVE before.

If the install hardware previously ran Proxmox, the installer detects the existing `pve` LVM volume group and prompts to either cancel or rename it. Choosing rename lets the install proceed and tags the leftover VG as `pve-OLD-<hash>`. The leftover VG must be torn down before step 9, otherwise `zpool create` fails on the secondary SSD.

Do this once after first boot, before starting step 1.

### Identify the leftover VG

```bash
lsblk
vgs
pvs
```
Expect a second VG named `pve-OLD-<hash>` and a `pvs` line showing which partition backs it (commonly `/dev/sda3` on the secondary SSD).

### Remove the leftover VG and wipe the disk

Substitute the actual VG name and PV path from the previous step. LVM names are case sensitive.

```bash
mount | grep pve-OLD
```
Expect no output. If anything prints, stop and investigate before continuing.

```bash
vgchange -an pve-OLD-<hash>
vgremove pve-OLD-<hash>
pvremove /dev/sda3
wipefs -a /dev/sda1 /dev/sda2 /dev/sda3
sgdisk --zap-all /dev/sda
wipefs -a /dev/sda
```

### Verify

```bash
lsblk
vgs
```
Expected, the secondary SSD shows no child partitions, `vgs` shows only `pve`.

## Procedure

### 1. Hostname check
```bash
hostnamectl
hostname --fqdn
```
Expect short hostname `pveNN`, FQDN `pveNN.infra.kerflegal.com`. Fix `/etc/hostname` and `/etc/hosts` and reboot if wrong.

### 2. Network configuration
**Run from console only.** `ifreload -a` will drop the management session on a new IP. Every step after this can be done from SSH.

VLAN-aware bridge with management tagged on VLAN 10 per [ADR-0015](../decisions/ADR-0015-hypervisor-trunk-tagging-policy.md). PVE 9 uses stable NIC naming, the bridge member is typically `nic0`. Substitute the actual name from `ip -br link` if different.

Edit `/etc/network/interfaces`, replacing the installer-generated `vmbr0` block:
```
auto nic0
iface nic0 inet manual

auto vmbr0
iface vmbr0 inet manual
    bridge-ports nic0
    bridge-stp off
    bridge-fd 0
    bridge-vlan-aware yes
    bridge-vids 2-4094

auto vmbr0.10
iface vmbr0.10 inet static
    address 10.0.10.<host-octet>/24
    gateway 10.0.10.1
    dns-nameservers 10.0.10.1
    dns-search infra.kerflegal.com
```
Apply with `ifreload -a`. Verify:
```bash
ip -br addr
ping -c 3 10.0.10.1
```
Expect `vmbr0.10` carrying the configured IP and ping replies from fw01.

### 3. Package repositories
PVE 9 / Debian 13 trixie ships APT sources in deb822 format (`.sources` files). Disable enterprise repos, enable no-subscription.
```bash
echo 'Enabled: no' >> /etc/apt/sources.list.d/pve-enterprise.sources
echo 'Enabled: no' >> /etc/apt/sources.list.d/ceph.sources

cat > /etc/apt/sources.list.d/pve-no-subscription.sources <<'EOF'
Types: deb
URIs: http://download.proxmox.com/debian/pve
Suites: trixie
Components: pve-no-subscription
Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg
EOF
```
Leave `debian.sources` enabled, that is the base Debian repo.

Verify with `apt update`. Expect fetches from `download.proxmox.com` and the Debian mirrors, no fetches from `enterprise.proxmox.com` or the ceph URI.

**CIS:** Package repositories and source trust.

### 4. System update
```bash
apt update
apt -y dist-upgrade
apt -y autoremove
```
Reboot if the kernel was upgraded.

### 5. Time synchronization
Point chrony at fw01 only. Non-optional for cluster operation.
```bash
cat > /etc/chrony/chrony.conf <<'EOF'
server 10.0.10.1 iburst
driftfile /var/lib/chrony/chrony.drift
makestep 1.0 3
rtcsync
logdir /var/log/chrony
EOF

systemctl restart chrony
chronyc sources -v
```
Expect a `^*` line on `10.0.10.1` within a minute.

**CIS:** Time synchronization.

### 6. Root SSH key
Add the workstation public key for root before tightening sshd. The PVE installer leaves password auth enabled for root, so `ssh-copy-id` is the cleanest path.

From your workstation:
```bash
ssh-copy-id root@10.0.10.<host-octet>
```

Confirm key auth works before continuing:
```bash
ssh root@10.0.10.<host-octet>
```
You should land in a shell with no password prompt.

### 7. Local admin account
Per [ADR-0009](../decisions/ADR-0009-account-naming-convention.md) amendment, plain `firstname.lastname` for non-domain hosts. The `--allow-bad-names` flag bypasses Debian's default `NAME_REGEX` rejection of the period in the username.
```bash
adduser --gecos "" --allow-bad-names noah.barrett
passwd noah.barrett
usermod -aG sudo noah.barrett
```
Set a strong password at the `passwd` prompt and store it in the password manager. Setting it explicitly avoids the case where the `adduser` interactive password prompt is skipped.

Verify the password works locally before moving to SSH:
```bash
su - noah.barrett
```
Should accept the password and drop you into noah.barrett's shell.

From your workstation, deposit your key on the new account. The `noah.barrett@` prefix is required, otherwise SSH falls back to your workstation's local username:
```bash
ssh-copy-id noah.barrett@10.0.10.<host-octet>
```

PVE web UI permissions, logged in as `root@pam`:
1. Datacenter > Permissions > Groups, create `Administrators`.
2. Datacenter > Permissions > Users, Add. User name `noah.barrett`, Realm `Linux PAM standard authentication`.
3. Datacenter > Permissions, Add > Group Permission. Path `/`, Group `Administrators`, Role `Administrator`.
4. Edit `noah.barrett`, add to group `Administrators`.
5. Log out, log back in as `noah.barrett@pam`. Expect full menus, not a stripped-down view.

**CIS:** Local user accounts and sudo.

### 8. SSH hardening
Drop a hardening file in `/etc/ssh/sshd_config.d/`. PVE 9 / Debian 13 sources that directory from the main config, isolating the changes from package upgrades.

Confirm the include line is present:
```bash
grep -i '^Include' /etc/ssh/sshd_config
```
Expect `Include /etc/ssh/sshd_config.d/*.conf`.

Write the hardening file. As `noah.barrett`, use `sudo tee` (a plain `sudo cat >` will fail because the shell redirection runs as your user before `sudo` gets involved). As `root`, the heredoc and `cat >` work directly.
```bash
sudo tee /etc/ssh/sshd_config.d/00-hardening.conf > /dev/null <<'EOF'
PermitRootLogin prohibit-password
PasswordAuthentication no
PubkeyAuthentication yes
PermitEmptyPasswords no
X11Forwarding no
EOF
```

Validate and reload. The `sshd` binary lives in `/usr/sbin/`, which is not on a regular user's `$PATH`, so the absolute path is required under `sudo`.
```bash
sudo /usr/sbin/sshd -t && sudo systemctl reload ssh
```

Keep the existing session open. From a second terminal, confirm key auth works for both `root` and `noah.barrett`, and password auth is refused.

**CIS:** SSH server configuration.

### 9. ZFS pool on secondary SSD
Cap the ARC. Default is ~50% of host RAM, which competes with VMs. 4 GiB is generous for lab workloads and uniform across pve01-04.
```bash
sudo tee /etc/modprobe.d/zfs.conf > /dev/null <<'EOF'
options zfs zfs_arc_max=4294967296
EOF

sudo update-initramfs -u -k all
echo 4294967296 | sudo tee /sys/module/zfs/parameters/zfs_arc_max > /dev/null
```

Identify the secondary SSD. Verify device and serial before continuing.
```bash
lsblk -o NAME,SIZE,MODEL,SERIAL,MOUNTPOINT
```

Find the stable `/dev/disk/by-id/` symlink for that device. Use the `ata-` or `nvme-` name (model + serial), not the `wwn-` form.
```bash
ls -l /dev/disk/by-id/ | grep -v part
```

Pool name `vmpool` is uniform across pve01-04 per [ADR-0013](../decisions/ADR-0013-cluster-storage-and-replication.md), which is what lets replication target peers by storage ID.
```bash
sudo zpool create -o ashift=12 -O compression=lz4 -O atime=off vmpool /dev/disk/by-id/<ssd-id>
sudo zpool status vmpool
```
Cluster-level storage registration happens at cluster formation, not here.

### 10. Disable subscription nag
The PVE web UI shows a "No valid subscription" dialog at every login on a no-subscription install. Patch the toolkit JS to suppress it. Reverts on `proxmox-widget-toolkit` upgrades and needs reapplying after each.

The check lives in `/usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js`. As of PVE 9 it reads `res.data.status.toLowerCase() !== 'active'`. Confirm the line is present in your version before patching:
```bash
grep -n "data.status.toLowerCase" /usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js
```
If it returns a single line, apply the patch:
```bash
sudo sed -i.bak "s/res\.data\.status\.toLowerCase() !== 'active'/false/g" /usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js
sudo systemctl restart pveproxy
```
Hard refresh (Ctrl+Shift+R) the web UI tab and log back in. The dialog should not appear.

### 11. Reboot
```bash
sudo systemctl reboot
```

## Validation
- `ssh root@10.0.10.<host-octet>` and `ssh noah.barrett@10.0.10.<host-octet>` succeed via key auth
- Password auth refused
- Web UI login as `noah.barrett@pam` reaches full menus
- No subscription nag dialog at web UI login
- `chronyc sources -v` shows `^*` on `10.0.10.1`
- `zpool status vmpool` reports `ONLINE`
- `cat /sys/module/zfs/parameters/zfs_arc_max` returns `4294967296`
- Password manager has entries for root, `noah.barrett`, and the SSH key

## Rollback
- **Network:** console in, fix `/etc/network/interfaces` by hand, `ifreload -a`.
- **SSH:** `apt install --reinstall openssh-server`, reload.
- **Repos:** uncomment `pve-enterprise.list`, delete `pve-no-subscription.list`, `apt update`.
- **ZFS pool:** `zpool destroy vmpool` (destructive, only if empty).
- **PVE users:** Datacenter > Permissions, delete.
- **Full reset:** reinstall from ISO. Usually faster than unwinding on a fresh node.

## Troubleshooting

**Symptom:** Lost network access after step 2.
**Cause:** Wrong NIC name, or switch port not tagging VLAN 10.
**Fix:** Console in, fix `/etc/network/interfaces`, `ifreload -a`. Verify NIC with `ip -br link` and the switch port config per [ADR-0015](../decisions/ADR-0015-hypervisor-trunk-tagging-policy.md).

**Symptom:** `chronyc sources` shows `^?` on 10.0.10.1.
**Cause:** Missing fw01 rule INFRA → fw01:123/udp, or chrony on fw01 not listening on INFRA.
**Fix:** Add the rule, confirm Services > NTP is listening on INFRA.

**Symptom:** `ssh-copy-id` returns `Permission denied` for `noah.barrett`.
**Cause:** Username defaulted to workstation login (sshd log shows `invalid user noah` instead of `noah.barrett`), password not set on the account, or `ssh-agent` is offering keys that get rejected before the password prompt fires.
**Fix:** Watch sshd events with `journalctl -u ssh -f` and reattempt. Debian 13 uses journald, `/var/log/auth.log` is not present by default. The log line tells you which case applies. For username default, use the explicit `noah.barrett@` prefix. For password issues, run `passwd -S noah.barrett` on the host to confirm `P` (password set), then `su - noah.barrett` to verify locally. For agent interference, force password auth with `ssh-copy-id -o PreferredAuthentications=password noah.barrett@10.0.10.<host-octet>`.

## Deferred to hardening pass
- TOTP on web UI for `root@pam` and `noah.barrett@pam`
- Datacenter and node firewall enabled with input policy DROP
- Bind PVE web UI listener to the INFRA IP only
- Log forwarding to wazuh01
- CIS Benchmark mapping reconciled to `security/cis-control-mapping.ods`
