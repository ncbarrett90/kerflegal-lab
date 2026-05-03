# PBS Node Baseline

## Purpose
Bring a freshly installed Proxmox Backup Server (pbs01) to the minimum state required to serve as a backup target for the cluster and pve04 per [ADR-0014](../decisions/ADR-0014-backup-architecture.md). Hardening beyond SSH and the local admin account is deferred to a post-build pass.

## Prerequisites
- PBS 4 installed on the NVMe with ext4 root, hostname set per [ADR-0008](../decisions/ADR-0008-device-naming-convention.md)
- 2x 1TB HDDs present and unpartitioned, intended as the ZFS mirror per [ADR-0013](../decisions/ADR-0013-cluster-storage-and-replication.md)
- Static IP planned per [network-design.md](../design/network-design.md) (10.0.10.10)
- sw01 port 8 configured as access PVID 10 per [network-design.md](../design/network-design.md)
- fw01 INFRA allow rules in place per [04-network-cutover](04-network-cutover.md) step 6 (ICMP, DNS, NTP, internet egress)
- Workstation SSH public key on hand and `ssh-copy-id` available
- Password manager and console access (KVM or physical)

## Procedure

Steps 1 and 2 run from the console. After step 2, SSH to the host as `root` from your workstation using the installer password. Step 3 replaces the installer root password with key auth, step 8 hardens sshd to refuse password auth entirely.

Build phase uses a typable password for `root` and `noah.barrett`. Step 11 rotates both to 20-character strong passwords as the final action before reboot.

### 1. Hostname check
```bash
hostnamectl
hostname --fqdn
```
Expect short hostname `pbs01`, FQDN `pbs01.infra.kerflegal.com`. Fix `/etc/hostname` and `/etc/hosts` and reboot if wrong.

### 2. Network configuration
**Run from console only.** Switch port 8 is access PVID 10, so pbs01 sits on VLAN 10 untagged. No VLAN-aware bridge is needed since PBS does not host VMs.

Edit `/etc/network/interfaces`. The NIC name is typically `nic0` on PBS 4 stable naming. Substitute from `ip -br link` if different.

```
auto nic0
iface nic0 inet static
    address 10.0.10.10/24
    gateway 10.0.10.1
    dns-nameservers 10.0.10.1
    dns-search infra.kerflegal.com
```

Apply with `ifreload -a`. Verify:
```bash
ip -br addr
ping -c 3 10.0.10.1
```
Expect the configured IP and replies from fw01.

### 3. Root SSH key
From your workstation:
```bash
ssh-copy-id root@10.0.10.10
ssh root@10.0.10.10
```
You should land in a shell with no password prompt. Stay in this session for the remainder.

### 4. Package repositories
Disable enterprise repos, enable no-subscription. PBS 4 / Debian 13 trixie ships APT sources in deb822 format.

```bash
echo 'Enabled: no' >> /etc/apt/sources.list.d/pbs-enterprise.sources

cat > /etc/apt/sources.list.d/pbs-no-subscription.sources <<'EOF'
Types: deb
URIs: http://download.proxmox.com/debian/pbs
Suites: trixie
Components: pbs-no-subscription
Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg
EOF
```

Verify with `apt update`. Expect fetches from `download.proxmox.com` and the Debian mirrors, no fetches from `enterprise.proxmox.com`.

### 5. System update
```bash
apt update
apt -y dist-upgrade
apt -y autoremove
```
Reboot if the kernel was upgraded.

### 6. Time synchronization
Point chrony at fw01 only.
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

### 7. Local admin account
Per [ADR-0009](../decisions/ADR-0009-account-naming-convention.md) amendment, plain `firstname.lastname` for non-domain hosts. PBS ships without `sudo`, install it before adding the user to the group.
```bash
apt install -y sudo
adduser --gecos "" --allow-bad-names noah.barrett
passwd noah.barrett
usermod -aG sudo noah.barrett
```
Set a typable build-phase password at the `passwd` prompt. Step 11 rotates this to a 20-character strong password.

```bash
passwd -S noah.barrett
```
First letter `P` means password set.

From your workstation, deposit your key on the new account:
```bash
ssh-copy-id noah.barrett@10.0.10.10
```

PBS user record and permissions. Unlike PVE, the PBS web UI does not expose the PAM realm for new user creation, only the internal `pbs` realm. Use the CLI to register the existing Linux account against the PAM realm and grant it Admin.
```bash
proxmox-backup-manager user create noah.barrett@pam
proxmox-backup-manager acl update / Admin --auth-id noah.barrett@pam
```
Log into the PBS web UI at `https://10.0.10.10:8007` as `noah.barrett@pam`. Expect full menus.

### 8. SSH hardening
```bash
grep -i '^Include' /etc/ssh/sshd_config
```
Expect `Include /etc/ssh/sshd_config.d/*.conf`.

```bash
cat > /etc/ssh/sshd_config.d/00-hardening.conf <<'EOF'
PermitRootLogin prohibit-password
PasswordAuthentication no
PubkeyAuthentication yes
PermitEmptyPasswords no
X11Forwarding no
EOF

/usr/sbin/sshd -t && systemctl reload ssh
```

Keep the existing root session open. From a second terminal, confirm key auth works for both `root` and `noah.barrett`, and password auth is refused.

### 9. ZFS mirror and datastore
Identify the two HDDs and confirm device and serial.
```bash
lsblk -o NAME,SIZE,MODEL,SERIAL,MOUNTPOINT
ls -l /dev/disk/by-id/ | grep -v part
```

Create the mirror pool. Name `bkpool` mirrors the `vmpool` pattern from [procedure 06](06-proxmox-node-baseline.md).
```bash
zpool create -o ashift=12 -O compression=lz4 -O atime=off bkpool mirror /dev/disk/by-id/<hdd1> /dev/disk/by-id/<hdd2>
zfs create bkpool/datastore
zpool status bkpool
```

Add the datastore in the PBS web UI:
1. Datastore > Add Datastore.
2. Name `ds01`. This is the name PVE nodes will reference when adding PBS as storage.
3. Backing Path `/bkpool/datastore`.
4. Save.

Confirm under Datastore > `<name>` > Summary that the datastore is online.

### 10. Disable subscription nag
The PBS web UI shows a "No valid subscription" dialog at every login on a no-subscription install. Reverts on `proxmox-widget-toolkit` upgrades and needs reapplying after each.
```bash
grep -n "data.status.toLowerCase" /usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js
```
If one or more matches return (PBS typically shows two):
```bash
sed -i.bak "s/res\.data\.status\.toLowerCase() !== 'active'/false/g" /usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js
systemctl restart proxmox-backup-proxy
```
Hard refresh the web UI and log back in. The dialog should not appear.

### 11. Rotate passwords
Replace the build-phase typable passwords with 20-character strong passwords for `root` and `noah.barrett`. Store both in the password manager.
```bash
passwd root
passwd noah.barrett
```

### 12. Reboot
```bash
systemctl reboot
```

## Validation
- `ssh root@10.0.10.10` and `ssh noah.barrett@10.0.10.10` succeed via key auth
- Password auth refused
- Web UI login at `https://10.0.10.10:8007` as `noah.barrett@pam` reaches full menus
- No subscription nag dialog
- `chronyc sources -v` shows `^*` on `10.0.10.1`
- `zpool status bkpool` reports `ONLINE` with both mirror members healthy
- Datastore visible under Datastore in the web UI
- Password manager has 20-character entries for `root` and `noah.barrett`, plus the SSH key

## Rollback
Reinstall from ISO. Fastest path for a fresh box with no production state.

## Troubleshooting
