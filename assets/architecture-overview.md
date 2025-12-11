# SWAMP Platform Architecture Diagram

## High-Level System Architecture

```mermaid
graph TB
    subgraph "Field Layer"
        S1[Soil Sensor 1]
        S2[Soil Sensor 2]
        S3[Soil Sensor N]
    end
    
    subgraph "IoT Gateway"
        MQTT[MQTT Broker<br/>HiveMQ/Mosquitto]
    end
    
    subgraph "Cloud Infrastructure - AWS"
        ECS[ECS Fargate<br/>MQTT Bridge Service]
        ALB[Application<br/>Load Balancer]
    end
    
    subgraph "Backend - Supabase"
        PG[(PostgreSQL<br/>Time-series Data)]
        AUTH[Auth Service<br/>Row Level Security]
        EDGE[Edge Functions<br/>Data Processing]
        RT[Realtime<br/>WebSocket Server]
    end
    
    subgraph "Frontend - Vercel"
        WEB[React Dashboard<br/>Vite + TypeScript]
        ZUSTAND[Zustand Store<br/>Real-time State]
    end
    
    subgraph "DevOps"
        GHA[GitHub Actions<br/>CI/CD Pipeline]
        TRIVY[Trivy Scanner<br/>Security Checks]
    end
    
    %% Data Flow
    S1 & S2 & S3 -->|MQTT Pub| MQTT
    MQTT -->|Subscribe| ECS
    ECS -->|Insert Data| EDGE
    EDGE -->|Write| PG
    PG -.->|Realtime Events| RT
    RT -.->|WebSocket| WEB
    WEB -->|REST API| EDGE
    WEB -->|Auth| AUTH
    ZUSTAND -.->|State Sync| WEB
    
    %% CI/CD
    GHA -->|Deploy| ECS
    GHA -->|Deploy| WEB
    TRIVY -.->|Scan| GHA
    
    %% Styling
    classDef iot fill:#e3f2fd,stroke:#1976d2,stroke-width:2px
    classDef cloud fill:#fff3e0,stroke:#f57c00,stroke-width:2px
    classDef backend fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px
    classDef frontend fill:#e8f5e9,stroke:#388e3c,stroke-width:2px
    classDef devops fill:#fce4ec,stroke:#c2185b,stroke-width:2px
    
    class S1,S2,S3,MQTT iot
    class ECS,ALB cloud
    class PG,AUTH,EDGE,RT backend
    class WEB,ZUSTAND frontend
    class GHA,TRIVY devops
```

## Data Flow Explanation

1. **Sensors** publish soil moisture, temperature, and battery data via MQTT
2. **MQTT Broker** receives high-frequency telemetry (1 message/minute per sensor)
3. **ECS Bridge Service** subscribes to topics, validates, and batches data
4. **Edge Functions** normalize data and insert into PostgreSQL with timestamp
5. **PostgreSQL** stores time-series data with partitioning by month
6. **Realtime Server** broadcasts changes to connected web clients
7. **React Dashboard** updates state in real-time using Zustand
8. **CI/CD Pipeline** deploys containerized services with security scanning

## Key Architectural Decisions

### Why MQTT?
- Lightweight protocol for IoT devices with limited bandwidth
- Pub/Sub pattern decouples sensors from backend
- Built-in QoS levels for reliable delivery

### Why Supabase?
- PostgreSQL with built-in auth and real-time capabilities
- Edge Functions eliminate need for separate Node.js API
- Row Level Security for multi-tenancy

### Why ECS Fargate?
- Serverless containers for MQTT bridge (no server management)
- Auto-scaling based on CPU/memory
- Integrates with ALB for health checks

### Why Zustand?
- Lightweight state management (3KB vs Redux 10KB+)
- No context/provider boilerplate
- Optimized for high-frequency updates

## Security Layers

```mermaid
graph LR
    subgraph "Defense in Depth"
        L1[MQTT TLS<br/>Client Certs]
        L2[VPC<br/>Private Subnets]
        L3[RLS<br/>PostgreSQL]
        L4[JWT<br/>Auth Tokens]
        L5[WAF<br/>Rate Limiting]
    end
    
    L1 --> L2 --> L3 --> L4 --> L5
    
    style L1 fill:#ffcdd2
    style L2 fill:#f8bbd0
    style L3 fill:#e1bee7
    style L4 fill:#d1c4e9
    style L5 fill:#c5cae9
```

## Scalability Strategy

| Component | Scaling Method | Target Capacity |
|-----------|----------------|-----------------|
| MQTT Broker | Horizontal (cluster) | 10K+ connections |
| ECS Tasks | Auto-scaling (CPU) | 2-10 tasks |
| PostgreSQL | Vertical + Read Replicas | 100K writes/min |
| Edge Functions | Automatic (Deno Deploy) | Unlimited |
| Frontend | CDN (Vercel Edge) | Global |

---

*This architecture supports 1000+ concurrent sensors with <500ms end-to-end latency*
