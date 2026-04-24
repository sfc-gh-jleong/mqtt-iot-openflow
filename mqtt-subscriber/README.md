# mqtt-subscriber — Battery MQTT Telemetry

Snowflake OpenFlow runtime that subscribes to HiveMQ Cloud and streams factory machine telemetry into Snowflake via Snowpipe Streaming v2.

---

## Runtime

| Property | Value |
|----------|-------|
| Runtime name | `mqtt` |
| Flow name | *Battery MQTT Telemetry* |
| Process Group ID | `0c40c568-019c-1000-ffff-fffff72adeaf` |
| Authentication | `SNOWFLAKE_MANAGED` (SPCS session token — no key-pair needed) |
| External Access Integration | `HIVEMQ_EAI` |
| Flow definition | [`flow.json`](flow.json) |

---

## Flow Topology

```
ConsumeMQTT - HiveMQ
        |  [Message]
        v
PublishSnowpipeStreaming
        |  [failure]
        v
     Funnel
```

---

## Processors

### ConsumeMQTT - HiveMQ

Subscribes to HiveMQ Cloud over TLS and receives machine telemetry messages published by the `mqtt-publisher` runtime.

| Property | Value |
|----------|-------|
| Broker URI | `ssl://1a720034763944d0a28a49866a22cb78.s1.eu.hivemq.cloud:8883` |
| MQTT Spec | v3.1 (MQTT 4) |
| Topic Filter | `factory/oee/#` |
| Quality of Service | 1 (at-least-once) |
| Client ID | `openflow-mqtt-consumer` |
| Session Expiry | 24 hrs (durable session — survives runtime restarts) |
| SSL Context Service | `HiveMQ SSL Service v2` — JVM cacerts (`${java.home}/lib/security/cacerts`), TLSv1.2 |
| Max Queue Size | 10,000 messages |

### PublishSnowpipeStreaming

Streams FlowFile content directly into Snowflake using the Snowpipe Streaming v2 API (channel-based insert — no stage or copy required).

| Property | Value |
|----------|-------|
| Authentication Strategy | `SNOWFLAKE_MANAGED` |
| Destination Type | `PIPE` |
| Database | `SNOWFLAKE_DEMO` |
| Schema | `MQTT_OEE` |
| Pipe | `OEE_STREAMING_PIPE` |
| Transfer Strategy | `MANAGED` |
| Channel Group | `SHARED` |
| Offset Tracking Resolution | `DISABLED` |
| Web Client Service | `StandardWebClientServiceProvider` |

---

## Controller Services

| Service | Type | Purpose |
|---------|------|---------|
| `HiveMQ SSL Service v2` | `StandardSSLContextService` | TLS for HiveMQ Cloud — uses JVM cacerts, TLSv1.2 |
| `Web Client Service` | `StandardWebClientServiceProvider` | HTTP/2 client for Snowpipe Streaming API calls |
| `StandardPrivateKeyService` | `StandardPrivateKeyService` | Reserved for key-pair auth (not active — `SNOWFLAKE_MANAGED` is used) |

---

## Snowflake Objects

### Landing Table — `SNOWFLAKE_DEMO.MQTT_OEE.MACHINE_TELEMETRY`

```sql
CREATE TABLE SNOWFLAKE_DEMO.MQTT_OEE.MACHINE_TELEMETRY (
  MACHINE_ID        VARCHAR(20),
  LINE_ID           VARCHAR(20),
  TIMESTAMP_MS      VARCHAR(30),
  MACHINE_STATE     VARCHAR(20),
  PARTS_PRODUCED    NUMBER(38,0),
  PARTS_REJECTED    NUMBER(38,0),
  CYCLE_TIME_MS     NUMBER(38,0),
  IDEAL_CYCLE_MS    NUMBER(38,0),
  SPEED_PCT         NUMBER(38,0),
  TEMPERATURE_C     NUMBER(38,0),
  VIBRATION_MM_S    NUMBER(38,2),
  FAULT_CODE        VARCHAR(30),
  SHIFT_ID          VARCHAR(20),
  INGEST_TIME       TIMESTAMP_LTZ DEFAULT CURRENT_TIMESTAMP()
);
```

### Snowpipe Streaming v2 Pipe — `SNOWFLAKE_DEMO.MQTT_OEE.OEE_STREAMING_PIPE`

Uses `DATA_SOURCE(TYPE => 'STREAMING')` — no internal stage required. Owner must be `SYSADMIN` for the streaming API to acquire a channel.

```sql
CREATE OR REPLACE PIPE SNOWFLAKE_DEMO.MQTT_OEE.OEE_STREAMING_PIPE
  AS COPY INTO SNOWFLAKE_DEMO.MQTT_OEE.MACHINE_TELEMETRY (
    MACHINE_ID, LINE_ID, TIMESTAMP_MS, MACHINE_STATE,
    PARTS_PRODUCED, PARTS_REJECTED, CYCLE_TIME_MS, IDEAL_CYCLE_MS,
    SPEED_PCT, TEMPERATURE_C, VIBRATION_MM_S, FAULT_CODE, SHIFT_ID
  )
  FROM (
    SELECT
      $1:MACHINE_ID::VARCHAR(20),   $1:LINE_ID::VARCHAR(20),
      $1:TIMESTAMP_MS::VARCHAR(30), $1:MACHINE_STATE::VARCHAR(20),
      $1:PARTS_PRODUCED::NUMBER(38,0), $1:PARTS_REJECTED::NUMBER(38,0),
      $1:CYCLE_TIME_MS::NUMBER(38,0),  $1:IDEAL_CYCLE_MS::NUMBER(38,0),
      $1:SPEED_PCT::NUMBER(38,0),      $1:TEMPERATURE_C::NUMBER(38,0),
      $1:VIBRATION_MM_S::NUMBER(38,2), $1:FAULT_CODE::VARCHAR(30),
      $1:SHIFT_ID::VARCHAR(20)
    FROM TABLE(DATA_SOURCE(TYPE => 'STREAMING'))
  );
```

---

## Analytics — Dynamic Tables

Three downstream dynamic tables in `SNOWFLAKE_DEMO.MQTT_OEE` provide OEE analytics on top of `MACHINE_TELEMETRY`:

| Table | Lag | Purpose |
|-------|-----|---------|
| `DT_MACHINE_LATEST` | DOWNSTREAM | Latest reading per machine ID (QUALIFY window function) |
| `DT_OEE_5MIN_STATS` | DOWNSTREAM | 5-minute OEE aggregations — Availability, Performance, Quality per machine per window |
| `DT_OEE_HEALTH_MONITOR` | 1 minute | Health classification per machine — state, temperature, vibration, and performance tiers with `HAS_ALERT` flag |

`DT_OEE_HEALTH_MONITOR` reads from `DT_MACHINE_LATEST`, forming a three-layer pipeline:

```
MACHINE_TELEMETRY
    └── DT_MACHINE_LATEST  (DOWNSTREAM)
            ├── DT_OEE_5MIN_STATS       (DOWNSTREAM)
            └── DT_OEE_HEALTH_MONITOR   (1 minute)
```

**OEE component derivations in `DT_OEE_5MIN_STATS`:**

| OEE Component | Formula |
|---------------|---------|
| Availability | `RUNNING readings / total readings × 100` |
| Performance | `IDEAL_CYCLE_MS / AVG(CYCLE_TIME_MS) × 100` |
| Quality | `SUM(PARTS_PRODUCED) / (SUM(PARTS_PRODUCED) + SUM(PARTS_REJECTED)) × 100` |

**Health classification thresholds in `DT_OEE_HEALTH_MONITOR`:**

| Dimension | CRITICAL | WARNING | NORMAL/ON_PACE |
|-----------|----------|---------|----------------|
| Temperature | > 90°C | > 75°C | ≤ 75°C |
| Vibration | > 15 mm/s | > 10 mm/s | ≤ 10 mm/s |
| Cycle Time | > 150% of ideal | > 120% of ideal | ≤ 120% of ideal |
| Machine State | FAULT | — | RUNNING / IDLE / CHANGEOVER |

`HAS_ALERT = TRUE` when any of: `MACHINE_STATE = 'FAULT'`, temp > 75°C, vibration > 10 mm/s, or cycle time > 150% of ideal.

---

## Infrastructure

| Component | Value |
|-----------|-------|
| MQTT Broker | HiveMQ Cloud — `1a720034763944d0a28a49866a22cb78.s1.eu.hivemq.cloud:8883` |
| Network Rule | `OPENFLOW.OPENFLOW.MQTT_GITHUB_REGISTRY_NETWORK_RULE` |
| External Access Integration | `HIVEMQ_EAI` |
| Snowflake Account | `SFSENORTHAMERICA-JLEONG_AWS1` |
| Warehouse (DTs) | `COMPUTE_WH` |
