# FirstProject
FirstProject

```mermaid
flowchart LR
  %% Internet & DNS
  subgraph Internet
    C[Client]
    DNS[Public DNS]
  end

  C -->|DNS Query app.example.com| DNS
  DNS -->|A: 203.0.113.10| C
  DNS -->|A: 203.0.113.11| C

  %% Firewall / Edge
  C -->|HTTPS 443| FW[Perimeter Firewalls]

  %% Single DC with 2 F5 HA pairs
  subgraph DC1[Single Data Center]

    %% VIPs mapped from public IPs
    FW -->|203.0.113.10 → 10.10.10.10| VIP1[(VIP1 10.10.10.10:443)]
    FW -->|203.0.113.11 → 10.10.10.20| VIP2[(VIP2 10.10.10.20:443)]

    %% Pair 1
    subgraph PAIR1[F5 LTM Pair 1]
      A1[F5-DC1-A1 (Active)]
      A2[F5-DC1-A2 (Standby)]
    end
    VIP1 --> A1
    A1 -. HA Sync/Failover .- A2

    %% Pair 2
    subgraph PAIR2[F5 LTM Pair 2]
      B1[F5-DC1-B1 (Active)]
      B2[F5-DC1-B2 (Standby)]
    end
    VIP2 --> B1
    B1 -. HA Sync/Failover .- B2

    %% Backend app servers (can be shared or sharded)
    subgraph APP_TIER[App Tier]
      APP1[APP1 10.20.20.11:443]
      APP2[APP2 10.20.20.12:443]
      APP3[APP3 10.20.20.13:443]
      APP4[APP4 10.20.20.14:443]
    end

    %% Example: Pair 1 uses APP1/APP2, Pair 2 uses APP3/APP4
    A1 --> APP1
    A1 --> APP2
    B1 --> APP3
    B1 --> APP4
```

  end
