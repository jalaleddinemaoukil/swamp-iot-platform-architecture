# API Contracts & TypeScript Interfaces

This document defines the API contracts and TypeScript type definitions for the SWAMP platform.

## Core Data Types

### Sensor Device

```typescript
interface Device {
  id: string;                    // UUID
  deviceId: string;              // Hardware MAC address
  farmId: string;                // Foreign key to farm
  zoneId: string;                // Foreign key to zone
  type: 'soil_moisture' | 'weather' | 'camera';
  status: 'active' | 'offline' | 'maintenance';
  lastSeen: string;              // ISO timestamp
  batteryLevel: number;          // 0-100
  firmwareVersion: string;
  location: {
    lat: number;
    lng: number;
  };
  config: {
    sampleInterval: number;      // Minutes
    thresholds: {
      moistureMin: number;
      moistureMax: number;
      batteryMin: number;
    };
  };
  createdAt: string;
  updatedAt: string;
}
```

### Sensor Reading (Telemetry)

```typescript
interface SensorReading {
  id: string;
  deviceId: string;
  timestamp: string;             // ISO timestamp
  data: {
    soilMoisture?: number;       // Percentage (0-100)
    temperature?: number;        // Celsius
    humidity?: number;           // Percentage (0-100)
    batteryVoltage?: number;     // Volts
    signalStrength?: number;     // dBm
  };
  metadata?: {
    messageId: string;
    receivedAt: string;
    mqttTopic: string;
  };
}
```

### Farm & Zone

```typescript
interface Farm {
  id: string;
  organizationId: string;
  name: string;
  location: string;
  totalArea: number;             // Hectares
  timezone: string;
  createdAt: string;
}

interface Zone {
  id: string;
  farmId: string;
  name: string;
  cropType?: string;
  area: number;                  // Hectares
  irrigationType?: 'drip' | 'sprinkler' | 'flood';
  deviceCount: number;
}
```

### Alert

```typescript
interface Alert {
  id: string;
  deviceId: string;
  farmId: string;
  type: 'low_moisture' | 'low_battery' | 'device_offline' | 'anomaly';
  severity: 'info' | 'warning' | 'critical';
  message: string;
  triggeredAt: string;
  acknowledgedAt?: string;
  acknowledgedBy?: string;
  resolved: boolean;
  resolvedAt?: string;
}
```

## REST API Endpoints

### Device Management

#### Get All Devices
```http
GET /api/devices?farmId={farmId}&status={status}

Headers:
  Authorization: Bearer {jwt_token}

Response: 200 OK
{
  "devices": Device[],
  "total": number,
  "page": number,
  "pageSize": number
}
```

#### Get Device by ID
```http
GET /api/devices/{deviceId}

Headers:
  Authorization: Bearer {jwt_token}

Response: 200 OK
{
  "device": Device
}
```

#### Update Device Configuration
```http
PATCH /api/devices/{deviceId}

Headers:
  Authorization: Bearer {jwt_token}
  Content-Type: application/json

Body:
{
  "config": {
    "sampleInterval": 15,
    "thresholds": {
      "moistureMin": 30,
      "moistureMax": 80,
      "batteryMin": 20
    }
  }
}

Response: 200 OK
{
  "device": Device
}
```

### Telemetry Data

#### Get Recent Readings
```http
GET /api/telemetry?deviceId={deviceId}&start={iso_date}&end={iso_date}&limit={number}

Headers:
  Authorization: Bearer {jwt_token}

Response: 200 OK
{
  "readings": SensorReading[],
  "total": number,
  "hasMore": boolean
}
```

#### Get Aggregated Data
```http
GET /api/telemetry/aggregate?deviceId={deviceId}&interval=1h&metric=avg

Headers:
  Authorization: Bearer {jwt_token}

Response: 200 OK
{
  "data": [
    {
      "timestamp": "2025-12-11T10:00:00Z",
      "avgSoilMoisture": 65.5,
      "avgTemperature": 22.3
    }
  ]
}
```

### Farm & Zone Management

#### Get Farms
```http
GET /api/farms?organizationId={orgId}

Headers:
  Authorization: Bearer {jwt_token}

Response: 200 OK
{
  "farms": Farm[]
}
```

#### Get Zones for Farm
```http
GET /api/farms/{farmId}/zones

Headers:
  Authorization: Bearer {jwt_token}

Response: 200 OK
{
  "zones": Zone[]
}
```

### Alerts

#### Get Active Alerts
```http
GET /api/alerts?farmId={farmId}&resolved=false

Headers:
  Authorization: Bearer {jwt_token}

Response: 200 OK
{
  "alerts": Alert[],
  "total": number
}
```

#### Acknowledge Alert
```http
POST /api/alerts/{alertId}/acknowledge

Headers:
  Authorization: Bearer {jwt_token}

Response: 200 OK
{
  "alert": Alert
}
```

## MQTT Topics & Payloads

### Device to Cloud (Publish)

#### Sensor Data
```
Topic: swamp/devices/{deviceId}/telemetry

Payload:
{
  "timestamp": "2025-12-11T10:30:00Z",
  "soilMoisture": 65.5,
  "temperature": 22.3,
  "batteryVoltage": 3.7,
  "signalStrength": -72
}
```

#### Device Status
```
Topic: swamp/devices/{deviceId}/status

Payload:
{
  "status": "online",
  "firmwareVersion": "1.2.3",
  "uptime": 3600
}
```

### Cloud to Device (Subscribe)

#### Configuration Update
```
Topic: swamp/devices/{deviceId}/config

Payload:
{
  "sampleInterval": 15,
  "thresholds": {
    "moistureMin": 30,
    "moistureMax": 80
  }
}
```

#### Command
```
Topic: swamp/devices/{deviceId}/command

Payload:
{
  "command": "restart" | "calibrate" | "update_firmware",
  "params": {}
}
```

## WebSocket Events (Realtime)

### Subscribe to Device Updates
```typescript
// Client subscribes to channel
const channel = supabase
  .channel('device-telemetry')
  .on('postgres_changes', {
    event: 'INSERT',
    schema: 'public',
    table: 'telemetry',
    filter: `device_id=eq.${deviceId}`
  }, (payload) => {
    console.log('New reading:', payload.new);
  })
  .subscribe();
```

### Realtime Payload Format
```typescript
interface RealtimePayload {
  schema: 'public';
  table: string;
  commit_timestamp: string;
  eventType: 'INSERT' | 'UPDATE' | 'DELETE';
  new: SensorReading;      // For INSERT/UPDATE
  old: SensorReading;      // For UPDATE/DELETE
}
```

## Error Responses

### Standard Error Format
```typescript
interface ApiError {
  error: {
    code: string;
    message: string;
    details?: any;
  };
  status: number;
}
```

### Common Error Codes
```typescript
// 400 Bad Request
{
  "error": {
    "code": "INVALID_PARAMS",
    "message": "deviceId is required"
  },
  "status": 400
}

// 401 Unauthorized
{
  "error": {
    "code": "UNAUTHORIZED",
    "message": "Invalid or expired token"
  },
  "status": 401
}

// 403 Forbidden
{
  "error": {
    "code": "FORBIDDEN",
    "message": "Access denied to this resource"
  },
  "status": 403
}

// 404 Not Found
{
  "error": {
    "code": "NOT_FOUND",
    "message": "Device not found"
  },
  "status": 404
}

// 429 Too Many Requests
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Too many requests, please try again later"
  },
  "status": 429
}

// 500 Internal Server Error
{
  "error": {
    "code": "INTERNAL_ERROR",
    "message": "An unexpected error occurred"
  },
  "status": 500
}
```

## Rate Limiting

- **API Endpoints:** 100 requests per minute per user
- **MQTT Publish:** 1 message per second per device
- **WebSocket Connections:** 10 concurrent connections per user

## Authentication

All API requests require a JWT token in the `Authorization` header:

```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

Tokens are obtained via Supabase Auth:
```typescript
const { data, error } = await supabase.auth.signInWithPassword({
  email: 'user@example.com',
  password: 'password'
});

const token = data.session.access_token;
```

---

*These contracts define the interface between frontend, backend, and IoT devices.*
