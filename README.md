```mermaid

flowchart LR

    %% Client + DNS
    Client[Client] --> DNS[DNS]
    DNS -->|203.0.113.10| Client
    DNS -->|203.0.113.11| Client

    %% Edge FW
    Client --> FW[Perimeter Firewall]

    %% VIPs on external VLAN
    FW --> VIP1[VIP1 10.10.10.10:443\nExternal VLAN 10.10.10.0/24]
    FW --> VIP2[VIP2 10.10.10.20:443\nExternal VLAN 10.10.10.0/24]

    %% F5 HA Pair 1
    subgraph F5_Pair1[HA Pair 1]
        A1[F5-A1 (Active)\nLTM Only]
        A2[F5-A2 (Standby)]
    end
    VIP1 --> A1

    %% F5 HA Pair 2
    subgraph F5_Pair2[HA Pair 2]
        B1[F5-B1 (Active)\nLTM Only]
        B2[F5-B2 (Standby)]
    end
    VIP2 --> B1

    %% App VLAN
    subgraph App_VLAN[App VLAN 10.20.20.0/24]
        APP1[APP1 10.20.20.11:443]
        APP2[APP2 10.20.20.12:443]
        APP3[APP3 10.20.20.13:443]
        APP4[APP4 10.20.20.14:443]
    end

    %% DB VLAN
    subgraph DB_VLAN[DB VLAN 10.30.30.0/24]
        DB[DB Cluster 10.30.30.10]
    end

    %% Traffic from F5 to App servers (re-encrypted HTTPS)
    A1 --> APP1
    A1 --> APP2

    B1 --> APP3
    B1 --> APP4

    %% App to DB
    APP1 --> DB
    APP2 --> DB
    APP3 --> DB
    APP4 --> DB
```
