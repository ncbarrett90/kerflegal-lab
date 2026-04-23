# Runbook: ISO Hash Verification

> **Usage:** Run before any downloaded ISO is attached to a VM, written to installation media, or uploaded to a Proxmox host. Applies to both Linux and Windows workstations.

---

## Purpose

Verify that a downloaded ISO matches the hash published by the vendor. Confirms the file is not corrupted and has not been tampered with in transit.

## Prerequisites

- ISO downloaded from the official vendor source over HTTPS
- Expected hash obtained from the vendor's website over HTTPS (never from a forum, mirror, or README inside the ISO)
- For signed checksums: vendor's GPG public key imported from a trusted source
- PowerShell 5.1+ (Windows) or coreutils (Linux, built-in)

## Procedure

### Linux

1. Compute the hash of the ISO. Match the algorithm the vendor publishes.

```bash
sha256sum /path/to/file.iso
# or
sha512sum /path/to/file.iso
```

2. If the vendor publishes a checksum file (e.g. `SHA256SUMS`), verify in one step.

```bash
cd /path/to/iso/directory
sha256sum -c SHA256SUMS --ignore-missing
```

3. If the vendor publishes a GPG-signed checksum file, verify the signature before trusting the hashes.

```bash
gpg --verify SHA256SUMS.gpg SHA256SUMS
sha256sum -c SHA256SUMS --ignore-missing
```

### Windows

1. Compute the hash with `Get-FileHash`.

```powershell
Get-FileHash -Path "C:\path\to\file.iso" -Algorithm SHA256
```

2. Compare against the expected value.

```powershell
$expected = "paste-vendor-hash-here"
$actual = (Get-FileHash -Path "C:\path\to\file.iso" -Algorithm SHA256).Hash
if ($actual -eq $expected.ToUpper()) { "MATCH" } else { "MISMATCH" }
```

## Validation

- `sha256sum -c` returns `filename: OK`
- PowerShell comparison returns `MATCH`
- `gpg --verify` returns `Good signature` (when applicable)

## Rollback

Not applicable. Verification is read-only.

If the hash mismatches, the ISO is considered compromised. Delete the file and re-download from the official vendor source. Do not reuse or retry with the same file.

## Troubleshooting

**Symptom:**
**Cause:**
**Fix:**