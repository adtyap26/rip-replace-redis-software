# Redis Enterprise Upgrade POC (6.2.8 → 7.22)

This repository documents a proof of concept for upgrading a Redis Enterprise cluster from version 6.2.8 to 7.22.

Two upgrade approaches were tested:

1. **Rip-and-Replace Migration** — spin up a fresh 7.22 cluster, replicate all databases from the old 6.2 cluster using Replica Of, then cut over clients and decommission the old cluster.
2. **In-place Upgrade** — upgrade the existing cluster nodes directly, stepping through an intermediate version (7.4.x) before reaching 7.22.

## Files

| File                           | Description                                         |
| ------------------------------ | --------------------------------------------------- |
| `result.md`                    | POC summary and findings                            |
| `rip_and_replace_migration.md` | Step-by-step guide for the Rip-and-Replace approach |
| `Upgrade-using-inplace.md`     | Notes and steps for the in-place upgrade approach   |
| `assets/`                      | Screenshots                                         |
