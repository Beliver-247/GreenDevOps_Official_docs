# Deploy Component — API Reference

The Green AI Agent exposes a REST API on port `5002` (configurable via `AGENT_PORT`). Jenkins calls the main endpoint before every deployment; the debug endpoints are useful for testing and monitoring.

---

## Base URL

```
http://<agent-host>:5002
```

In the integrated pipeline, Jenkins reaches the agent at `http://172.17.0.1:5002` (the Docker host gateway IP).

---

## Main Endpoint

### `POST /api/check`

The primary endpoint. Jenkins calls this before every deployment to get a strategy decision.

**Request body (JSON):**

```json
{
  "job_name": "my-pipeline",
  "build_number": "42",
  "strategy": "rolling"
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `job_name` | string | Yes | Jenkins job name |
| `build_number` | string | Yes | Jenkins build number |
| `strategy` | string | No | Developer's preferred strategy — agent may override if conditions suggest otherwise |

**Response (JSON):**

```json
{
  "decision": "deploy",
  "strategy": "canary",
  "reason": "Carbon is 293 gCO2/kWh (high). Canary selected to minimise load spike.",
  "carbon_rating": "high",
  "green_score": 80,
  "green_grade": "Good",
  "confidence": 0.90,
  "estimated_co2_saving_pct": 0,
  "next_green_window": null,
  "score_breakdown": {
    "carbon": 8,
    "cpu": 20,
    "memory": 10,
    "business_time": 5,
    "history": 15,
    "timing": 8
  },
  "job_name": "my-pipeline",
  "build_number": "42",
  "analyzed_at": "2026-08-30T06:09:32.176333"
}
```

**HTTP status:** `200 OK` always (agent never returns 4xx/5xx for a successful analysis — check `decision` field).

---

## Monitoring Endpoints

### `GET /api/health`

Check if the agent server is running and which model it is using.

```json
{
  "status": "ok",
  "model": "qwen2.5:1.5b",
  "ollama_url": "http://localhost:11434",
  "timestamp": "2026-08-30T06:00:00.000000"
}
```

---

### `GET /api/history`

Returns the last 20 deployment decisions made by the agent (in-memory, reset on restart).

```json
{
  "total": 5,
  "decisions": [
    {
      "decision": "deploy",
      "strategy": "canary",
      "green_score": 80,
      "green_grade": "Good",
      "carbon_rating": "high",
      "job_name": "my-pipeline",
      "build_number": "42",
      "analyzed_at": "2026-08-30T06:09:32.176333"
    }
  ]
}
```

---

### `GET /api/stats`

Aggregated statistics across all decisions since the last restart.

```json
{
  "total_decisions": 10,
  "deploy_count": 8,
  "wait_count": 2,
  "wait_percentage": 20.0,
  "strategy_breakdown": {
    "canary": 5,
    "rolling": 3,
    "recreate": 0
  },
  "avg_co2_saving_pct": 12.5,
  "avg_green_score": 74.3,
  "grade_distribution": {
    "Excellent": 1,
    "Good": 5,
    "Moderate": 3,
    "Poor": 1
  },
  "method_breakdown": {
    "llm_react": 9,
    "score_fallback": 1
  }
}
```

---

## Tool Debug Endpoints

These endpoints let you test each agent tool independently without triggering a full decision. Useful for verifying your environment setup.

| Endpoint | Tool tested | What it returns |
|---|---|---|
| `GET /api/tools/carbon` | `get_carbon_intensity` | Current grid carbon intensity (live or estimated) |
| `GET /api/tools/resources` | `get_system_resources` | CPU%, memory%, disk%, deployment safety |
| `GET /api/tools/time` | `get_time_context` | Current hour, business hours flag, time strategy hint |
| `GET /api/tools/history` | `get_deployment_history` | Recent deployments from DB + training CSV insights |
| `GET /api/tools/window` | `find_next_green_window` | Next recommended deployment time + top 3 windows |

### Example: `GET /api/tools/carbon`

```json
{
  "intensity_gco2_kwh": 146.0,
  "source": "electricity_maps_live",
  "zone": "LK",
  "rating": "low",
  "timestamp": "2026-08-30T06:07:53.999410"
}
```

### Example: `GET /api/tools/window`

```json
{
  "next_green_window": "2026-08-30 03:00",
  "hours_from_now": 3,
  "estimated_carbon": 100,
  "good_timing_pct": 85.0,
  "top_3_windows": [
    {"window": "2026-08-30 03:00", "hour": 3, "carbon_estimate": 100, "good_timing_pct": 85.0},
    {"window": "2026-08-30 02:00", "hour": 2, "carbon_estimate": 110, "good_timing_pct": 80.0},
    {"window": "2026-08-30 04:00", "hour": 4, "carbon_estimate": 110, "good_timing_pct": 78.0}
  ],
  "reason": "Hour 03:00 has estimated carbon intensity of 100 gCO2/kWh (vs current 293 gCO2/kWh)"
}
```

---

## Environment Variables

| Variable | Default | Description |
|---|---|---|
| `AGENT_PORT` | `5002` | Port the Flask server listens on |
| `OLLAMA_URL` | `http://localhost:11434` | URL of the local Ollama instance |
| `OLLAMA_MODEL` | `qwen2.5:1.5b` | Ollama model to use for reasoning |
| `ELECTRICITY_MAPS_KEY` | _(empty)_ | API key for live carbon data — falls back to CSV estimate if not set |
| `CARBON_ZONE` | `IN-SO` | Electricity Maps grid zone |
| `WATCH_DIR` | `/opt/energy-profiller-hiran` | Directory containing `deployments.db` and `training_dataset.csv` |
| `TRAINING_CSV` | `$WATCH_DIR/training_dataset.csv` | Path to the historical timing CSV |
