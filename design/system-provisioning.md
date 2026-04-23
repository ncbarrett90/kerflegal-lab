# System Provisioning

Physical hardware inventory.

## 1. Firewall

### fw01 (SilverStone Desktop)

Intel Core i3 10th Gen
8GB RAM
500GB SSD
4-port NIC

OPNsense firewall. Gateway for all lab zones. WAN uplink to ISP modem.

## 2. Proxmox Cluster

Three-node cluster with asymmetric RAM, constrained by the available DDR4 inventory of 9x 8GB and 6x 4GB modules.

### pve01 (Dell Optiplex 5070)

Intel Core i7 8th Gen
32GB RAM (4x 8GB)
512GB NVMe
480GB SSD

Proxmox hypervisor. Cluster primary. Strongest CPU in cluster.

### pve02 (Dell Optiplex 5050)

Intel Core i5 7th Gen
32GB RAM (4x 8GB)
500GB NVMe
500GB SSD

Proxmox hypervisor. Cluster node.

### pve03 (Dell Optiplex 5050)

Intel Core i5 7th Gen
16GB RAM (1x 8GB + 2x 4GB)
512GB NVMe
500GB SSD

Proxmox hypervisor. Cluster node. Upgrade to 32GB pending 3x 8GB DDR4 modules.

## 3. PAW Hypervisor

### pve04 (Intel NUC 10th Gen)

Intel Core i5 10th Gen
16GB RAM
500GB NVMe
250GB SSD

Standalone Proxmox hypervisor. Hosts paw01 and paw02 on isolated hardware, outside the cluster.

## 4. Backup

### pbs01 (HP M01-F3224)

AMD Ryzen 5 5600G
8GB RAM
2x 1TB HDD (ZFS mirror)

Proxmox Backup Server. ZFS mirror protects against single-disk failure.

## 5. SIEM

### wazuh01 (Intel NUC 8th Gen)

Intel Core i5 8th Gen
16GB RAM
500GB NVMe
250GB SSD

Dedicated Wazuh SIEM. Runs on its own hardware, off the Proxmox cluster.

## 6. Network

### sw01 (Netgear GS728TPV2)

24-port managed switch, PoE on even ports.

Distribution switch. All cluster nodes, pve04, pbs01, wazuh01, ap01, and the fw01 LAN uplink terminate here.

### sw02 (TP-Link TL-SG108E)

8-port managed switch.

Access switch. Uplinks to sw01.

### ap01 (Ubiquiti AC Pro)

Wi-Fi access point, PoE-powered.