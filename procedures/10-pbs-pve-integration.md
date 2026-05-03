# PBS-PVE Integration

## Purpose
Wire pbs01 in as a backup target for each Proxmox node using per-node API tokens and per-node namespaces per [ADR-0014](../decisions/ADR-0014-backup-architecture.md). After this, each node can write backups to pbs01 but cannot read, modify, or prune any other node's history.

## Prerequisites
- [09-pbs-baseline](09-pbs-baseline.md) complete, datastore `ds01` healthy on pbs01
- [08-cluster-formation](08-cluster-formation.md) complete on pve01-03
- pve04 baselined per [06-proxmox-node-baseline](06-proxmox-node-baseline.md)
- Root SSH access to pbs01 and to any cluster node
- A scratch place to capture the four token secrets as they're generated. PBS only displays each secret once.

## How PBS resolves token permissions
A token's effective permission on a path is the intersection of the token's own ACL entries and its owner user's ACL entries. Both the user and the token need ACL entries on each relevant path or the intersection is empty. We assign the user a broader role on its namespace (`DatastorePowerUser`) and the token a narrower role (`DatastoreBackup`), so the intersection works out to `DatastoreBackup`, which is write-only with no prune capability. That is the prune protection from ADR-0014.

## Procedure

### 1. Capture the PBS cert fingerprint
SSH to pbs01 as `root`. Every PVE storage entry below needs this same fingerprint.
```bash
proxmox-backup-manager cert info | grep -i fingerprint
```
Save the colon-separated hex string.

### 2. Create per-node namespaces on `ds01`
In the PBS web UI at `https://10.0.10.10:8007`:

1. Datastore > ds01 in the left sidebar.
2. Select the datastore root in the content area and click **Add NS** at the top.
3. Create namespace `pve01`. Repeat for `pve02`, `pve03`, `pve04`.

Each namespace is a top-level isolation boundary inside the datastore.

### 3. Set up the first node (pve01)
Do all of these on pbs01 as `root`, then verify before moving to the next node.

```bash
# Create user
proxmox-backup-manager user create pve01@pbs --comment "backup writer for pve01"

# User ACLs (upper bound for the token)
proxmox-backup-manager acl update /datastore DatastoreAudit --auth-id 'pve01@pbs'
proxmox-backup-manager acl update /datastore/ds01/pve01 DatastorePowerUser --auth-id 'pve01@pbs'

# Token (capture the value field from the output before doing anything else)
proxmox-backup-manager user generate-token pve01@pbs backup

# Token ACLs (these intersect with the user ACLs to give the effective permission)
proxmox-backup-manager acl update /datastore DatastoreAudit --auth-id 'pve01@pbs!backup'
proxmox-backup-manager acl update /datastore/ds01/pve01 DatastoreBackup --auth-id 'pve01@pbs!backup'
```

Verify the token can see the datastore. Run from any cluster node:
```bash
pvesm scan pbs 10.0.10.10 'pve01@pbs!backup' --password '<token-secret-pve01>' --fingerprint '<fingerprint>'
```
Expect `ds01` to appear. If it does not, stop and diagnose before continuing. The most common causes are a mistyped token secret and a missing user-side ACL.

### 4. Repeat step 3 for pve02, pve03, pve04
Substitute the node name in the user id, token id, namespace path, and `pvesm scan` user argument. After each node, the scan from step 3's verification block should also return `ds01`.

### 5. Add cluster storage entries
SSH to any cluster node as `root`. `/etc/pve/storage.cfg` is cluster-shared, so adding from one node propagates to all three.
```bash
pvesm add pbs pbs-pve01 \
  --server 10.0.10.10 --datastore ds01 --namespace pve01 \
  --username 'pve01@pbs!backup' --password '<token-secret-pve01>' \
  --fingerprint '<fingerprint>' --nodes pve01

pvesm add pbs pbs-pve02 \
  --server 10.0.10.10 --datastore ds01 --namespace pve02 \
  --username 'pve02@pbs!backup' --password '<token-secret-pve02>' \
  --fingerprint '<fingerprint>' --nodes pve02

pvesm add pbs pbs-pve03 \
  --server 10.0.10.10 --datastore ds01 --namespace pve03 \
  --username 'pve03@pbs!backup' --password '<token-secret-pve03>' \
  --fingerprint '<fingerprint>' --nodes pve03
```
The `--nodes` flag restricts each entry to one node, so a compromise of pve02 cannot use pve01's storage entry.

### 6. Add pve04 storage entry
SSH to pve04 as `root`. pve04 is standalone so this lives in its local `/etc/pve/storage.cfg`.
```bash
pvesm add pbs pbs-pve04 \
  --server 10.0.10.10 --datastore ds01 --namespace pve04 \
  --username 'pve04@pbs!backup' --password '<token-secret-pve04>' \
  --fingerprint '<fingerprint>' --nodes pve04
```

### 7. Confirm each storage entry is healthy
In the PVE web UI, the `pbs-pveNN` storage row on each node should show usage statistics pulled live from pbs01. CLI version on each node:
```bash
pvesm status
```
The `pbs-pveNN` row for the local node should show `active 1` with real Total/Used. Other nodes' entries do not appear because of the `--nodes` restriction.

## Validation
- PBS web UI shows four namespaces under Datastore > ds01 > Content (`pve01`, `pve02`, `pve03`, `pve04`)
- `proxmox-backup-manager user list` on pbs01 shows the four PBS-realm users plus `root@pam` and `noah.barrett@pam`
- `proxmox-backup-manager acl list` on pbs01 shows two entries per node (one on the user, one on the token)
- `pvesm status` on each PVE node shows exactly one `pbs-pveNN` entry, active and reporting usage
- The four token secrets are saved in the password manager

## Rollback
- **PVE side:** `pvesm remove pbs-pveNN` on each affected node. Cluster entries can be removed from any cluster node.
- **PBS side:** delete the tokens, then the users. ACLs are cleaned up automatically when their auth-id no longer exists.

## Troubleshooting

**Symptom:** `pvesm scan pbs` returns no datastores, or `pvesm add pbs` fails with "Cannot find datastore".
**Cause:** Token has ACLs but the owner user does not. PBS intersects token and user permissions, so an empty user ACL means the token effectively has nothing.
**Fix:** Add matching user-side ACLs (broader role like `DatastorePowerUser`) on the same paths as the token. Step 3 covers this.
