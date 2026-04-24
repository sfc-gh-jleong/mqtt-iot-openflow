# mqtt-iot-openflow

OpenFlow NiFi flow definitions for an end-to-end factory OEE monitoring pipeline built on Snowflake.

## Architecture Overview

```
[ mqtt-publisher runtime ]            [ mqtt-subscriber runtime ]
  MQTT Publisher flow                   Battery MQTT Telemetry flow
  Generate Telemetry (GFF)              ConsumeMQTT - HiveMQ
       |                                     |
  Set MQTT Topic (UpdateAttr)          PublishSnowpipeStreaming v2
       |                                     |
  Publish to HiveMQ ---MQTT--->   SNOWFLAKE_DEMO.MQTT_OEE.MACHINE_TELEMETRY
  topic: factory/oee                         |
                                    DT_MACHINE_LATEST  (DOWNSTREAM)
                                         |          |
                               DT_OEE_5MIN_STATS   DT_OEE_HEALTH_MONITOR
                               (DOWNSTREAM)        (1 minute)
```

Both runtimes connect to the same **HiveMQ Cloud** MQTT broker over TLS (port 8883). The publisher generates synthetic factory machine telemetry every 5 seconds; the subscriber consumes it and streams it into Snowflake via Snowpipe Streaming v2 for downstream OEE analytics.

---

## Folders

| Folder | Runtime | Flow | Purpose |
|--------|---------|------|---------|
| [`mqtt-publisher/`](mqtt-publisher/) | `mqttpublisher` | *MQTT Publisher* | Generate and publish synthetic machine telemetry |
| [`mqtt-subscriber/`](mqtt-subscriber/) | `mqtt` | *Battery MQTT Telemetry* | Subscribe, ingest, and stream to Snowflake |

Each folder contains:
- `README.md` — detailed documentation for that runtime
- `flow.json` — exported NiFi process group definition (for import/restore)

---

## Publisher — `mqtt-publisher/`

**Runtime:** `mqttpublisher` | **Topic:** `factory/oee`

Processors: `Generate Telemetry` → `Set MQTT Topic` → `Publish to HiveMQ`

**Generated payload example:**
```json
{
  "MACHINE_ID":      "MCH-003",
  "LINE_ID":         "LINE-02",
  "TIMESTAMP_MS":    "1745440000000",
  "MACHINE_STATE":   "RUNNING",
  "PARTS_PRODUCED":  7,
  "PARTS_REJECTED":  1,
  "CYCLE_TIME_MS":   3450,
  "IDEAL_CYCLE_MS":  3000,
  "SPEED_PCT":       85,
  "TEMPERATURE_C":   63,
  "VIBRATION_MM_S":  "8.4",
  "FAULT_CODE":      "NONE",
  "SHIFT_ID":        "SHIFT-A"
}
```

---

## Subscriber — `mqtt-subscriber/`

**Runtime:** `mqtt` | **Topic filter:** `factory/oee/#`

Processors: `ConsumeMQTT - HiveMQ` → `PublishSnowpipeStreaming`

Streams directly into `SNOWFLAKE_DEMO.MQTT_OEE.MACHINE_TELEMETRY` via Snowpipe Streaming v2 (`DATA_SOURCE(TYPE => 'STREAMING')`) using `SNOWFLAKE_MANAGED` auth — no key-pair required.

---

## Snowflake Objects — `SNOWFLAKE_DEMO.MQTT_OEE`

| Object | Type | Details |
|--------|------|---------|
| `MACHINE_TELEMETRY` | Table | 13-field OEE landing table + `INGEST_TIME` |
| `OEE_STREAMING_PIPE` | Pipe | Snowpipe Streaming v2, owner: SYSADMIN |
| `DT_MACHINE_LATEST` | Dynamic Table | Latest reading per machine (DOWNSTREAM) |
| `DT_OEE_5MIN_STATS` | Dynamic Table | 5-min OEE aggregations — Availability, Performance, Quality (DOWNSTREAM) |
| `DT_OEE_HEALTH_MONITOR` | Dynamic Table | Health classification per machine (1 minute) |

---

## Infrastructure

| Component | Details |
|-----------|---------|
| MQTT Broker | HiveMQ Cloud (TLS, port 8883) |
| Publisher runtime | `mqttpublisher` — Snowflake OpenFlow SPCS |
| Subscriber runtime | `mqtt` — Snowflake OpenFlow SPCS |
| External Access Integration | `HIVEMQ_EAI` |
| Network Rule | `OPENFLOW.OPENFLOW.MQTT_GITHUB_REGISTRY_NETWORK_RULE` |
| Snowflake Account | `SFSENORTHAMERICA-JLEONG_AWS1` |
| Warehouse (DTs) | `COMPUTE_WH` |
