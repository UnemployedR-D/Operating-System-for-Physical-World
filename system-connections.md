# System Connections to AI Operational Supervisor

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        AI OPERATIONAL SUPERVISOR                             │
│                         (Central Hub)                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐  │
│   │   ADAPTERS  │    │  FACILITY   │    │INTELLIGENCE │    │   OUTPUTS   │  │
│   │   (Ingest)  │───▶│   STATE     │───▶│   LAYER     │───▶│  (Act/Notify)│  │
│   └─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘  │
│        │                                        │                  │        │
│        ▼                                        ▼                  ▼        │
│   ┌─────────────┐                         ┌─────────────┐    ┌─────────────┐  │
│   │  NORMALIZE  │                         │  EXCEPTION  │    │  SLACK/     │  │
│   │  TO CANONICAL│                        │  ENGINE +   │    │  TEAMS/     │  │
│   │   TYPES      │                        │  LLM REASON │    │  SMS/       │  │
│   └─────────────┘                         └─────────────┘    │  MOBILE     │  │
│                                                         │    │  DEEP LINKS │  │
│                                                         ▼    └─────────────┘  │
│                                                ┌─────────────┐                │
│                                                │ GROUND TRUTH│                │
│                                                │  CAPTURE    │                │
│                                                └──────┬──────┘                │
│                                                       │                      │
│                                                       ▼                      │
│                                                ┌─────────────┐               │
│                                                │ EVENT LOG   │               │
│                                                │ + LEARNING  │               │
│                                                └─────────────┘               │
└─────────────────────────────────────────────────────────────────────────────┘
                                      ▲
          ┌───────────────────────────┼───────────────────────────┐
          │                           │                           │
          ▼                           ▼                           ▼
┌─────────────────────┐     ┌─────────────────────┐     ┌─────────────────────┐
│      TIER 0         │     │      TIER 1a        │     │      TIER 1b        │
│      WMS            │     │    ROBOT FLEET      │     │      LABOR          │
│                     │     │                     │     │                     │
│  ShipHero           │     │  Locus / VDA 5050   │     │  Worker Mgmt Sys    │
│  Manhattan          │     │  Fetch / 6 River    │     │  (UKG, Kronos,      │
│  Blue Yonder        │     │  MiR / OTTO         │     │   custom)           │
│  HighJump           │     │  Custom AGV         │     │                     │
│                     │     │                     │     │                     │
│ COMMUNICATION:      │     │ COMMUNICATION:      │     │ COMMUNICATION:      │
│  • REST API (poll)  │     │  • REST API (poll)  │     │  • REST API (poll)  │
│  • Webhooks         │     │  • Webhooks         │     │  • Webhooks         │
│  • GraphQL (some)   │     │  • MQTT (some)      │     │  • gRPC (some)      │
│  • SFTP (batch)     │     │  • WebSocket        │     │                     │
│                     │     │  • VDA 5050 std     │     │                     │
│ DATA:               │     │ DATA:               │     │ DATA:               │
│  • Waves/Orders     │     │  • Robot poses      │     │  • Worker location  │
│  • Inventory        │     │  • Status/mission   │     │  • Certifications   │
│  • Locations        │     │  • Events (obstacle)│     │  • Shift/tasks      │
│  • Pick/Pack        │     │  • Battery/charge   │     │  • Time/attendance  │
└─────────────────────┘     └─────────────────────┘     └─────────────────────┘
          │                           │                           │
          │                    ┌──────┴──────┐                    │
          │                    ▼             ▼                    │
          │            ┌─────────────┐ ┌─────────────┐           │
          │            │   TIER 1c   │ │   TIER 1d   │           │
          │            │    CMMS     │ │   CAMERAS   │           │
          │            │             │ │             │           │
          │            │  Fiix       │ │  Verkada    │           │
          │            │  UpKeep     │ │  Rhombus    │           │
          │            │  eMaint     │ │  Axis       │           │
          │            │  MaintainX  │ │  Custom CV  │           │
          │            │             │ │             │           │
          │            │ COMMUNICATION:│ │ COMMUNICATION:│          │
          │            │  • REST API │ │  • REST API │           │
          │            │  • Webhooks │ │  • Webhooks │           │
          │            │             │ │  • RTSP/WebRTC│          │
          │            │ DATA:       │ │  • Events   │           │
          │            │  • Work orders│ │   (person, │           │
          │            │  • Tech sched │ │   motion,  │           │
          │            │  • Asset hist │ │   stopped) │           │
          │            └─────────────┘ └─────────────┘           │
          │                                                       │
          │                    ┌─────────────────────┐           │
          │                    │      TIER 2         │           │
          │                    │   EQUIPMENT/PLC     │           │
          │                    │                     │           │
          │                    │  OPC UA             │           │
          │                    │  Modbus TCP         │           │
          │                    │  EtherNet/IP        │           │
          │                    │  MQTT Sparkplug B   │           │
          │                    │                     │           │
          │                    │ COMMUNICATION:      │           │
          │                    │  • OPC UA (sub)     │           │
          │                    │  • Modbus (poll)    │           │
          │                    │  • MQTT (sub)       │           │
          │                    │  • Edge gateway     │           │
          │                    │                     │           │
          │                    │ DATA:               │           │
          │                    │  • Conveyor speed   │           │
          │                    │  • Motor current    │           │
          │                    │  • Sensor states    │           │
          │                    │  • Fault codes      │           │
          │                    └─────────────────────┘           │
          │                                                       │
          └───────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                         NOTIFICATION CHANNELS (Bidirectional)                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐       │
│   │   SLACK     │  │   TEAMS     │  │    SMS      │  │   MOBILE    │       │
│   │  (Primary)  │  │  (Alt)      │  │  (Critical) │  │   PWA       │       │
│   │             │  │             │  │             │  │  (Ground    │       │
│   │ • Blocks UI │  │ • Adaptive  │  │ • Twilio/   │  │   Truth)    │       │
│   │ • Deep links│  │   Cards     │  │   Telnyx    │  │             │       │
│   │ • Actions   │  │ • Actions   │  │ • High pri  │  │ • Offline   │       │
│   │ • Threads   │  │             │  │   only      │  │   capable   │       │
│   └─────────────┘  └─────────────┘  └─────────────┘  └─────────────┘       │
│                                                                              │
│   COMMUNICATION: HTTPS Webhooks (outbound) + Deep Links (inbound)           │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Communication Pattern Summary

| System Tier | Examples | Primary Protocol | Polling | Webhooks/Stream | Data Direction |
|-------------|----------|------------------|---------|-----------------|----------------|
| **Tier 0** | ShipHero, Manhattan, Blue Yonder | REST + Webhooks | 30-60s | Yes (wave, pick, inventory) | In → Supervisor |
| **Tier 1a** | Locus, Fetch, MiR, VDA 5050 | REST + Webhooks + MQTT | 10-30s | Yes (obstacle, status) | In → Supervisor |
| **Tier 1b** | UKG, Kronos, Custom | REST + Webhooks | 30-60s | Yes (task, shift) | In → Supervisor, Out ← Dispatch |
| **Tier 1c** | Fiix, UpKeep, MaintainX | REST + Webhooks | 60s | Yes (WO, tech) | In → Supervisor, Out ← Dispatch |
| **Tier 1d** | Verkada, Rhombus, Axis | REST + Webhooks + RTSP | — | Yes (events) | In → Supervisor |
| **Tier 2** | Conveyors, Sorters, PLCs | OPC UA / Modbus / MQTT | 1-5s | Yes (subscriptions) | In → Supervisor |
| **Notifications** | Slack, Teams, SMS, PWA | HTTPS Webhooks | — | N/A | Bidirectional |

## Key Principles

1. **Adapter per vendor** — Each external system gets a dedicated adapter that normalizes to canonical types
2. **Polling as baseline** — Reliable, simple, works everywhere. Webhooks for latency-critical paths
3. **Canonical model** — Internal FacilityState never sees vendor-specific formats
4. **Bidirectional for action** — Labor, CMMS, Robot fleets receive dispatch commands back
5. **Edge gateway for Tier 2** — PLC/equipment data often needs on-premise gateway (OPC UA → MQTT/HTTPS)
