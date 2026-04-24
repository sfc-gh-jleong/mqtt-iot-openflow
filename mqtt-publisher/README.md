# mqtt-publisher — MQTT Publisher

Snowflake OpenFlow runtime that generates synthetic IoT battery telemetry and publishes it to HiveMQ Cloud every 5 seconds.

---

## Runtime

| Property | Value |
|----------|-------|
| Runtime name | `mqttpublisher` |
| Flow name | *MQTT Publisher* |
| Process Group ID | `bb8641b4-019d-1000-0000-00005aa8c3d0` |
| External Access Integration | `HIVEMQ_EAI` |

---

## Flow Topology

```
GenerateFlowFile (every 5s)
        |  [success]
        v
UpdateAttribute  (set mqtt.topic)
        |  [success]
        v
PublishMQTT → HiveMQ Cloud
        |  [failure]
        v
     Funnel
```

---

## Processors

### GenerateFlowFile — Generate Telemetry

Produces a synthetic JSON battery record every 5 seconds using NiFi Expression Language randomisation across 5 batteries (`BAT-001` – `BAT-005`) and 3 devices (`DEVICE-01` – `DEVICE-03`).

**Generated payload:**
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

**NiFi Expression Language template:**
```
{"BATTERY_ID":"BAT-00${random():mod(5):plus(1)}","DEVICE_ID":"DEVICE-0${random():mod(3):plus(1)}","TIMESTAMP_MS":"${now()}","VOLTAGE_MV":"${random():mod(1700):plus(2500)}","CURRENT_MA":"${random():mod(5000)}","TEMPERATURE_C":"${random():mod(30):plus(15)}","STATE_OF_CHARGE_PCT":"${random():mod(101)}","STATE_OF_HEALTH_PCT":"${random():mod(21):plus(80)}","CYCLE_COUNT":"${random():mod(500)}","CAPACITY_REMAINING_MAH":"${random():mod(2500):plus(2500)}","IS_CHARGING":"${random():mod(2):equals('0')}","ERROR_CODE":"NONE"}
```

**Field reference:**

| Field | Unit | Simulated Range | Notes |
|-------|------|-----------------|-------|
| `BATTERY_ID` | — | BAT-001 – BAT-005 | 5 simulated battery packs |
| `DEVICE_ID` | — | DEVICE-01 – DEVICE-03 | 3 simulated BMS devices |
| `TIMESTAMP_MS` | epoch ms | current time | `${now()}` returns epoch milliseconds |
| `VOLTAGE_MV` | millivolts | 2500–4200 | Li-ion cell operating range |
| `CURRENT_MA` | milliamps | 0–5000 | Positive = discharging |
| `TEMPERATURE_C` | Celsius | 15–45 | Normal operating range |
| `STATE_OF_CHARGE_PCT` | percent | 0–100 | Battery charge level |
| `STATE_OF_HEALTH_PCT` | percent | 80–100 | Battery health (degradation) |
| `CYCLE_COUNT` | count | 0–500 | Full charge/discharge cycles |
| `CAPACITY_REMAINING_MAH` | milliamp-hours | 2500–5000 | Usable capacity remaining |
| `IS_CHARGING` | boolean string | `"true"` / `"false"` | Charge state flag |
| `ERROR_CODE` | string | `"NONE"` | Static — no fault simulation |

### UpdateAttribute — Set MQTT Topic

Sets the FlowFile attribute `mqtt.topic = battery/telemetry` so the downstream `PublishMQTT` processor knows which topic to publish to.

### PublishMQTT — Publish to HiveMQ

Publishes the FlowFile content to HiveMQ Cloud over TLS using username/password authentication.

| Property | Value |
|----------|-------|
| Broker URI | `ssl://1a720034763944d0a28a49866a22cb78.s1.eu.hivemq.cloud:8883` |
| MQTT Spec | v3.1 (MQTT 4) |
| Topic | `${mqtt.topic}` (from UpdateAttribute) |
| Quality of Service | 1 (at-least-once) |
| Retain | false |
| SSL Context Service | `HiveMQ SSL Context` — JVM cacerts (`${java.home}/lib/security/cacerts`), TLSv1.2 |

---

## Controller Services

| Service | Type | Purpose |
|---------|------|---------|
| `HiveMQ SSL Context` | `StandardSSLContextService` | TLS for HiveMQ Cloud — uses JVM cacerts, TLSv1.2 |

---

## Infrastructure

| Component | Value |
|-----------|-------|
| MQTT Broker | HiveMQ Cloud — `1a720034763944d0a28a49866a22cb78.s1.eu.hivemq.cloud:8883` |
| Publish interval | 5 seconds |
| Topic | `battery/telemetry` |
| Network Rule | `OPENFLOW.OPENFLOW.MQTT_GITHUB_REGISTRY_NETWORK_RULE` |
| External Access Integration | `HIVEMQ_EAI` |
| Snowflake Account | `SFSENORTHAMERICA-JLEONG_AWS1` |

---

## Notes

- This runtime has no Snowflake destination — it is a pure MQTT publisher.
- The `mqtt` subscriber runtime consumes from the same `battery/telemetry/#` topic and lands data in `SNOWFLAKE_DEMO.MQTT_CLONE.MQTT_TELEMETRY`.
- Flow is version-controlled in this repo under the `mqtt-publisher/` subfolder using a scoped OpenFlow git registry client.
