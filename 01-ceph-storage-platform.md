# Ceph Storage Platform

A multi-cluster Ceph platform that serves block (RBD), object (S3/Swift via RGW) and file
(CephFS, NFS) storage to a public cloud.

**Scale operated:** 10+ production clusters; the largest with ~800–1,000 OSDs across dozens of hosts,
mixed HDD and SSD/NVMe tiers, 100 GbE storage networking.

## Topology

```mermaid
flowchart TB
    subgraph clients["Clients"]
        VM["Cloud VMs / hypervisors<br/>(RBD block volumes)"]
        S3C["S3 / Swift clients"]
        NFSC["NFS clients"]
    end

    subgraph edge["Gateway tier"]
        LB["Reverse proxy / TLS termination"]
        RGW["RGW daemons<br/>(one service per tenant class)"]
        NFS["NFS-Ganesha"]
    end

    subgraph ctrl["Control plane"]
        MON["MON quorum (3 or 5)"]
        MGR["MGR active + standby<br/>orchestrator, balancer, exporter"]
        MDS["MDS active + standby"]
    end

    subgraph data["Data plane (RADOS)"]
        HDD["OSD - HDD tier<br/>capacity pools"]
        SSD["OSD - SSD/NVMe tier<br/>index, metadata, fast pools"]
    end

    VM --> MON
    VM --> HDD
    VM --> SSD
    S3C --> LB --> RGW --> HDD
    RGW --> SSD
    NFSC --> NFS --> MDS
    MDS --> SSD
    MGR --- MON
```

## Design decisions

| Area | Approach |
|---|---|
| Deployment | `cephadm` with containerized daemons; placement by host labels (`mon`, `mgr`, `osd`, `rgw`, `mds`) |
| Failure domains | CRUSH hierarchy root → rack → host; separate CRUSH roots for HDD and SSD device classes |
| Data protection | 3× replication for latency-sensitive pools; erasure coding (e.g. 4+2, 6+2) for capacity pools |
| Pool layout | RGW index and CephFS metadata on SSD; bulk data on HDD |
| Networking | Separate public and cluster (replication) networks; LACP bonds; lossless RoCEv2 (PFC + ECN/DCQCN) on 100 GbE rails |
| Object gateway | Multiple RGW services isolated by port and placement, behind a reverse proxy for TLS |

## Network

```mermaid
flowchart LR
    subgraph host["Storage host"]
        B1["Bond: public network<br/>2x25 GbE LACP"]
        B2["Cluster network<br/>100 GbE"]
        MG["Management / OOB"]
    end
    B1 --> SW1["ToR switch pair (MLAG)"]
    B2 --> SW2["Storage fabric<br/>PFC + ECN, lossless class"]
    MG --> SW3["Management switch"]
```

## Operations I run

- **Build:** hardware/BIOS baseline, OS tuning, bootstrap with `cephadm`, acceptance testing with `fio` and `rados bench`.
- **Lifecycle:** add/remove OSDs and hosts, expand MON quorum, host maintenance mode, failed-disk replacement.
- **Data movement:** controlled reweighting, backfill/recovery throttling, balancer tuning to avoid client impact.
- **Upgrades:** rolling, one failure domain at a time, with health gates between steps.
- **Health:** start-of-shift checks (`ceph -s`, PG states, slow ops, capacity per root), alert triage flow.
- **Incidents:** e.g. false-positive OSD "down" events causing slow ops and inactive PGs; investigated
  heartbeats, network and host logs, wrote a postmortem and runbook.

## Monitoring

`ceph-exporter` + `node-exporter` on every host → Prometheus → Grafana dashboards and Alertmanager.
Key signals: health status, PG states, slow ops, OSD up/in, capacity and fill ratio per CRUSH root,
recovery throughput, disk latency, NIC errors and PFC pause frames.
