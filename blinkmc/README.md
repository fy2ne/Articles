# BlinkMC Complete Technical Architecture & Bypass Engine

A structural blueprint of the BlinkMC Minecraft bot fleet engine, detailing its zero-leak network firewall, proxy verification pipeline, custom protocol framing, gateway bypass armor, and anti-bot probe subsystem.

```mermaid
flowchart TD
  %% ==========================================
  %% CLASS DEFINITIONS & PALETTE (NO EMOJIS)
  %% ==========================================
  classDef kernel fill:#0b1329,stroke:#00e5ff,stroke-width:2px,color:#e0f7fa
  classDef proxy fill:#14112e,stroke:#a78bfa,stroke-width:2px,color:#ede9fe
  classDef daemon fill:#09211c,stroke:#10b981,stroke-width:2px,color:#d1fae5
  classDef proto fill:#2b0e27,stroke:#f43f5e,stroke-width:2px,color:#ffe4e6
  classDef armor fill:#2c1b07,stroke:#f59e0b,stroke-width:2px,color:#fef3c7
  classDef hold fill:#062024,stroke:#06b6d4,stroke-width:2px,color:#cffafe
  classDef probe fill:#2b0b0b,stroke:#ef4444,stroke-width:2px,color:#fee2e2
  classDef decision fill:#1e293b,stroke:#cbd5e1,stroke-width:2px,color:#f8fafc

  %% ==========================================
  %% SUBGRAPH 1: ZERO-LEAK SECURITY KERNEL
  %% ==========================================
  subgraph KERNEL["1. ZERO-LEAK PROCESS SECURITY KERNEL (lib/ip-guard.js)"]
    K1["Runtime Bootstrap"] --> K2["installIpGuard() Hook Activated"]
    K2 --> K3["Monkeypatch net.createConnection"]
    K3 --> K4{"Target in Allowed Set?"}
    K4 -- "NO: Non-Proxy / Direct Target" --> K5["THROW: Direct TCP Blocked (Real IP Shielded)"]
    K4 -- "YES: SOCKS5 Proxy IP or Loopback" --> K6["Authorize Raw Outbound TCP Socket"]
    K6 --> K7["SOCKS5 Domain ATYP 0x03 Enforced"]
    K7 --> K8["Remote Proxy Resolves Target DNS (No Local DNS Leak)"]
  end

  %% ==========================================
  %% SUBGRAPH 2: PROXY INTELLIGENCE & VERIFICATION
  %% ==========================================
  subgraph PROXY_TIER["2. PROXY HARVESTING & PRE-FLIGHT VALIDATION (lib/proxy-*.js)"]
    P1["Multi-Source Public Proxy Feeds"] --> P2["Ingest & Normalize IP:PORT Lists"]
    P2 --> P3["Deduplicate & Sanitize Schemes"]
    P3 --> P4["Pre-Flight Checker: 300 Parallel Pings"]
    P4 --> P5["Send SOCKS5 Greeting: 05 01 00"]
    P5 --> P6{"Proxy Answers 05 00 < 1200ms?"}
    P6 -- "Timeout / Reset / Egress Block" --> P7["Mark Dead: deadUntil = now + 30m"]
    P6 -- "Valid SOCKS5 Greeting" --> P8["Flag ok=true + Record RTT latency"]
    P8 --> P9["Atomic Store: .state/proxies.json"]
    P9 --> P10["Live Pool Ready (Assigned to Bot Sessions)"]
  end

  K8 ==> SOCKS_CONN["socks5Connect(proxyHost, proxyPort, targetHost, targetPort)"]
  P10 ==> SOCKS_CONN

  %% ==========================================
  %% SUBGRAPH 3: FLEET ORCHESTRATION & DAEMON
  %% ==========================================
  subgraph ORCHESTRATION["3. FLEET DAEMON & MULTI-RUNTIME RUNTIME (daemon.js & main.js)"]
    D1["node daemon.js --rt=main / --rt=ghost"] --> D2["Load config/settings.json + usernames.txt"]
    D2 --> D3["Check Ephemeral .state/disabled.json"]
    D3 --> D4["Spawn McClient Instances per Username"]
    D4 --> D5["Global Join Gate (Max N handshakes/sec)"]
    D5 --> D6["Per-Proxy Concurrency Slot Locking"]
    D6 --> D7["Dispatch Client to SOCKS5 Pipeline"]
  end

  D7 ==> SOCKS_CONN

  %% ==========================================
  %% SUBGRAPH 4: LOW-LEVEL PROTOCOL ENGINE
  %% ==========================================
  subgraph PROTO_ENGINE["4. VANILLA PROTOCOL 761 ENGINE (lib/protocol.js)"]
    SOCKS_CONN --> M1["Tunnel Established to Target Port 25565"]
    M1 --> M2["Synthesize Handshake Packet (id=0x00, proto=761, next=2)"]
    M2 --> M3["Synthesize LoginStart Packet (id=0x00, username)"]
    M3 --> M4["Transmit Handshake + LoginStart Wire Sequence"]
    M4 --> M5["FrameReader Stream Buffer Push"]
    M5 --> M6["VarInt Stream Parser + Length Validation"]
    M6 --> M7{"Set Compression (id=0x03) Received?"}
    M7 -- "YES" --> M8["Activate Zlib Inflate / Deflate Engine"]
    M7 -- "NO" --> M9["Parse Body: Login Packets"]
    M8 --> M9
  end

  %% ==========================================
  %% SUBGRAPH 5: GATEWAY & ANTI-BOT BYPASS ARMOR
  %% ==========================================
  subgraph BYPASS_ARMOR["5. FLAMECORD & GATEWAY BYPASS ARMOR (lib/session.js)"]
    M9 --> A1{"Login State Packet Evaluation"}
    A1 -- "packet.id == 0x00" --> A2["Extract Chat JSON & Decode Plain Text"]
    A2 --> A3["classifyKick(text) Taxonomy Engine"]
    
    A3 -- "rejoin_gate: 'Please rejoin to connect'" --> G1["FlameCord Token Challenge Detected"]
    G1 --> G2{"Rejoin Streak <= 3?"}
    G2 -- "YES" --> G3["Hold Same Proxy IP -> Backoff 2000ms"]
    G3 ==> M4
    G2 -- "NO (Rejected 3x)" --> G4["Demote Proxy -> Rotate to Fresh IP"]
    G4 ==> P10

    A3 -- "throttle: 'Logging in too fast'" --> T1["Exponential Throttle Backoff: 600ms to 5s"]
    T1 --> T2["Retain Current Proxy IP -> Redial after Delay"]
    T2 ==> M4

    A3 -- "duplicate: 'Already online'" --> DP1{"Ghost Mode Enabled?"}
    DP1 -- "YES" --> DP2["Ghost Slot Tap: Fast 400ms Same-IP Retry (up to 4x)"]
    DP2 ==> M4
    DP1 -- "NO" --> DP3["Rotate Proxy IP -> Wait 4000ms -> Retry"]
    DP3 ==> P10

    A3 -- "banned: Anti-Bot Punishment" --> BN1["Emit Bannned Event -> Flag Proxy -> Rotate"]
    BN1 ==> P10

    A1 -- "packet.id == 0x02" --> S1["Login Success: Disarm Login Timeout"]
  end

  %% ==========================================
  %% SUBGRAPH 6: SESSION HOLD & LIMBO SURVIVAL
  %% ==========================================
  subgraph HOLD_LOOP["6. SESSION HOLD & PLAY IDENTITY (Session Holding)"]
    S1 --> H1["State -> PLAY (Hold Mode Activated)"]
    H1 --> H2["Transmit Client Settings (id=0x05, bitmask, lang)"]
    H2 --> H3["Transmit Client Brand Plugin Message (id=0x0B, 'vanilla')"]
    H3 --> H4["Arm Play-Idle Watchdog (120s silence timer)"]
    H4 --> H5["Receive Play Packets"]
    H5 --> H6{"Evaluate Play Packet"}
    H6 -- "packet.id == 0x00 (Spawn Entity)" --> H7["Consume Coordinate Doubles & Entity UUID (Ignore)"]
    H6 -- "Payload == 8 Bytes (KeepAlive)" --> H8["Echo KeepAlive Back (Same ID + Data + Compression)"]
    H8 --> H9["Increment Session KA Counter -> Refresh Watchdog"]
    H9 --> H5
    H6 -- "packet.id == 0x17 (Play Disconnect)" --> H10["Parse Play Kick Reason -> Clean Socket Drop"]
    H6 -- "Socket Closed by Server (AuthMe 80s Limbo)" --> H11["Normal Session Cycle -> Instant Re-dial"]
    H11 ==> M4
  end

  %% ==========================================
  %% SUBGRAPH 7: ANTI-BOT PROBE SUBSYSTEM
  %% ==========================================
  subgraph PROBE_SYS["7. RAPID MULTI-IP ANTI-BOT PROBE (probe.js)"]
    PR1["node probe.js --username=target --limit=N"] --> PR2["Install Security Kernel & Pick Verified Working Proxy"]
    PR2 --> PR3["Connect SOCKS5 -> Send Handshake & LoginStart"]
    PR3 --> PR4{"Connection Outcome"}
    PR4 -- "FlameCord Rejoin Gate" --> PR5["Rejoin Same IP with 2000ms Delay (up to 3x)"]
    PR5 --> PR3
    PR4 -- "Login Timeout (5s)" --> PR6["Kill Socket -> Mark Proxy Dead -> Rotate"]
    PR6 --> PR2
    PR4 -- "Server IN PLAY (Success)" --> PR7["Register Server Join Count (#serverJoins)"]
    PR7 --> PR8["Hold 2.0s for Server-Side Anti-Bot Join Event"]
    PR8 --> PR9["Drop Connection Cleanly -> Rotate to Next Fresh IP"]
    PR9 --> PR2
    PR4 -- "Ban Kick / Anti-Bot Punishment" --> PR10["ANTI-BOT BAN DETECTED BANNER"]
    PR10 --> PR11["Dump Telemetry: Total Server Joins Before Ban -> Exit 0"]
  end

  %% STYLING ASSIGNMENTS
  class K1,K2,K3,K5,K6,K7,K8 kernel
  class P1,P2,P3,P4,P5,P7,P8,P9,P10 proxy
  class D1,D2,D3,D4,D5,D6,D7 daemon
  class M1,M2,M3,M4,M5,M6,M8,M9 proto
  class A2,A3,G1,G3,G4,T1,T2,DP2,DP3,BN1 armor
  class S1,H1,H2,H3,H4,H5,H7,H8,H9,H10,H11 hold
  class PR1,PR2,PR3,PR5,PR6,PR7,PR8,PR9,PR10,PR11 probe
  class K4,P6,M7,A1,G2,DP1,H6,PR4 decision
```
