# OpenStack Swift Object Storage

An object storage cluster based on OpenStack Swift, with Keystone for identity. I have built it as
an all-in-one setup for dev/staging and as a multi-node setup for internal production use.

## Topology

```mermaid
flowchart TB
    C["Clients<br/>(Swift API / S3 middleware)"] --> LB["Load balancer / TLS proxy"]
    LB --> P1["Swift proxy"]
    LB --> P2["Swift proxy"]
    P1 --> KS["Keystone<br/>authentication + tokens"]
    P2 --> KS
    P1 --> MC["Memcached<br/>token + account cache"]
    P2 --> MC

    subgraph storage["Storage nodes (one or more zones)"]
        S1["Node 1<br/>account / container / object servers"]
        S2["Node 2<br/>account / container / object servers"]
        S3["Node N<br/>account / container / object servers"]
    end

    P1 --> S1
    P1 --> S2
    P2 --> S3
    S1 <-. "replicators / auditors" .-> S2
    S2 <-. "replicators / auditors" .-> S3
```

## Design decisions

| Area | Approach |
|---|---|
| Rings | Separate account, container and object rings; partition power sized for long-term growth |
| Durability | 3 replicas spread across zones (hosts or racks) |
| Disks | XFS-formatted data disks mounted per device; container/account DBs on SSD where possible |
| Auth | Keystone with service users per consumer project |
| Scaling | Stateless proxies scale out behind the load balancer; storage scales by adding devices to the rings |

## Operations I run

- Installation of Keystone, proxies and storage nodes; ring creation and rebalancing.
- Adding capacity: add devices with gradual weight increase, rebalance, push rings to all nodes.
- Failed-disk handling: unmount, remove from rings or set weight to 0, replace, re-add.
- Debugging: proxy and object-server logs, `swift-recon` for replication and disk health,
  `swift-get-nodes` to trace where an object lives.
