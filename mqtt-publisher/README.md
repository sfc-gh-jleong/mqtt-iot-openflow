# mqtt-publisher — MQTT Publisher

Snowflake OpenFlow runtime that generates synthetic factory machine telemetry and publishes it to HiveMQ Cloud every 5 seconds.

---

## Runtime

| Property | Value |
|----------|-------|
| Runtime name | `mqttpublisher` |
| Flow name | *MQTT Publisher* |
| Process Group ID | `bb8641b4-019d-1000-0000-00005aa8c3d0` |
| External Access Integration | `HIVEMQ_EAI` |
| Flow definition | [`flow.json`](flow.json) |

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

Produces a synthetic JSON machine record every 5 seconds using NiFi Expression Language randomisation across 5 machines (`MCH-001` – `MCH-005`) and 3 production lines (`LINE-01` – `LINE-03`).

**Generated payload:**
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

**NiFi Expression Language template:**
```
{"MACHINE_ID":"MCH-00${random():mod(5):plus(1)}","LINE_ID":"LINE-0${random():mod(3):plus(1)}","TIMESTAMP_MS":"${now()}","MACHINE_STATE":"${random():mod(10):equals('0'):ifElse('FAULT','RUNNING')}","PARTS_PRODUCED":"${random():mod(10):plus(1)}","PARTS_REJECTED":"${random():mod(3)}","CYCLE_TIME_MS":"${random():mod(1500):plus(3000)}","IDEAL_CYCLE_MS":"3000","SPEED_PCT":"${random():mod(40):plus(60)}","TEMPERATURE_C":"${random():mod(50):plus(40)}","VIBRATION_MM_S":"${random():mod(20):plus(1)}.${random():mod(10)}","FAULT_CODE":"${random():mod(10):equals('0'):ifElse('ERR_OVERTEMP','NONE')}","SHIFT_ID":"SHIFT-A"}
```

**Field reference:**

| Field | Unit | Simulated Range | Notes |
|-------|------|-----------------|-------|
| `MACHINE_ID` | — | MCH-001 – MCH-005 | 5 simulated CNC/assembly machines |
| `LINE_ID` | — | LINE-01 – LINE-03 | 3 simulated production lines |
| `TIMESTAMP_MS` | epoch ms | current time | `${now()}` returns epoch milliseconds |
| `MACHINE_STATE` | — | RUNNING / FAULT | 10% FAULT, 90% RUNNING |
| `PARTS_PRODUCED` | count | 1–10 | Good parts produced in this cycle |
| `PARTS_REJECTED` | count | 0–2 | Defective/rejected parts |
| `CYCLE_TIME_MS` | milliseconds | 3000–4500 | Actual cycle duration (ideal = 3000ms) |
| `IDEAL_CYCLE_MS` | milliseconds | 3000 | Fixed design cycle time for this machine family |
| `SPEED_PCT` | percent | 60–100 | Machine speed as % of rated speed |
| `TEMPERATURE_C` | Celsius | 40–90 | Spindle/motor temperature |
| `VIBRATION_MM_S` | mm/s | 1.0–20.9 | Vibration RMS reading |
| `FAULT_CODE` | string | ERR_OVERTEMP / NONE | 10% chance of ERR_OVERTEMP |
| `SHIFT_ID` | — | SHIFT-A | Current production shift |

### UpdateAttribute — Set MQTT Topic

Sets the FlowFile attribute `mqtt.topic = factory/oee` so the downstream `PublishMQTT` processor knows which topic to publish to.

### PublishMQTT — Publish to HiveMQ

Publishes the FlowFile content to HiveMQ Cloud over TLS using username/password authentication.

| Property | Value |
|----------|-------|
| Broker URI | `ssl://1a720034763944d0a28a49866a22cb78.s1.eu.hivemq.cloud:8883` |
| MQTT Spec | v3.1 (MQTT 4) |
| Topic | `${mqtt.topic}` (from UpdateAttribute) → `factory/oee` |
| Quality of Service | 0 (fire-and-forget) |
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
| Topic | `factory/oee` |
| Network Rule | `OPENFLOW.OPENFLOW.MQTT_GITHUB_REGISTRY_NETWORK_RULE` |
| External Access Integration | `HIVEMQ_EAI` |
| Snowflake Account | `SFSENORTHAMERICA-JLEONG_AWS1` |

---

## Notes

- This runtime has no Snowflake destination — it is a pure MQTT publisher.
- The `mqtt` subscriber runtime consumes from `factory/oee/#` and lands data in `SNOWFLAKE_DEMO.MQTT_OEE.MACHINE_TELEMETRY`.
- Flow is version-controlled in this repo under the `mqtt-publisher/` subfolder using a scoped OpenFlow git registry client.
- To import this flow into a new runtime: use `flow.json` with the NiFi UI Process Group import or `nipyapi` CLI.
