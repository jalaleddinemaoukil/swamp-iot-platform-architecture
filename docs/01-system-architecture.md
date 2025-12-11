# System Architecture

## High-Level Data Flow

The architecture is designed to decouple the **Ingestion Layer** (high volume, fast writes) from the **Application Layer** (complex reads, user interaction).

```mermaid
graph TD
    %% Nodes
    Sensor([IoT Sensor Node])
    Gateway[LoRaWAN Gateway]
    MQTT[MQTT Broker]
    Bridge[Node.js Ingestion Bridge]
    
    subgraph Cloud [Supabase Ecosystem]
        Edge[Edge Function: Normalize]
        DB[(PostgreSQL)]
        Auth[Auth Service]
    end
    
    subgraph Client [Frontend App]
        Dashboard[React Dashboard]
        Zustand[State Store]
    end

    %% Edges
    Sensor -->|LoRa/WiFi| Gateway
    Gateway -->|MQTT Pub| MQTT
    MQTT -->|Sub| Bridge
    Bridge -->|POST /ingest| Edge
    Edge -->|Insert| DB
    
    DB -.->|Realtime Event| Dashboard
    Dashboard -->|Fetch History| Edge
    Dashboard -->|Login| Auth
    
    style Sensor fill:#e3f2fd
    style MQTT fill:#fff3e0
    style Bridge fill:#fff3e0
    style DB fill:#f3e5f5
    style Dashboard fill:#e8f5e9
```

## Component Layers

### 1. Device Layer
**Sensors** collect environmental data (soil moisture, temperature, battery level) and publish to MQTT broker via:
- **LoRaWAN:** Long-range, low-power for remote areas
- **WiFi:** Higher bandwidth for on-farm deployments
- **Cellular (4G/LTE):** Backup connectivity

**MQTT Topics:**
```
swamp/devices/{deviceId}/telemetry    # Sensor data
swamp/devices/{deviceId}/status       # Heartbeat/online status
swamp/devices/{deviceId}/config       # Configuration updates (cloud → device)
```

### 2. Ingestion Layer
**MQTT Broker** (HiveMQ/Mosquitto):
- Handles 1000+ concurrent connections
- QoS 1 (at least once delivery)
- TLS encryption with client certificates

**Node.js Bridge Service:**
- Subscribes to all device topics
- Validates message format
- Batches writes (5-second window)
- Forwards to Supabase Edge Functions

### 3. Backend Layer (Supabase)
**PostgreSQL 15:**
- Time-series telemetry data (partitioned by month)
- Row Level Security for multi-tenancy
- Triggers for alert generation

**Edge Functions (Deno):**
- `/api/ingest` - Receive data from MQTT bridge
- `/api/devices` - Device management CRUD
- `/api/telemetry` - Query historical data
- `/api/alerts` - Alert management

**Auth Service (GoTrue):**
- JWT-based authentication
- Email/password + OAuth
- Role-based access control

### 4. Frontend Layer
**React 18 + Vite:**
- Fast build times (<2s)
- Hot module replacement
- TypeScript for type safety

**Zustand State Management:**
- Lightweight (3KB vs Redux 45KB)
- No context/provider boilerplate
- Selective subscriptions

---

*This architecture supports 1000+ devices with <500ms latency.*
