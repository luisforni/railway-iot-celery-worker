# railway-iot-celery-worker

Celery async worker service for **railway-iot-platform**.

---

## Responsibilities

- Process incoming MQTT sensor messages (persist to TimescaleDB)
- Evaluate threshold alert rules and create alerts
- Use Redis as both the task broker and result backend

---

## Stack

| Component | Library |
|---|---|
| Task queue | Celery |
| Broker | Redis |
| Result backend | Redis |
| Database | TimescaleDB (psycopg2) |

---

## Registered Tasks

### `apps.tasks.ingest.persist_reading`

Persists a validated sensor reading to TimescaleDB.

**Triggered by:** Django MQTT consumer on every inbound MQTT message.

**Input payload:**
```python
{
    "device_id": "track-sensor-01",
    "zone": "zone-a",
    "metric": "temperature",
    "value": 68.4,
    "timestamp": "2026-03-17T10:30:00Z"
}
```

**Validation performed before persistence:**
- Required fields present
- `device_id` and `zone` match regex `^[\w\-]{1,64}$`
- `metric` in whitelist (`temperature`, `vibration`, `rpm`, `brake-pressure`, `load-weight`)
- `value` within expected float bounds per metric
- `timestamp` is a valid ISO 8601 string

Invalid payloads are dropped silently.

---

### `apps.tasks.alert_rules.check_threshold_alert`

Evaluates whether a sensor reading exceeds warning or critical thresholds. Creates an `Alert` record if a threshold is breached.

**Triggered by:** Django MQTT consumer immediately after `persist_reading`.

**Threshold table:**

| Metric | Warning | Critical |
|---|---|---|
| `temperature` | ≥ 70 °C | ≥ 82 °C |
| `vibration` | ≥ 7 mm/s | ≥ 9 mm/s |
| `rpm` | ≥ 1600 rpm | ≥ 1750 rpm |
| `brake-pressure` | ≥ 7 bar | ≥ 7.8 bar |
| `load-weight` | ≥ 72,000 kg | ≥ 78,000 kg |

A new alert is only created if no unacknowledged alert already exists for the same `device_id` + `metric` combination (deduplication).

---

## Concurrency

The worker is configured with **4 concurrent processes** (configurable via `CELERY_CONCURRENCY`).

Recommended MQTT publish interval: **≥ 2,000 ms** per device to avoid queue overflow. At 2 s intervals with 25 devices × 5 metrics = 125 messages/s × 2 tasks = 250 tasks/s, well within worker capacity.

---

## Monitoring

```bash
# View worker logs
docker compose logs -f celery

# Inspect active queues (requires Redis CLI)
docker compose exec redis redis-cli -a "$REDIS_PASSWORD" LLEN celery

# Flush all queued tasks (use with care)
docker compose exec redis redis-cli -a "$REDIS_PASSWORD" FLUSHALL
```
