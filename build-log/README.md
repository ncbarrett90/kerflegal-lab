# Build log

The build log is the narrative side of the build. Each phase gets a writeup that walks through what was done, why, and what almost went wrong, so a reader landing in the repo can understand how the lab came together before opening procedures or decisions for the underlying detail.

That detail lives elsewhere in the repo. Step-by-step device buildout is collected in [`procedures/`](../procedures/), and the architectural reasoning along with the alternatives rejected is in [`decisions/`](../decisions/). The change log, which is gitignored, is the chronological ledger of state changes that this narrative covers.

## Index

| Phase | Period | Summary |
|---|---|---|
| [Network Buildout](2026-04-network-buildout/) | April 2026 | Firewall hardening, VLAN segmentation, switch buildout, network cutover, and AP adoption |
