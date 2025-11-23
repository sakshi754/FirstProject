# F5 BIG-IP LTM High Availability Architecture Diagrams

## Single Datacenter Active-Standby HA Architecture

```mermaid
graph TB
    subgraph Internet["Internet Edge"]
        Client1[("👤 Client<br/>Browser")]
        Client2[("👤 Mobile<br/>App")]
    end
    
    subgraph Border["Border/Firewall Layer"]
        FW1["🔥 Firewall-1"]
        FW2["🔥 Firewall-2"]
        Router1["🔀 Router-1<br/>172.16.10.1"]
        Router2["🔀 Router-2<br/>172.16.10.2"]
    end
    
    subgraph External["External VLAN 100 - DMZ<br/>172.16.10.0/24"]
        VIP1["🌐 VIP: 172.16.10.100:443<br/>api.bank.com<br/>HTTPS Virtual Server"]
        FloatExt["📍 Floating Self-IP<br/>172.16.10.80"]
    end
    
    subgraph F5_HA["F5 BIG-IP HA Pair"]
        subgraph F5_Device1["F5 BIG-IP-01 (ACTIVE)"]
            F5A_Mgmt["💻 Mgmt: 10.10.200.98"]
            F5A_Ext["🔌 External Self-IP<br/>172.16.10.70"]
            F5A_Int["🔌 Internal Self-IP<br/>192.168.10.70"]
            F5A_HA["🔌 HA Self-IP<br/>10.10.202.98"]
            F5A_SSL["🔐 SSL Offload/Re-encrypt<br/>TLS 1.3 + AES-256-GCM"]
            F5A_LB["⚖️ Load Balancer<br/>Least Connections"]
            F5A_Persist["🍪 Cookie Persistence<br/>AES-192 Encrypted"]
        end
        
        subgraph F5_Device2["F5 BIG-IP-02 (STANDBY)"]
            F5B_Mgmt["💻 Mgmt: 10.10.200.99"]
            F5B_Ext["🔌 External Self-IP<br/>172.16.10.71"]
            F5B_Int["🔌 Internal Self-IP<br/>192.168.10.71"]
            F5B_HA["🔌 HA Self-IP<br/>10.10.202.99"]
            F5B_Mirror["📋 Connection Mirror<br/>Persistence Sync"]
        end
        
        F5A_HA -.->|"ConfigSync TCP:4353<br/>Failover UDP:1026<br/>Mirroring TCP:1029"| F5B_HA
    end
    
    subgraph Internal["Internal VLAN 200 - App Tier<br/>192.168.10.0/24"]
        FloatInt["📍 Floating Self-IP<br/>192.168.10.80"]
        Pool["⚙️ Backend Pool<br/>backend_https_pool"]
    end
    
    subgraph AppServers["Application Servers"]
        App1["🖥️ App-Server-1<br/>10.10.10.11:8443<br/>Priority: 10"]
        App2["🖥️ App-Server-2<br/>10.10.10.12:8443<br/>Priority: 10"]
        App3["🖥️ App-Server-3<br/>10.10.10.13:8443<br/>Priority: 10"]
        AppDR1["🖥️ DR-App-1<br/>10.20.10.11:8443<br/>Priority: 5"]
        AppDR2["🖥️ DR-App-2<br/>10.20.10.12:8443<br/>Priority: 5"]
    end
    
    subgraph HealthMon["Health Monitoring"]
        Monitor["🏥 HTTPS Monitor<br/>GET /health<br/>Interval: 5s<br/>Timeout: 16s"]
    end
    
    Client1 -->|"1️⃣ HTTPS Request<br/>TLS 1.3 Handshake"| FW1
    Client2 --> FW2
    FW1 --> Router1
    FW2 --> Router2
    Router1 -->|"2️⃣ Route to VIP"| VIP1
    Router2 --> VIP1
    
    VIP1 -->|"3️⃣ SSL Offload<br/>Decrypt & Inspect"| F5A_SSL
    F5A_SSL -->|"4️⃣ Apply Security Policies<br/>Insert Headers"| F5A_LB
    F5A_LB -->|"5️⃣ Cookie Persistence Check"| F5A_Persist
    F5A_Persist -->|"6️⃣ Select Pool Member<br/>Least Connections"| Pool
    
    Pool -->|"7️⃣ SSL Re-encrypt<br/>New TLS Connection"| App1
    Pool --> App2
    Pool --> App3
    Pool -.->|"Fallback if primary down"| AppDR1
    Pool -.-> AppDR2
    
    Monitor -.->|"Continuous Health Checks"| App1
    Monitor -.-> App2
    Monitor -.-> App3
    Monitor -.-> AppDR1
    Monitor -.-> AppDR2
    
    F5A_Int -->|"Active Processing"| FloatInt
    F5A_Ext --> FloatExt
    FloatInt --> Pool
    FloatExt --> VIP1
    
    F5B_Mirror -.->|"Standby Mode<br/>Config Synced"| F5A_LB
    
    style F5_Device1 fill:#90EE90,stroke:#006400,stroke-width:3px
    style F5_Device2 fill:#FFB6C1,stroke:#8B0000,stroke-width:2px
    style VIP1 fill:#4169E1,stroke:#000080,stroke-width:3px,color:#FFF
    style F5A_SSL fill:#FFD700,stroke:#FF8C00,stroke-width:2px
    style Pool fill:#DDA0DD,stroke:#8B008B,stroke-width:2px
```

## Multi-Datacenter Active-Active with DNS GSLB

```mermaid
graph TB
    subgraph Global["Global Internet"]
        User1["👤 User<br/>North America"]
        User2["👤 User<br/>Europe"]
        User3["👤 User<br/>Asia"]
    end
    
    subgraph DNS_Layer["F5 BIG-IP DNS (GSLB) Layer"]
        DNS1["🌍 DNS-1 East<br/>ns1.bank.com"]
        DNS2["🌍 DNS-2 West<br/>ns2.bank.com"]
        DNS3["🌍 DNS-3 Europe<br/>ns3.bank.com"]
        
        DNS1 <-.->|"iQuery TCP:4353<br/>Encrypted Sync"| DNS2
        DNS2 <-.-> DNS3
        DNS3 <-.-> DNS1
    end
    
    subgraph DC_East["🏢 Primary Datacenter - East (Ashburn)<br/>Active Site"]
        subgraph F5_East["F5 HA Pair - East"]
            F5E1["⚡ F5-East-01<br/>ACTIVE<br/>VIP: 172.16.10.100"]
            F5E2["💤 F5-East-02<br/>STANDBY"]
            F5E1 -.->|"HA Sync"| F5E2
        end
        
        subgraph Apps_East["Application Tier"]
            AppE1["🖥️ App-East-1"]
            AppE2["🖥️ App-East-2"]
            AppE3["🖥️ App-East-3"]
        end
        
        subgraph Data_East["Data Tier"]
            DB_East["🗄️ Database<br/>Primary"]
            Redis_East["📦 Redis<br/>Session Store"]
        end
        
        F5E1 --> AppE1 & AppE2 & AppE3
        AppE1 & AppE2 & AppE3 --> DB_East
        AppE1 & AppE2 & AppE3 --> Redis_East
    end
    
    subgraph DC_West["🏢 DR Datacenter - West (San Francisco)<br/>Active Site"]
        subgraph F5_West["F5 HA Pair - West"]
            F5W1["⚡ F5-West-01<br/>ACTIVE<br/>VIP: 10.20.10.100"]
            F5W2["💤 F5-West-02<br/>STANDBY"]
            F5W1 -.-> F5W2
        end
        
        subgraph Apps_West["Application Tier"]
            AppW1["🖥️ App-West-1"]
            AppW2["🖥️ App-West-2"]
            AppW3["🖥️ App-West-3"]
        end
        
        subgraph Data_West["Data Tier"]
            DB_West["🗄️ Database<br/>Replica"]
            Redis_West["📦 Redis<br/>Session Store"]
        end
        
        F5W1 --> AppW1 & AppW2 & AppW3
        AppW1 & AppW2 & AppW3 --> DB_West
        AppW1 & AppW2 & AppW3 --> Redis_West
    end
    
    subgraph DC_Europe["🏢 Edge Datacenter - Europe (London)<br/>Active Site"]
        subgraph F5_EU["F5 HA Pair - Europe"]
            F5EU1["⚡ F5-EU-01<br/>ACTIVE<br/>VIP: 10.30.10.100"]
            F5EU2["💤 F5-EU-02<br/>STANDBY"]
            F5EU1 -.-> F5EU2
        end
        
        subgraph Apps_EU["Application Tier"]
            AppEU1["🖥️ App-EU-1"]
            AppEU2["🖥️ App-EU-2"]
        end
        
        subgraph Data_EU["Data Tier"]
            DB_EU["🗄️ Database<br/>Replica"]
            Redis_EU["📦 Redis<br/>Session Store"]
        end
        
        F5EU1 --> AppEU1 & AppEU2
        AppEU1 & AppEU2 --> DB_EU
        AppEU1 & AppEU2 --> Redis_EU
    end
    
    subgraph Management["🔧 Central Management"]
        BIGIQ["☁️ F5 BIG-IQ<br/>Centralized Config<br/>& Licensing"]
        SIEM["📊 SIEM/Logging<br/>Splunk/ELK"]
        Monitoring["📈 Monitoring<br/>Prometheus/Grafana"]
    end
    
    User1 -->|"1️⃣ DNS Query<br/>api.bank.com"| DNS1
    User2 --> DNS2
    User3 --> DNS3
    
    DNS1 -->|"2️⃣ Topology Routing<br/>172.16.10.100"| User1
    DNS2 -->|"2️⃣ Geographic<br/>10.30.10.100"| User2
    DNS3 -->|"2️⃣ Proximity<br/>10.20.10.100"| User3
    
    User1 -->|"3️⃣ HTTPS to VIP"| F5E1
    User2 --> F5EU1
    User3 --> F5W1
    
    DB_East <-.->|"Replication"| DB_West
    DB_West <-.-> DB_EU
    DB_EU <-.-> DB_East
    
    Redis_East <-.->|"Session Sync"| Redis_West
    Redis_West <-.-> Redis_EU
    Redis_EU <-.-> Redis_East
    
    BIGIQ -.->|"Config Push"| F5E1 & F5W1 & F5EU1
    
    F5E1 & F5W1 & F5EU1 -->|"HSL Logging"| SIEM
    F5E1 & F5W1 & F5EU1 -->|"SNMP/Metrics"| Monitoring
    
    DNS1 <-.->|"Health Monitoring<br/>iQuery"| F5E1
    DNS2 <-.-> F5W1
    DNS3 <-.-> F5EU1
    
    style DC_East fill:#E8F5E9,stroke:#4CAF50,stroke-width:3px
    style DC_West fill:#FFF3E0,stroke:#FF9800,stroke-width:3px
    style DC_Europe fill:#E3F2FD,stroke:#2196F3,stroke-width:3px
    style DNS_Layer fill:#F3E5F5,stroke:#9C27B0,stroke-width:3px
    style F5E1 fill:#90EE90,stroke:#006400,stroke-width:3px
    style F5W1 fill:#90EE90,stroke:#006400,stroke-width:3px
    style F5EU1 fill:#90EE90,stroke:#006400,stroke-width:3px
```

## SSL/TLS Traffic Flow: End-to-End Encryption

```mermaid
sequenceDiagram
    participant Client as 👤 Client Browser
    participant F5_VIP as 🌐 F5 Virtual Server<br/>VIP: 172.16.10.100:443
    participant ClientSSL as 🔐 Client SSL Profile<br/>TLS 1.3 Offload
    participant Inspect as 🔍 Traffic Inspection<br/>iRules, Security Policies
    participant ServerSSL as 🔐 Server SSL Profile<br/>TLS 1.2+ Re-encrypt
    participant Backend as 🖥️ Backend Server<br/>App-01:8443
    
    Note over Client,Backend: Initial Connection - SSL/TLS Handshake
    
    Client->>F5_VIP: 1. TCP SYN to 172.16.10.100:443
    F5_VIP->>Client: 2. TCP SYN-ACK
    Client->>F5_VIP: 3. TCP ACK
    
    Note over Client,ClientSSL: Client-Side TLS 1.3 Handshake
    
    Client->>ClientSSL: 4. ClientHello (TLS 1.3, Cipher Suites)
    ClientSSL->>Client: 5. ServerHello + Certificate<br/>(*.bank.com, ECDSA P-256)
    ClientSSL->>Client: 6. Certificate + Server Key Exchange
    Client->>ClientSSL: 7. Client Key Exchange + Change Cipher Spec
    ClientSSL->>Client: 8. Change Cipher Spec + Finished
    
    Note over ClientSSL: ✅ TLS Session Established<br/>Cipher: TLS13-AES256-GCM-SHA384
    
    Client->>ClientSSL: 9. Encrypted HTTP Request<br/>POST /api/v1/transfer
    
    Note over ClientSSL,Inspect: SSL Offload - Cleartext Processing
    
    ClientSSL->>Inspect: 10. Decrypted Request (Cleartext)<br/>Headers: Host, User-Agent, Cookies
    
    Inspect->>Inspect: 11. Apply Security Policies<br/>• DLP Scan for PII/PCI data<br/>• Check WAF rules<br/>• Execute iRules<br/>• Cookie Persistence Check<br/>• Load Balancing Decision
    
    Inspect->>Inspect: 12. Insert Headers<br/>• X-Forwarded-For: Client-IP<br/>• X-Forwarded-Proto: https<br/>• X-SSL-Cipher: TLS13-AES256-GCM<br/>• Strict-Transport-Security
    
    Note over Inspect,ServerSSL: Server-Side TLS Re-encryption
    
    Inspect->>ServerSSL: 13. Send Cleartext to Server SSL Profile
    ServerSSL->>Backend: 14. New TLS Handshake<br/>ClientHello (TLS 1.2+)
    Backend->>ServerSSL: 15. ServerHello + Backend Certificate
    Backend->>ServerSSL: 16. Certificate + Key Exchange
    ServerSSL->>Backend: 17. Client Finished
    Backend->>ServerSSL: 18. Server Finished
    
    Note over ServerSSL,Backend: ✅ Backend TLS Session Established<br/>Cipher: ECDHE-RSA-AES256-GCM-SHA384
    
    ServerSSL->>Backend: 19. Encrypted HTTP Request<br/>(Re-encrypted with backend TLS)
    
    Backend->>Backend: 20. Process Transaction<br/>Database Query, Business Logic
    
    Backend->>ServerSSL: 21. Encrypted HTTP Response<br/>200 OK + JSON Response
    
    Note over ServerSSL,ClientSSL: Response Flow - Decrypt & Re-encrypt
    
    ServerSSL->>Inspect: 22. Decrypt Backend Response
    Inspect->>Inspect: 23. Apply Response Policies<br/>• Add Security Headers<br/>• Insert Persistence Cookie<br/>• Content Compression
    
    Inspect->>ClientSSL: 24. Cleartext Response
    ClientSSL->>Client: 25. Re-encrypt with Client TLS Session<br/>Encrypted Response to Client
    
    Note over Client: ✅ Transaction Complete<br/>End-to-End Encryption Maintained<br/>Full Visibility at F5 Layer
    
    rect rgb(255, 240, 245)
        Note over Client,Backend: 🔒 Security Features Active:<br/>• Zero Trust Architecture<br/>• Deep Packet Inspection<br/>• DLP & WAF Protection<br/>• Encrypted Cookie Persistence<br/>• Perfect Forward Secrecy (PFS)
    end
```

## Failover Sequence: Active to Standby Transition

```mermaid
sequenceDiagram
    participant Active as ⚡ F5-01 ACTIVE<br/>Processing Traffic
    participant Standby as 💤 F5-02 STANDBY<br/>Config Synced
    participant Network as 🔀 Network Switch
    participant Client as 👤 Active Client
    participant Backend as 🖥️ Backend Pool
    
    Note over Active,Standby: Normal Operations - HA Pair Healthy
    
    Active->>Standby: ConfigSync (TCP 4353)<br/>Continuous Sync
    Active->>Standby: Network Failover (UDP 1026)<br/>Heartbeat every 1 second
    Active->>Standby: Connection Mirror (TCP 1029)<br/>Active Connection States
    Active->>Standby: Persistence Mirror (TCP 1028)<br/>Cookie Persistence Records
    
    Client->>Active: HTTP Requests<br/>172.16.10.100:443
    Active->>Backend: Load Balanced Traffic
    Backend->>Active: Responses
    Active->>Client: Responses with Persistence Cookie
    
    Note over Active: ❌ FAILURE EVENT<br/>Hardware Power Loss
    
    rect rgb(255, 230, 230)
        Note over Active: Device Goes Offline<br/>⏱️ T+0 seconds
    end
    
    Active--xStandby: ❌ Heartbeat Lost (UDP 1026)
    Active--xStandby: ❌ ConfigSync Lost (TCP 4353)
    
    Note over Standby: ⏱️ T+3 seconds<br/>Timeout Threshold Reached
    
    Standby->>Standby: 🚨 Failure Detected<br/>• UDP heartbeat timeout<br/>• TCP keepalive timeout<br/>• Serial cable signal lost
    
    Standby->>Standby: 🔄 Initiate Failover Process<br/>• Change state to ACTIVE<br/>• Assume Traffic Group ownership<br/>• Bind floating self-IPs
    
    Note over Standby: ⏱️ T+3-5 seconds<br/>Becoming Active
    
    Standby->>Network: 📢 Gratuitous ARP<br/>MAC address update for:<br/>• 172.16.10.80 (Float External)<br/>• 192.168.10.80 (Float Internal)<br/>• 172.16.10.100 (VIP)
    
    Network->>Network: 🔄 Update MAC Table<br/>Re-learn port associations
    
    Note over Standby: ✅ NOW ACTIVE<br/>⏱️ T+5 seconds
    
    Client->>Standby: New HTTP Request<br/>Same VIP 172.16.10.100:443
    
    Standby->>Standby: ✅ Lookup Mirrored State<br/>• Connection table present<br/>• Persistence record exists<br/>• Route to same backend server
    
    Standby->>Backend: Traffic Resumes<br/>Same backend as before
    Backend->>Standby: Response
    Standby->>Client: Seamless Response<br/>Session Maintained
    
    rect rgb(230, 255, 230)
        Note over Client,Backend: ✅ Failover Complete<br/>Total Time: ~5 seconds<br/>Active Connections: Preserved<br/>New Connections: Immediate<br/>Zero Data Loss
    end
    
    Note over Active,Standby: Post-Failover State
    
    Note over Active: ⚠️ OFFLINE<br/>Awaiting Repair
    Note over Standby: ⚡ ACTIVE<br/>Processing All Traffic
    
    opt Auto-Failback Disabled (Recommended)
        Note over Standby: Remains ACTIVE until<br/>manual intervention
    end
    
    opt Auto-Failback Enabled
        Note over Active: Device Returns Online
        Active->>Standby: Rejoin Device Group
        Note over Active,Standby: Wait 60 seconds (default)
        Standby->>Active: Failback Traffic Group
        Note over Active: ⚡ ACTIVE Again
        Note over Standby: 💤 STANDBY Again
    end
```

## Horizontal Scaling with ECMP

```mermaid
graph TB
    subgraph Internet["Internet / WAN"]
        Clients["👥 Clients<br/>api.bank.com"]
    end
    
    subgraph Core_Routers["Core Network - ECMP Enabled"]
        Router1["🔀 Core Router-1<br/>BGP AS 65000<br/>ECMP Max-Paths: 8"]
        Router2["🔀 Core Router-2<br/>BGP AS 65000<br/>ECMP Max-Paths: 8"]
        
        Router1 <-.->|"iBGP Peering"| Router2
    end
    
    subgraph ECMP_VIP["Advertised VIP Route<br/>172.16.10.100/32"]
        VIP_Route["📍 Equal-Cost Multi-Path<br/>8 Active Paths to VIP"]
    end
    
    subgraph Pair1["F5 HA Pair 1 (Primary)"]
        F5_1A["⚡ F5-01A ACTIVE<br/>BGP AS 65001<br/>Advertises 172.16.10.100/32"]
        F5_1B["💤 F5-01B STANDBY"]
        F5_1A <-.->|"HA Sync"| F5_1B
        F5_1A -->|"eBGP Peer<br/>Route: 172.16.10.100/32"| Router1
        F5_1A --> Router2
    end
    
    subgraph Pair2["F5 HA Pair 2"]
        F5_2A["⚡ F5-02A ACTIVE<br/>BGP AS 65002<br/>Advertises 172.16.10.100/32"]
        F5_2B["💤 F5-02B STANDBY"]
        F5_2A <-.-> F5_2B
        F5_2A --> Router1
        F5_2A --> Router2
    end
    
    subgraph Pair3["F5 HA Pair 3"]
        F5_3A["⚡ F5-03A ACTIVE<br/>BGP AS 65003<br/>Advertises 172.16.10.100/32"]
        F5_3B["💤 F5-03B STANDBY"]
        F5_3A <-.-> F5_3B
        F5_3A --> Router1
        F5_3A --> Router2
    end
    
    subgraph Pair4["F5 HA Pair 4"]
        F5_4A["⚡ F5-04A ACTIVE<br/>BGP AS 65004<br/>Advertises 172.16.10.100/32"]
        F5_4B["💤 F5-04B STANDBY"]
        F5_4A <-.-> F5_4B
        F5_4A --> Router1
        F5_4A --> Router2
    end
    
    subgraph Backend["Backend Application Pool"]
        AppPool["🖥️🖥️🖥️ Application Servers<br/>Shared by all F5 pairs"]
    end
    
    Clients -->|"1️⃣ Traffic to VIP"| Router1
    Clients --> Router2
    
    Router1 -->|"2️⃣ ECMP Hash<br/>5-tuple: SrcIP, DstIP<br/>SrcPort, DstPort, Proto"| VIP_Route
    Router2 --> VIP_Route
    
    VIP_Route -->|"3️⃣ Path 1<br/>~25% Traffic"| F5_1A
    VIP_Route -->|"3️⃣ Path 2<br/>~25% Traffic"| F5_2A
    VIP_Route -->|"3️⃣ Path 3<br/>~25% Traffic"| F5_3A
    VIP_Route -->|"3️⃣ Path 4<br/>~25% Traffic"| F5_4A
    
    F5_1A -->|"4️⃣ Load Balance"| AppPool
    F5_2A --> AppPool
    F5_3A --> AppPool
    F5_4A --> AppPool
    
    subgraph Monitoring["Health Monitoring"]
        BFD1["⚡ BFD to Pair 1<br/>Interval: 50ms"]
        BFD2["⚡ BFD to Pair 2<br/>Interval: 50ms"]
        BFD3["⚡ BFD to Pair 3<br/>Interval: 50ms"]
        BFD4["⚡ BFD to Pair 4<br/>Interval: 50ms"]
    end
    
    Router1 -.->|"BFD Monitor"| BFD1
    Router1 -.-> BFD2
    Router1 -.-> BFD3
    Router1 -.-> BFD4
    
    BFD1 -.-> F5_1A
    BFD2 -.-> F5_2A
    BFD3 -.-> F5_3A
    BFD4 -.-> F5_4A
    
    subgraph Capacity["Capacity & Scaling"]
        Cap1["📊 Total Capacity:<br/>4 pairs × 20 Gbps = 80 Gbps"]
        Cap2["📈 Effective Capacity:<br/>60 Gbps (N+1 redundancy)"]
        Cap3["➕ Scale Out:<br/>Add pairs incrementally<br/>No downtime required"]
    end
    
    style VIP_Route fill:#4169E1,stroke:#000080,stroke-width:3px,color:#FFF
    style F5_1A fill:#90EE90,stroke:#006400,stroke-width:2px
    style F5_2A fill:#90EE90,stroke:#006400,stroke-width:2px
    style F5_3A fill:#90EE90,stroke:#006400,stroke-width:2px
    style F5_4A fill:#90EE90,stroke:#006400,stroke-width:2px
    style AppPool fill:#DDA0DD,stroke:#8B008B,stroke-width:2px
```

## Complete Network Topology with VLANs

```mermaid
graph TB
    subgraph Physical["Physical Infrastructure"]
        subgraph Rack["Rack 1 - F5 HA Pair"]
            Device1["🖥️ F5 BIG-IP-01<br/>Hardware: i10800<br/>80 Gbps Throughput"]
            Device2["🖥️ F5 BIG-IP-02<br/>Hardware: i10800<br/>80 Gbps Throughput"]
        end
        
        subgraph Switches["Network Switches"]
            CoreSW1["⚡ Core Switch-1<br/>Cisco Nexus 9K"]
            CoreSW2["⚡ Core Switch-2<br/>Cisco Nexus 9K"]
        end
    end
    
    subgraph VLAN_External["🌐 VLAN 100 - External DMZ<br/>172.16.10.0/24"]
        ExtVLAN["External Network Segment"]
        VIP_Prod["📍 VIP: 172.16.10.100:443<br/>api.bank.com"]
        VIP_Admin["📍 VIP: 172.16.10.101:443<br/>admin.bank.com"]
        Float_Ext["📍 Floating: 172.16.10.80"]
        Self_Ext1["📍 F5-01: 172.16.10.70"]
        Self_Ext2["📍 F5-02: 172.16.10.71"]
        Gateway_Ext["🚪 Gateway: 172.16.10.1"]
    end
    
    subgraph VLAN_Internal["🏢 VLAN 200 - Internal App<br/>192.168.10.0/24"]
        IntVLAN["Internal Network Segment"]
        Float_Int["📍 Floating: 192.168.10.80"]
        Self_Int1["📍 F5-01: 192.168.10.70"]
        Self_Int2["📍 F5-02: 192.168.10.71"]
        Pool_Web["🖥️ Web Pool<br/>192.168.10.10-20"]
        Pool_API["🖥️ API Pool<br/>192.168.10.30-40"]
    end
    
    subgraph VLAN_Database["🗄️ VLAN 201 - Database<br/>192.168.20.0/24"]
        DBVLAN["Database Network Segment"]
        DB_Primary["💾 DB-Primary<br/>192.168.20.10"]
        DB_Replica["💾 DB-Replica<br/>192.168.20.11"]
        Redis_Cluster["📦 Redis Cluster<br/>192.168.20.20-22"]
    end
    
    subgraph VLAN_HA["🔄 VLAN 300 - HA/Sync<br/>10.10.202.0/30"]
        HAVLAN["HA Network Segment<br/>Dedicated Isolated"]
        Self_HA1["📍 F5-01: 10.10.202.98"]
        Self_HA2["📍 F5-02: 10.10.202.99"]
        HA_Traffic["🔒 ConfigSync TCP:4353<br/>🔒 Failover UDP:1026<br/>🔒 Mirroring TCP:1029"]
    end
    
    subgraph VLAN_Mgmt["🔧 VLAN 1 - Management<br/>10.10.200.0/24"]
        MgmtVLAN["Management Network<br/>Out-of-Band"]
        Mgmt1["📍 F5-01 Mgmt: 10.10.200.98"]
        Mgmt2["📍 F5-02 Mgmt: 10.10.200.99"]
        JumpBox["💻 Jump Server<br/>10.10.200.50"]
        BIGIQ["☁️ BIG-IQ<br/>10.10.200.70"]
        SIEM["📊 SIEM Collector<br/>10.10.200.80"]
    end
    
    Device1 -.->|"Port 1.1 Tagged"| ExtVLAN
    Device1 -.->|"Port 1.2 Tagged"| IntVLAN
    Device1 -.->|"Port 1.3 Untagged"| HAVLAN
    Device1 -.->|"Mgmt Port"| MgmtVLAN
    
    Device2 -.-> ExtVLAN
    Device2 -.-> IntVLAN
    Device2 -.-> HAVLAN
    Device2 -.-> MgmtVLAN
    
    ExtVLAN --> CoreSW1 & CoreSW2
    IntVLAN --> CoreSW1 & CoreSW2
    HAVLAN --> CoreSW1 & CoreSW2
    MgmtVLAN --> CoreSW1 & CoreSW2
    
    Float_Ext --> VIP_Prod & VIP_Admin
    Self_Ext1 -.->|"Non-Floating"| Device1
    Self_Ext2 -.->|"Non-Floating"| Device2
    
    Float_Int --> Pool_Web & Pool_API
    Self_Int1 -.-> Device1
    Self_Int2 -.-> Device2
    
    Pool_API --> DB_Primary & Redis_Cluster
    
    Self_HA1 <-.->|"HA Traffic"| Self_HA2
    
    Mgmt1 --> BIGIQ & SIEM
    Mgmt2 --> BIGIQ & SIEM
    JumpBox --> Mgmt1 & Mgmt2
    
    subgraph Security["🔐 Security Zones"]
        Zone1["Zone 1: Untrusted (VLAN 100)"]
        Zone2["Zone 2: Semi-Trusted (VLAN 200)"]
        Zone3["Zone 3: Restricted (VLAN 201)"]
        Zone4["Zone 4: Management (VLAN 1)"]
        Zone5["Zone 5: HA Only (VLAN 300)"]
    end
    
    subgraph Firewall_Rules["🔥 AFM Firewall Policies"]
        Rule1["External → Internal:<br/>Allow HTTPS 8443 only"]
        Rule2["Internal → Database:<br/>Allow SQL 3306/5432"]
        Rule3["Internal → Redis:<br/>Allow 6379"]
        Rule4["Management → Devices:<br/>SSH 22, HTTPS 443 only"]
        Rule5["HA VLAN:<br/>Isolated, no routing"]
    end
    
    style VLAN_External fill:#FFE4E1,stroke:#DC143C,stroke-width:2px
    style VLAN_Internal fill:#E0FFE0,stroke:#228B22,stroke-width:2px
    style VLAN_Database fill:#E6E6FA,stroke:#4B0082,stroke-width:2px
    style VLAN_HA fill:#FFFACD,stroke:#FFD700,stroke-width:3px
    style VLAN_Mgmt fill:#F0F8FF,stroke:#4682B4,stroke-width:2px
    style Device1 fill:#90EE90,stroke:#006400,stroke-width:3px
    style Device2 fill:#FFB6C1,stroke:#8B0000,stroke-width:2px
```

---

## Usage Instructions

### Rendering in GitHub README.md

1. Copy any of the Mermaid diagrams above
2. Paste directly into your `README.md` file
3. GitHub will automatically render the diagrams

### Example README Structure

```markdown
# F5 BIG-IP LTM Architecture

## Architecture Overview
Our enterprise F5 deployment uses active-standby HA pairs...

## Network Diagram
[Include Single Datacenter diagram here]

## Multi-Site Design  
[Include Multi-Datacenter diagram here]

## SSL Traffic Flow
[Include SSL/TLS sequence diagram here]
```

### Diagram Capabilities

- ✅ **Fully GitHub Compatible** - All diagrams use Mermaid syntax supported by GitHub's native renderer
- ✅ **Interactive** - Readers can zoom and pan on complex diagrams
- ✅ **Version Controlled** - Diagrams are text-based, perfect for Git diffs
- ✅ **No External Dependencies** - No need for separate image files
- ✅ **Easy Updates** - Modify diagram code to update architecture

### Customization Tips

1. **Change Colors**: Modify `style` statements at the bottom of diagrams
2. **Add Components**: Insert new subgraphs or nodes using existing syntax patterns
3. **Adjust Layout**: Change `TB` (top-bottom) to `LR` (left-right) for horizontal layouts
4. **Add Details**: Include additional labels, ports, or IP addresses as needed

### Testing Diagrams

Before committing, test your diagrams at:
- [Mermaid Live Editor](https://mermaid.live)
- Preview in VS Code with Mermaid extension
- GitHub's preview tab

---

## Diagram Legend

| Symbol | Meaning |
|--------|---------|
| ⚡ | Active Device |
| 💤 | Standby Device |
| 🌐 | Virtual IP (VIP) |
| 📍 | IP Address |
| 🔐 | SSL/Encryption |
| ⚖️ | Load Balancer |
| 🖥️ | Server |
| 🗄️ | Database |
| 📦 | Cache/Session Store |
| 🔀 | Router/Switch |
| 🔥 | Firewall |
| 👤 | End User |
| 🏥 | Health Monitor |
| ☁️ | Management System |
| 📊 | Monitoring/Logging |

---

## Version History

- **v1.0** - Initial architecture diagrams
- **v1.1** - Added ECMP scaling diagram
- **v1.2** - Enhanced VLAN topology with security zones
- **v1.3** - Added detailed SSL traffic flow sequence
