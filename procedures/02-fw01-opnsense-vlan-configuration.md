# fw01 OPNsense VLAN Configuration

## Purpose
Add a single VLAN-tagged interface to an OPNsense firewall with IP addressing, optional DHCP, and optional Unbound listening. Additive only. No firewall rules are created by this procedure, zone policy lives in a separate procedure.

## Prerequisites
- OPNsense baseline hardened per 01-fw01-opnsense-initial-configuration
- Parameters decided before starting:
  - Parent physical interface (the device the VLAN rides on)
  - VLAN tag
  - Zone name (used as the OPNsense interface description)
  - IPv4 address and CIDR for fw01 in this zone
  - DHCP required (yes/no) and range if yes
  - Unbound listening required (yes/no)
- Admin account reachable via GUI

## Procedure

### 1. Pre-work backup
System > Configuration > Backups. Download as `config-pre-vlan-<zone>-YYYY-MM-DD.xml`. Store off-box.

### 2. Create the VLAN device
Interfaces > Other Types > VLAN. Click `+`.
- Parent: `<parent-interface>`
- VLAN tag: `<vlan-tag>`
- Description: `<zone>`

Save.

### 3. Assign the VLAN as an interface
Interfaces > Assignments. Select the new VLAN device from the "Available network ports" dropdown. Click `+`. It lands as `OPTn`. Save.

### 4. Configure the interface
Interfaces > [OPTn].
- Enable: checked
- Description: `<zone>`
- IPv4 Configuration Type: Static IPv4
- IPv4 address: `<ip>/<cidr>`
- IPv6 Configuration Type: None
- Block private networks: unchecked
- Block bogon networks: unchecked

Save. Apply Changes.

### 5. Configure DHCP (skip if not required)
Services > DHCPv4 > [zone].
- Enable: checked
- Range: `<dhcp-start>` to `<dhcp-end>`
- Gateway: `<fw01-ip-in-zone>`
- DNS servers: `<dns-target>` (fw01 if Unbound serves this zone, otherwise the zone's DNS provider)
- Domain name: `<domain-if-applicable>` or blank
- Default lease time / Max lease time: defaults unless a reason to change

Save.

### 6. Add to Unbound (skip if not required)
Services > Unbound DNS > General. Add the new zone to "Network Interfaces". Leave existing entries in place. Apply.

### 7. Post-work backup
System > Configuration > Backups. Download as `config-post-vlan-<zone>-YYYY-MM-DD.xml`. Store off-box.

## Validation

- Interfaces > Overview shows the new zone, status up, correct IP
- If DHCP was configured, Services > DHCPv4 > [zone] shows enabled with the expected range
- If Unbound listening was added, Services > Unbound DNS > General shows the zone in Network Interfaces
- No impact to existing management session
- Packet capture on the new interface (Interfaces > Diagnostics > Packet Capture) may show nothing if no switch port is tagging yet, this is expected

## Rollback

Additive only. Any of three paths reverts cleanly.

1. **GUI — History revert:** System > Configuration > History. Revert to the entry before step 2.
2. **GUI — Restore from XML:** System > Configuration > Backups > Restore. Upload `config-pre-vlan-<zone>-YYYY-MM-DD.xml`.
3. **Per-object cleanup:** Services > DHCPv4 > [zone], disable. Services > Unbound DNS, remove from Network Interfaces. Interfaces > Assignments, unassign the OPT. Interfaces > Other Types > VLAN, delete the VLAN device.

## Troubleshooting

**Symptom:** Kea fails to start on new zone: `DHCPSRV_OPEN_SOCKET_FAIL ... Address already in use`.
**Cause:** dnsmasq binds `*:67` wildcard. OPNsense 26.1 "Strict Interface Binding" does not release other interfaces.
**Fix:** Disable dnsmasq entirely (Services > Dnsmasq DNS & DHCP > uncheck Enable). Move any other-zone DHCP to Kea.
