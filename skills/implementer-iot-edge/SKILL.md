---
name: implementer-iot-edge
description: Use when implementing IoT edge tasks involving MQTT, BLE, or device connectivity with reliability and operational safety requirements.
---

# Implementer IoT Edge

Apply this skill for IoT edge and connectivity implementation tasks.

## Required Rules

- Make protocol behavior explicit (QoS, retries, reconnect, timeout).
- Design for unstable networks and partial failure.
- Preserve schema compatibility and message version safety.
- Include telemetry hooks needed for field diagnostics.
- Never hardcode broker URLs, credentials, or device identifiers.

## Minimum Quality Gates

- Validate changed connectivity behavior with evidence.
- Document reliability and security impact of changes.
- Confirm rollback-safe behavior for protocol/schema updates.
- Test reconnect and partial-failure paths explicitly.

## Output Expectations

- Changed files with rationale
- Validation and verification evidence
- Reliability/security risks and follow-up items

---

## Architecture

### Layered IoT Architecture

```
src/
├── connectivity/
│   ├── mqtt/
│   │   ├── MqttClient.ts        # Connection lifecycle, reconnect, publish/subscribe
│   │   ├── MqttTopicRegistry.ts # Centralized topic definitions — no magic strings
│   │   └── MqttMessageParser.ts # Schema validation + version routing
│   └── ble/
│       ├── BleManager.ts        # Scan, connect, disconnect, GATT operations
│       ├── BleProfile.ts        # GATT service/characteristic UUIDs
│       └── BleMessageCodec.ts   # Encode/decode byte arrays
├── telemetry/
│   └── TelemetryService.ts      # Structured diagnostic logging
├── store/                       # Device state — cached, survives reconnect
└── features/
    └── device-control/
        ├── DeviceCommandService.ts
        └── DeviceStatusService.ts
```

**Rule:** Connectivity layer must be **completely independent of UI**. UI consumes device state from store — never subscribes directly to MQTT/BLE.

---

## MQTT Rules

### QoS Level Selection

| QoS | Guarantee | When to use |
|---|---|---|
| 0 (At most once) | Fire and forget — no delivery guarantee | Telemetry/sensor readings where loss is acceptable |
| 1 (At least once) | Delivered at least once — duplicates possible | Commands, status updates — must use idempotent handlers |
| 2 (Exactly once) | Delivered exactly once — highest overhead | Financial/billing events, critical state changes |

```typescript
// ✅ Be explicit about QoS per message type
const PUBLISH_QOS = {
  sensorReading: 0,    // loss acceptable
  deviceCommand: 1,    // must arrive, handle duplicates
  firmwareUpdate: 2,   // exactly once — critical
} as const;

await client.publishAsync(topic, payload, { qos: PUBLISH_QOS.deviceCommand });
```

### Topic Naming Convention

```typescript
// ✅ Centralized registry — no magic strings anywhere else
export const MqttTopics = {
  // Pattern: {tenant}/{deviceId}/{direction}/{messageType}
  telemetry: (deviceId: string) => `devices/${deviceId}/telemetry`,
  command: (deviceId: string) => `devices/${deviceId}/commands/inbound`,
  commandAck: (deviceId: string) => `devices/${deviceId}/commands/ack`,
  status: (deviceId: string) => `devices/${deviceId}/status`,
  firmware: (deviceId: string) => `devices/${deviceId}/firmware/update`,
} as const;

// ✅ Wildcard subscriptions — document scope explicitly
const ALL_DEVICE_TELEMETRY = 'devices/+/telemetry';  // + = single level
const ALL_COMMANDS = 'devices/#';                     // # = multi level (use carefully)
```

### Retain Flag

```typescript
// ✅ Retain = true — broker stores last message, new subscribers receive it immediately
// Use for: device status, last-known state
await client.publishAsync(
  MqttTopics.status(deviceId),
  JSON.stringify(statusPayload),
  { qos: 1, retain: true }  // last status available to new subscribers
);

// ✅ Retain = false — ephemeral, use for commands and one-time events
await client.publishAsync(
  MqttTopics.command(deviceId),
  JSON.stringify(command),
  { qos: 1, retain: false }
);
```

### Last Will & Testament

```typescript
// ✅ Configure LWT at connection time — broker publishes if client disconnects ungracefully
const client = mqtt.connect(brokerUrl, {
  clientId: `edge-${deviceId}-${Date.now()}`,
  will: {
    topic: MqttTopics.status(deviceId),
    payload: JSON.stringify({ online: false, reason: 'unexpected_disconnect' }),
    qos: 1,
    retain: true,
  }
});
```

### Reconnection (Exponential Backoff)

```typescript
// ✅ Managed reconnect — never tight loop
class MqttClient {
  private retryCount = 0;
  private readonly MAX_RETRIES = 10;
  private readonly BASE_DELAY_MS = 1000;
  private readonly MAX_DELAY_MS = 30_000;

  private getBackoffDelay(): number {
    const delay = Math.min(
      this.BASE_DELAY_MS * Math.pow(2, this.retryCount),
      this.MAX_DELAY_MS
    );
    return delay + Math.random() * 1000; // jitter — avoid thundering herd
  }

  private onDisconnect() {
    if (this.retryCount >= this.MAX_RETRIES) {
      this.emit('fatal', new Error('Max reconnect attempts reached'));
      return;
    }
    const delay = this.getBackoffDelay();
    this.retryCount++;
    setTimeout(() => this.connect(), delay);
  }

  private onConnected() {
    this.retryCount = 0; // reset on successful connect
    this.resubscribeAll(); // re-subscribe after reconnect
  }
}
```

### Persistent Sessions

```typescript
// ✅ Use clean: false for devices that must not miss messages while offline
const client = mqtt.connect(brokerUrl, {
  clientId: `device-${deviceId}`,  // stable client ID required for persistence
  clean: false,                     // broker queues QoS 1/2 messages during disconnect
});

// ✅ Use clean: true for monitoring dashboards — always fresh state
const dashboardClient = mqtt.connect(brokerUrl, {
  clientId: `dashboard-${sessionId}`,
  clean: true,
});
```

---

## BLE Rules

### Connection Lifecycle

```typescript
// ✅ Always manage the full scan → connect → discover → use → disconnect lifecycle
class BleManager {
  async connectDevice(deviceId: string): Promise<BleDevice> {
    // 1. Scan with timeout
    const device = await this.scanForDevice(deviceId, { timeout: 10_000 });

    // 2. Connect with retry
    await this.connectWithRetry(device, { maxAttempts: 3 });

    // 3. Discover services — cache GATT profile after first discovery
    const services = await device.discoverServicesAndCharacteristics();

    // 4. Negotiate MTU for larger payloads
    await device.requestMTU(512);

    return device;
  }

  async disconnectDevice(device: BleDevice): Promise<void> {
    await device.cancelConnection();
    this.removeFromCache(device.id);
  }
}
```

### GATT Profile (Centralized UUIDs)

```typescript
// ✅ Never hardcode UUIDs inline — centralize in profile definition
export const BleProfile = {
  SERVICE: {
    SENSOR: '12345678-0000-1000-8000-00805f9b34fb',
    OTA: 'abcdef01-0000-1000-8000-00805f9b34fb',
  },
  CHARACTERISTIC: {
    SENSOR_DATA: '12345678-0001-1000-8000-00805f9b34fb',
    SENSOR_CONFIG: '12345678-0002-1000-8000-00805f9b34fb',
    OTA_CONTROL: 'abcdef01-0001-1000-8000-00805f9b34fb',
    OTA_DATA: 'abcdef01-0002-1000-8000-00805f9b34fb',
  }
} as const;
```

### Characteristic Subscription + Cleanup

```typescript
// ✅ Always unsubscribe on unmount/disconnect
class SensorSubscription {
  private subscription: Subscription | null = null;

  start(device: BleDevice, onData: (reading: SensorReading) => void) {
    this.subscription = device
      .monitorCharacteristicForService(
        BleProfile.SERVICE.SENSOR,
        BleProfile.CHARACTERISTIC.SENSOR_DATA
      )
      .subscribe({
        next: (char) => {
          const raw = char?.value;
          if (!raw) return;
          const reading = BleMessageCodec.decodeSensorReading(raw);
          onData(reading);
        },
        error: (err) => this.handleBleError(err),
      });
  }

  stop() {
    this.subscription?.unsubscribe();
    this.subscription = null;
  }
}
```

---

## Schema Versioning

All MQTT and BLE payloads MUST include a version field. Breaking changes require a new version, not in-place modification.

```typescript
// ✅ Versioned message structure
interface DeviceMessage {
  version: '1' | '2';  // explicit version union — new version = new union member
  deviceId: string;
  timestamp: string;   // ISO 8601
  payload: SensorPayloadV1 | SensorPayloadV2;
}

// ✅ Version router — handle multiple versions during migration
function parseSensorMessage(raw: unknown): SensorReading {
  const msg = validateMessage(raw); // validate structure first
  switch (msg.version) {
    case '1': return SensorPayloadV1.parse(msg.payload);
    case '2': return SensorPayloadV2.parse(msg.payload);
    default: throw new Error(`Unknown message version: ${(msg as any).version}`);
  }
}

// ✅ Backward-compatible changes (safe — no version bump needed):
// - Add optional field with default
// - Extend enum with new value (if consumers handle unknown gracefully)

// ❌ Breaking changes (require new version):
// - Remove or rename existing field
// - Change field type
// - Remove enum value
```

---

## Security

### Transport Security

```typescript
// ✅ Always TLS in production — use mqtts:// not mqtt://
const BROKER_URL = process.env.MQTT_BROKER_URL; // never hardcode
// Example: 'mqtts://broker.example.com:8883'

const client = mqtt.connect(BROKER_URL, {
  ca: fs.readFileSync('./certs/ca.crt'),      // CA certificate
  cert: fs.readFileSync('./certs/client.crt'), // client certificate (mTLS)
  key: fs.readFileSync('./certs/client.key'),  // client private key
  rejectUnauthorized: true,                    // never set to false in production
});
```

### Credentials

```typescript
// ✅ Credentials from environment — never in source code
const client = mqtt.connect(process.env.MQTT_BROKER_URL!, {
  username: process.env.MQTT_USERNAME,
  password: process.env.MQTT_PASSWORD,
});

// ✅ Validate payload size — protect against malicious payloads
const MAX_PAYLOAD_BYTES = 256 * 1024; // 256 KB
if (Buffer.byteLength(payload) > MAX_PAYLOAD_BYTES) {
  throw new Error('Payload exceeds size limit');
}
```

---

## Telemetry (Required for Field Diagnosis)

Every device event log must include these fields:

```typescript
interface TelemetryEvent {
  timestamp: string;         // ISO 8601 — always UTC
  deviceId: string;
  firmwareVersion: string;
  eventType: string;         // e.g. 'mqtt.reconnect', 'ble.disconnect', 'sensor.error'
  rssi?: number;             // signal strength for BLE/WiFi
  errorCode?: string;        // machine-readable error identifier
  durationMs?: number;       // for timed operations
  metadata?: Record<string, unknown>;
}

// ✅ Log connectivity events — critical for field diagnosis
function onMqttReconnect(attempt: number) {
  telemetry.log({
    timestamp: new Date().toISOString(),
    deviceId,
    firmwareVersion: APP_VERSION,
    eventType: 'mqtt.reconnect',
    metadata: { attempt, brokerUrl: maskedBrokerUrl },
  });
}
```

---

## Anti-Patterns to Avoid

| Anti-Pattern | Why | Fix |
|---|---|---|
| Hardcoded broker URL / credentials | Security risk, undeployable | Environment variables only |
| QoS 0 for commands | Commands can be lost | Use QoS 1 with idempotent handlers |
| Tight reconnect loop (no backoff) | Floods broker, drains battery | Exponential backoff + jitter |
| Subscribing in UI components directly | Tight coupling, hard to test | Publish to store, UI reads store |
| Magic string topic names | Typos, no autocomplete | MqttTopics registry |
| Missing LWT | Broker doesn't know client is offline | Configure will at connect time |
| No BLE subscription cleanup | Crash, battery drain | Always call `subscription.unsubscribe()` |
| Payload without version field | Breaking changes break old devices | Always version message contracts |
| `rejectUnauthorized: false` | Man-in-the-middle attack | Never disable TLS verification |
| Blocking reconnect on main thread | UI freeze | Async reconnect with state machine |

---

## Verification Matrix

| Gate | How to verify | Expected |
|---|---|---|
| Connectivity | Connect to mock broker (e.g. `aedes` locally) | Publish/subscribe roundtrip succeeds |
| Reconnect | Kill broker mid-session, restore it | Client reconnects, resubscribes |
| Schema validation | Send malformed payload | Parser rejects cleanly, logs error |
| QoS delivery | QoS 1: disconnect mid-publish, reconnect | Message delivered exactly, handler idempotent |
| Backoff | Trigger repeated disconnect | Delay grows, no tight loop in logs |
| Security | Run without TLS config | Connection refused or error thrown |
| BLE cleanup | Unmount component while connected | Subscription logs `unsubscribed`, no listener leak |
| Rollback safety | Deploy old firmware, new schema | Version router falls through gracefully |
