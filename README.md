flowchart LR
  %% Internet + DNS
  subgraph Internet
    C[Client]
    DNS[Public DNS]
  end

  C -->|DNS Query| DNS
  DNS -->|203.0.113.10| C
  DNS -->|203.0.113.11| C

  C -->|HTTPS 443| FW[Perimeter Firewall]

  %% ONE DATA CENTER
  subgraph DC1[Single Data Center]

    %% VIP1 + VIP2 (2 independent active VIPs)
    FW -->|203.0.113.10 → 10.10.10.10| VIP1[(VIP1 10.10.10.10:443)]
    FW -->|203.0.113.11 → 10.10.10.20| VIP2[(VIP2 10.10.10.20:443)]

    %% Pair 1
    subgraph PAIR1[F5 HA Pair 1]
      A1[F5-A1 (Active)]
      A2[F5-A2 (Standby)]
    end

    VIP1 --> A1
    A1 -. HA Sync/Failover .- A2

    %% Pair 2
    subgraph PAIR2[F5 HA Pair 2]
      B1[F5-B1 (Active)]
      B2[F5-B2 (Standby)]
    end

    VIP2 --> B1
    B1 -. HA Sync/Failover .- B2

    %% Backend App Servers
    subgraph APP[App Tier]
      APP1[APP1 10.20.20.11]
      APP2[APP2 10.20.20.12]
      APP3[APP3 10.20.20.13]
      APP4[APP4 10.20.20.14]
    end

    A1 --> APP1
    A1 --> APP2
    B1 --> APP3
    B1 --> APP4

  end
