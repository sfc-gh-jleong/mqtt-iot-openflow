# mqtt-iot-openflow

OpenFlow NiFi flow definitions for an end-to-end IoT battery telemetry pipeline built on Snowflake.

## Architecture Overview

```
[ mqtt-publisher runtime ]          [ mqtt-subscriber runtime ]
  MQTT Publisher flow                 Battery MQTT Telemetry flow
  GenerateFlowFile                    ConsumeMQTT
       ↓                                   ↓
  UpdateAttribute                    Flatten Payload (Jolt)
       ↓                                   ↓
  Publish to HiveMQ  ──── MQTT ────► PublishSnowpipeStreaming v2
                                           ↓
                              SNOWFLAKE_DEMO.MQTT.MQTT_TELEMETRY
```

Both runtimes connect to the same **HiveMQ Cloud** MQTT broker over TLS (port 8883). The publisher generates synthetic IoT battery telemetry every 5 seconds; the subscriber consumes it and lands it in Snowflake for downstream analytics.

---

## Folders

### `mqtt-publisher/`

**Runtime:** `mqttpublisher` (Snowflake OpenFlow SPCS runtime)

**Flow:** *MQTT Publisher*

Generates synthetic IoT battery telemetry and publishes it to HiveMQ Cloud on topic `battery/telemetry`.

**Processors:**
| Processor | Role |
|-----------|------|
| `Generate Telemetry` (GenerateFlowFile) | Produces a JSON battery record every 5 seconds using NiFi EL for randomised values across 5 batteries and 3 devices |
| `Set MQTT Topic` (UpdateAttribute) | Sets the `mqtt.topic` attribute to `battery/telemetry` |
| `Publish to HiveMQ` (PublishMQTT) | Publishes the FlowFile content to HiveMQ Cloud over TLS (ssl://) |

**Generated payload example:**
```json
{
  "BATTERY_ID":             "BAT-003",
  "DEVICE_ID":              "DEVICE-02",
  "TIMESTAMP_MS":           "1745440000000",
  "VOLTAGE_MV":             3720,
  "CURRENT_MA":             2500,
  "TEMPERATURE_C":          28,
  "STATE_OF_CHARGE_PCT":    75,
  "STATE_OF_HEALTH_PCT":    93,
  "CYCLE_COUNT":            142,
  "CAPACITY_REMAINING_MAH": 3800,
  "IS_CHARGING":            "false",
  "ERROR_CODE":             "NONE"
}
```

**Field reference:**
| Field | Unit | Range |
|-------|------|-------|
| `VOLTAGE_MV` | millivolts | 2500–4200 (Li-ion) |
| `CURRENT_MA` | milliamps (positive = discharging) | 0–5000 |
| `TEMPERATURE_C` | Celsius | 15–45 |
| `STATE_OF_CHARGE_PCT` | % | 0–100 |
| `STATE_OF_HEALTH_PCT` | % | 80–100 |
| `CYCLE_COUNT` | count | 0–500 |
| `CAPACITY_REMAINING_MAH` | milliamp-hours | 2500–5000 |

---

### `mqtt-subscriber/`

**Runtime:** `mqtt` (Snowflake OpenFlow SPCS runtime)

**Flow:** *Battery MQTT Telemetry*

Subscribes to HiveMQ Cloud on topic `battery/telemetry/#`, transforms the incoming messages, and streams them into Snowflake via Snowpipe Streaming v2.

**Processors:**
| Processor | Role |
|-----------|------|
| `ConsumeMQTT - HiveMQ` (ConsumeMQTT) | Subscribes to HiveMQ Cloud and receives battery telemetry messages |
| `Flatten Payload` (JoltTransformJSON) | Shifts JSON field names from the publisher format to the Snowflake column names |
| `PublishSnowpipeStreaming - Telemetry` (PublishSnowpipeStreaming v2) | Streams transformed records directly into Snowflake via the Snowpipe Streaming API |

**Snowflake destination:** `SNOWFLAKE_DEMO.MQTT.MQTT_TELEMETRY`

Authentication uses `SNOWFLAKE_MANAGED` (SPCS session token) — no key-pair credentials required.

---

## Infrastructure

| Component | Details |
|-----------|---------|
| MQTT Broker | HiveMQ Cloud (TLS, port 8883) |
| Publisher runtime | `mqttpublisher` — Snowflake OpenFlow SPCS |
| Subscriber runtime | `mqtt` — Snowflake OpenFlow SPCS |
| EAI | `HIVEMQ_EAI` — covers HiveMQ Cloud and `api.github.com` endpoints |
| Network rule | `OPENFLOW.OPENFLOW.MQTT_GITHUB_REGISTRY_NETWORK_RULE` |
| Snowflake table | `SNOWFLAKE_DEMO.MQTT.MQTT_TELEMETRY` |
| Snowflake stage | `SNOWFLAKE_DEMO.MQTT.MQTT_INGEST_STAGE` |
| Snowflake pipe | `SNOWFLAKE_DEMO.MQTT.MQTT_INGEST_PIPE` |
