# ap01 Initial Configuration

## Purpose
Adopt and configure the Ubiquiti UniFi AC Pro (ap01) as a single-SSID HOME radio with management on INFRA. AP comes up at its planned IP via a Kea reservation.

## Prerequisites
- 04-network-cutover complete
- AP cabled to sw01 port 2 (PVID 10 untagged, tagged 100, PoE on)
- AP MAC noted (from the unit's label)
- Ubuntu workstation with KVM/QEMU
- Windows guest VM with UniFi Network Application installed and running
- Workstation cable available to move to sw01 port 24

## Procedure

### 1. Stage sw01 port 24 and move workstation to INFRA
Switching > VLAN, port 24:
- PVID `10`
- VLAN member `10` untagged
- Acceptable Frame Types `Untagged Only`
- Ingress Filter on
- Admin enabled

Move workstation cable from sw01 port 3 to port 24. Static-assign workstation NIC to `10.0.10.50/24`, gateway `10.0.10.1`, DNS `10.0.10.1`. Set the macvtap-bridged VM to `10.0.10.40/24` via Kea reservation or Windows static.

### 2. Add INFRA subnet and AP reservation in Kea
Services > Kea DHCP > General > Interfaces: add `INFRA` alongside `HOME`.

Services > Kea DHCP > Subnets > Add:
- Subnet `10.0.10.0/24`
- No pool (reservations only)
- Option `routers` `10.0.10.1`
- Option `domain-name-servers` `10.0.10.1`
- Option `ntp-servers` `10.0.10.1`

Reservations > Add:
- Identifier: ap01 MAC
- IP `10.0.10.20`
- Hostname `ap01`

Apply.

### 3. Factory reset the AP
Power on, hold reset at least 5 seconds. LED settles solid white.

### 4. Confirm the lease
fw01 > Services > Kea DHCP > Leases. ap01 MAC at `10.0.10.20`.

### 5. Adopt
Devices > "Pending Adoption" > Adopt.

### 6. Create the HOME network
Settings > Networks > Add Network:
- Name `HOME`
- Network Type `VLAN-Only`
- VLAN ID `100`

### 7. Create the SSID
Settings > WiFi > Add WiFi Network:
- Name `HOME`
- Security WPA2/WPA3 mixed (or pure WPA3 if all clients support it)
- Passphrase 16+ chars, store in password manager
- Network `HOME`
- PMF Required
- Hide SSID on
- Fast roaming off

### 8. Configure radios
Settings > WiFi:
- Default WiFi Speeds `Maximum`
- Meshing off
- Autolink off

Devices > ap01 > Config > Radios:
- 5 GHz channel width `80 MHz`

### 9. System settings
Settings > System:
- Time zone local
- NTP server `10.0.10.1`
- Auto firmware updates on

### 10. Backup
- Controller: Settings > System > Backup, download `.unf` off-box
- sw01: Maintenance > Download > Configuration, save as `sw01-config-post-ap01-adopt-YYYY-MM-DD.cfg`

## Validation
- `ping 10.0.10.20` from an SSH session on fw01 succeeds
- AP shows online in the controller
- HOME SSID visible from a wireless client
- Wireless client pulls a Kea lease on `10.0.100.0/24`, resolves DNS, reaches internet

## Rollback
- AP misconfig: factory reset, restart from step 3
- Controller corruption: fresh install, restore `.unf` backup
- Bad Kea reservation: edit or delete in Services > Kea DHCP > Reservations

## Troubleshooting

**Symptom:** AP boots but no DHCP lease.
**Cause:** Reservation MAC mismatch, Kea not bound to INFRA, or sw01 port 2 misconfigured.
**Fix:** Verify MAC, confirm INFRA in Kea Interfaces, check port 2 against the port plan.

**Symptom:** AP has a lease but does not appear in the controller.
**Cause:** Controller VM not on the AP's L2 segment.
**Fix:** Confirm controller VM is on `10.0.10.0/24`. If discovery still misbehaves, L3-Adopt: `ssh ubnt@10.0.10.20` (default `ubnt`), then `set-inform http://<controller-ip>:8080/inform`. Re-run after clicking Adopt.

**Symptom:** Windows controller VM not seen by the AP after the workstation moves to INFRA.
**Cause:** macvtap NIC reconnects to new L2 but Windows may keep its old DHCP lease. Network profile may revert to Public, blocking UDP 10001 and TCP 8080.
**Fix:** Release/renew or disable/re-enable the NIC. Confirm INFRA IP applied. Set network profile to Private.

**Symptom:** Networks editor warns "needs to be configured on your gateway".
**Cause:** Network Type is Standard but no UniFi gateway is adopted.
**Fix:** Change Network Type to VLAN-Only, specify VLAN ID only.

## References
- `network-design.md` section 4 (sw01 port plan)
- `02-fw01-opnsense-vlan-configuration`
- ADR-0006
