# Runbook: Add VLAN Interface to OPNsense

## Purpose
Add a single VLAN-tagged interface to an OPNsense firewall with IP addressing, optional DHCP, and optional Unbound listening. Additive only.

## Prerequisites
- OPNsense baseline hardened per RB-opnsense-initial-configuration
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
Interfaces > Devices, VLAN section. Click `+`.
- Parent interface: `<parent-interface>`
- VLAN tag: `<vlan-tag>`
- VLAN priority: leave default unless 802.1p marking is required
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

### 5. Configure DHCP via Kea (skip if not required)

If this is the first zone using Kea on this firewall, enable the service first.

Services > Kea DHCP > Kea DHCPv4 > General.
- Enabled: checked
- Interfaces: add `<zone>` (keep any existing entries)
- Firewall rules: checked (auto-allow DHCP on selected interfaces)
- Valid lifetime: default unless a reason to change

Apply.

Create the router option for this zone. Kea requires options to be defined as objects before they can be attached to a subnet, and router addresses differ per subnet, so each zone gets its own.

Services > Kea DHCP > Kea DHCPv4 > Options. Click `+`.
- Name: `<zone>-router`
- Code: `routers`
- Type: IPv4 address
- Value: `<fw01-ip-in-zone>`

Save.

Add the subnet.

Services > Kea DHCP > Kea DHCPv4 > Subnets. Click `+`.
- Subnet: `<network>/<cidr>` (the zone network, not fw01's host address)
- Pools: `<dhcp-start>-<dhcp-end>`
- Option data: attach the `<zone>-router` option created above
- DNS servers: `<dns-target>` (fw01 if Unbound serves this zone, otherwise the zone's DNS provider)
- DNS forward zone: `<domain>` (required, use `home.arpa` for non-lab zones per RFC 8375)

Save. Apply.

### 6. Add to Unbound (skip if not required)
Services > Unbound DNS > General. Add the new zone to "Network Interfaces". Leave existing entries in place. Apply.

### 7. Post-work backup
System > Configuration > Backups. Download as `config-post-vlan-<zone>-YYYY-MM-DD.xml`. Store off-box.

## Validation

- Interfaces > Overview shows the new zone, status up, correct IP
- If DHCP was configured, Services > Kea DHCP > Kea DHCPv4 > Subnets lists the zone subnet with the expected pool; Leases tab populates as clients request addresses
- If Unbound listening was added, Services > Unbound DNS > General shows the zone in Network Interfaces
- No impact to existing management session
- Packet capture on the new interface (Interfaces > Diagnostics > Packet Capture) may show nothing if no switch port is tagging yet, this is expected

## Rollback

Additive only. Any of three paths reverts cleanly.

1. **GUI — History revert:** System > Configuration > History. Revert to the entry before step 2.
2. **GUI — Restore from XML:** System > Configuration > Backups > Restore. Upload `config-pre-vlan-<zone>-YYYY-MM-DD.xml`.
3. **Per-object cleanup:** Reverse in this order.
   - Services > Kea DHCP > Kea DHCPv4 > Subnets, delete the zone subnet
   - Services > Kea DHCP > Kea DHCPv4 > Options, delete the `<zone>-router` option
   - Services > Kea DHCP > Kea DHCPv4 > General, remove the zone from Interfaces if no other subnets use it
   - Services > Unbound DNS, remove from Network Interfaces
   - Interfaces > Assignments, unassign the OPT
   - Interfaces > Devices, delete the VLAN device

## Troubleshooting
*(Empty. Fill as issues arise.)*