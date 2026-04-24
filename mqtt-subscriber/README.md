# mqtt-subscriber — Battery MQTT Telemetry

Snowflake OpenFlow runtime that subscribes to HiveMQ Cloud and lands IoT battery telemetry into Snowflake via Snowpipe Streaming v2.

---

## Runtime

| Property | Value |
|----------|-------|
| Runtime name | `mqtt` |
| Flow name | *Battery MQTT Telemetry* |
| Process Group ID | `0c40c568-019c-1000-ffff-fffff72adeaf` |
| Authentication | `SNOWFLAKE_MANAGED` (SPCS session token — no key-pair needed) |
| External Access Integration | `HIVEMQ_EAI` |

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

Subscribes to HiveMQ Cloud over TLS and receives battery telemetry messages published by the `mqtt-publisher` runtime.

| Property | Value |
|----------|-------|
| Broker URI | `ssl://1a720034763944d0a28a49866a22cb78.s1.eu.hivemq.cloud:8883` |
| MQTT Spec | v3.1 (MQTT 4) |
| Topic Filter | `battery/telemetry/#` |
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
| Schema | `MQTT_CLONE` |
| Pipe | `MQTT_STREAMING_PIPE` |
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

### Landing Table — `SNOWFLAKE_DEMO.MQTT_CLONE.MQTT_TELEMETRY`

```sql
CREATE TABLE SNOWFLAKE_DEMO.MQTT_CLONE.MQTT_TELEMETRY (
  BATTERY_ID              VARCHAR(20),
  DEVICE_ID               VARCHAR(20),
  TIMESTAMP_MS            VARCHAR(30),
  VOLTAGE_MV              NUMBER(38,0),
  CURRENT_MA              NUMBER(38,0),
  TEMPERATURE_C           NUMBER(38,0),
  STATE_OF_CHARGE_PCT     NUMBER(38,0),
  STATE_OF_HEALTH_PCT     NUMBER(38,0),
  CYCLE_COUNT             NUMBER(38,0),
  CAPACITY_REMAINING_MAH  NUMBER(38,0),
  IS_CHARGING             VARCHAR(5),
  ERROR_CODE              VARCHAR(20),
  INGEST_TIME             TIMESTAMP_LTZ DEFAULT CURRENT_TIMESTAMP()
);
```

### Snowpipe Streaming v2 Pipe — `SNOWFLAKE_DEMO.MQTT_CLONE.MQTT_STREAMING_PIPE`

Uses `DATA_SOURCE(TYPE => 'STREAMING')` — no internal stage required. Owner must be `SYSADMIN` (or the role running the OpenFlow SPCS service) for the streaming API to acquire a channel.

```sql
CREATE OR REPLACE PIPE SNOWFLAKE_DEMO.MQTT_CLONE.MQTT_STREAMING_PIPE
  AS COPY INTO SNOWFLAKE_DEMO.MQTT_CLONE.MQTT_TELEMETRY (
    BATTERY_ID, DEVICE_ID, TIMESTAMP_MS, VOLTAGE_MV, CURRENT_MA,
    TEMPERATURE_C, STATE_OF_CHARGE_PCT, STATE_OF_HEALTH_PCT,
    CYCLE_COUNT, CAPACITY_REMAINING_MAH, IS_CHARGING, ERROR_CODE
  )
  FROM (
    SELECT
      $1:BATTERY_ID::VARCHAR(20),   $1:DEVICE_ID::VARCHAR(20),
      $1:TIMESTAMP_MS::VARCHAR(30), $1:VOLTAGE_MV::NUMBER(38,0),
      $1:CURRENT_MA::NUMBER(38,0),  $1:TEMPERATURE_C::NUMBER(38,0),
      $1:STATE_OF_CHARGE_PCT::NUMBER(38,0), $1:STATE_OF_HEALTH_PCT::NUMBER(38,0),
      $1:CYCLE_COUNT::NUMBER(38,0), $1:CAPACITY_REMAINING_MAH::NUMBER(38,0),
      $1:IS_CHARGING::VARCHAR(5),   $1:ERROR_CODE::VARCHAR(20)
    FROM TABLE(DATA_SOURCE(TYPE => 'STREAMING'))
  );
```

---

## Analytics — Dynamic Tables

Three downstream dynamic tables in `SNOWFLAKE_DEMO.MQTT_CLONE` provide BMS analytics on top of `MQTT_TELEMETRY`:

| Table | Lag | Purpose |
|-------|-----|---------|
| `DT_BATTERY_LATEST` | DOWNSTREAM | Latest reading per battery ID (QUALIFY window function) |
| `DT_BATTERY_5MIN_STATS` | DOWNSTREAM | 5-minute aggregated stats (avg/min/max voltage, current, temp, SOC, SOH) |
| `DT_BATTERY_HEALTH_MONITOR` | 1 minute | Health classification per battery — SOC, temperature and SOH tiers with an alert flag |

`DT_BATTERY_HEALTH_MONITOR` reads from `DT_BATTERY_LATEST`, forming a three-layer pipeline:

```
MQTT_TELEMETRY
    ├── DT_BATTERY_LATEST  (DOWNSTREAM)
    │       ├── DT_BATTERY_5MIN_STATS  (DOWNSTREAM)
    │       └── DT_BATTERY_HEALTH_MONITOR  (1 minute)
```

Health classification thresholds used by `DT_BATTERY_HEALTH_MONITOR`:

| Metric | CRITICAL | WARNING/LOW | FAIR | GOOD/NORMAL |
|--------|----------|-------------|------|-------------|
| SOC | < 10% | < 20% | < 50% | ≥ 50% |
| Temperature | > 45°C | > 40°C | — | 5–40°C |
| SOH | — | < 70% (REPLACE) / < 80% (DEGRADED) | < 90% (AGING) | ≥ 90% |

---

## Infrastructure

| Component | Value |
|-----------|-------|
| MQTT Broker | HiveMQ Cloud — `1a720034763944d0a28a49866a22cb78.s1.eu.hivemq.cloud:8883` |
| Network Rule | `OPENFLOW.OPENFLOW.MQTT_GITHUB_REGISTRY_NETWORK_RULE` |
| External Access Integration | `HIVEMQ_EAI` |
| Snowflake Account | `SFSENORTHAMERICA-JLEONG_AWS1` |
| Warehouse (DTs) | `COMPUTE_WH` |
