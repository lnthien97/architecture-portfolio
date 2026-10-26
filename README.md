# Architecture Portfolio

Generic reference architectures for the platforms I design, build and operate in production.

All pages are **sanitized**: no IP addresses, hostnames, domains, customer names, credentials or
internal product names. Numbers describe the order of magnitude of systems I have run, not any
specific environment.

| # | Architecture | What it covers |
|---|---|---|
| 1 | [Ceph storage platform](01-ceph-storage-platform.md) | Multi-cluster Ceph for block, object and file storage |
| 2 | [Swift object storage](02-swift-object-storage.md) | OpenStack Swift with Keystone authentication |
| 3 | [Cross-cluster backup & DR](03-backup-and-dr.md) | Incremental snapshot replication between storage clusters |
| 4 | [Observability platform](04-observability-platform.md) | Metrics and logs pipeline for a public cloud |
| 5 | [Kubernetes for the observability platform](05-kubernetes-observability-cluster.md) | Production cluster layout, node pools, operators, GitOps and secrets |

## Operating principles I apply everywhere

- **Read before write.** Every change starts with a read-only assessment and a written plan with rollback.
- **Failure domains first.** Placement rules (racks, hosts, sites) are designed before capacity.
- **Observe everything.** Each component exports metrics and has alerts with a runbook behind them.
- **Secrets never in docs or code.** Secrets live in a vault and are injected at runtime.
- **Write it down.** Runbooks, triage flows and incident write-ups are part of the deliverable.
