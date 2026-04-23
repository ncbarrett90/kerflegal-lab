# ADR-0007: DNS Architecture

| Field | Value |
|---|---|
| Status | Superseded |
| Date | 2026-04-18 |
| Supersedes | - |
| Superseded By | [ADR-0011](ADR-0011-infrastructure-dns-zone.md) |

## Context

DNS structure needs to be settled before any devices are named or joined to the domain. The lab also needs to support both internet-reachable services under a public domain and an internal-only namespace for the AD environment.

## Decision

The public domain `kerflegal.com` is registered with GoDaddy. Internal AD resources sit under the `corp.kerflegal.com` subdomain. This mirrors a common enterprise split-namespace pattern.

### Zones and hosting

- **kerflegal.com (external)** - Hosted on GoDaddy nameservers. Publicly resolvable.
- **corp.kerflegal.com (internal)** - AD-integrated DNS zone on the domain controllers. Resolvable only from inside the lab. No public delegation exists for this subdomain.

### Resolvers

- **OPNsense Unbound on fw01** - Recursive resolver for non-domain-joined devices (network gear, PVE nodes, PBS, Wazuh, PAWs, and other lab infrastructure not bound to AD).
- **Domain controllers** - Authoritative for `corp.kerflegal.com` and act as the resolver for domain-joined systems.
- **Upstream forwarder** - All external recursion forwards to Quad9 (9.9.9.9, 149.112.112.112).

### Per-zone resolver assignment

| Zone | Resolver |
|---|---|
| INFRA | fw01 Unbound |
| SECURITY | fw01 Unbound |
| PAW | fw01 Unbound |
| IDENTITY | Domain controllers (for non-DC devices, e.g. ca01) |
| SERVERS | Domain controllers |
| CLIENTS | Domain controllers |
| HOME | fw01 Unbound |

## Alternatives Considered

- **kerflegal.local** - Use a `.local` domain for internal DNS. Conflicts with mDNS and other modern network services. Rejected.
- **Local DNS only (no public domain)** - Limits the lab upfront and forecloses enterprise-style split DNS and any future externally-reachable services. Rejected.

## Consequences

- Upfront cost to register and renew the public domain.
- Split-horizon design is more complex than a single internal namespace.
- Cleaner separation of what is internal vs externally reachable.

## References

- [RFC 6762 - Multicast DNS (`.local` top-level domain)](https://datatracker.ietf.org/doc/html/rfc6762)
