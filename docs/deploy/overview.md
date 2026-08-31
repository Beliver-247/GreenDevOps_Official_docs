# Deploy Component Overview

The **Deploy Component** (also called the **Green AI Agent**) is the second of the two AI models in the GreenDevOps pipeline. While the ML Optimizer decides *when* to deploy, the Green AI Agent decides *how* to deploy — selecting the deployment strategy that minimises carbon footprint under current conditions.

---

## Where It Fits in the Pipeline

```
Git Push
   │
   ▼
Build Optimizer ─── WHEN to deploy? (LightGBM ML model)
   │
   ▼
Green AI Agent ──── HOW to deploy? (ReAct LLM Agent)
   │
   ▼
Jenkins executes the chosen strategy (rolling / canary / recreate)
```

The agent runs as a persistent **Flask HTTP server** on the deployment machine and is called by Jenkins before every deployment via a single `POST /api/check` request.

---

## The Three Deployment Strategies

The agent chooses between three Docker-based strategies:

| Strategy | Downtime | Infrastructure Overhead | When to use |
|---|---|---|---|
| **rolling** | None | 1.1× (brief overlap) | Business hours, production systems, general default |
| **canary** | None | 1.2× (two versions run simultaneously) | High carbon intensity, peak traffic — minimises load spike |
| **recreate** | 1–5 minutes | 1.0× (lowest CO₂ — no overlap) | Maintenance windows, low-traffic periods, simple apps |

!!! tip "Carbon logic behind strategy choice"
    Counterintuitively, **canary** is preferred during *high carbon* periods — not because it's the greenest overall, but because it routes only 10–20% of traffic to the new version, minimising the immediate compute spike. The full promotion happens once conditions improve.

---

## Decision Output

Every call to `/api/check` returns a structured JSON decision:

```json
{
  "decision": "deploy",
  "strategy": "canary",
  "reason": "Carbon is 293 gCO2/kWh (high). Canary selected to minimise load spike during peak hours.",
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
  }
}
```

| Field | Description |
|---|---|
| `decision` | `"deploy"` or `"wait"` |
| `strategy` | `"rolling"`, `"canary"`, or `"recreate"` |
| `reason` | Human-readable 2–3 sentence explanation |
| `carbon_rating` | `"low"` / `"medium"` / `"high"` / `"very_high"` |
| `green_score` | 0–100 composite score (see [Green Score](green_score.md)) |
| `green_grade` | `"Excellent"` / `"Good"` / `"Moderate"` / `"Poor"` |
| `confidence` | LLM confidence in its decision (0.0–1.0) |
| `estimated_co2_saving_pct` | Estimated % CO₂ savings vs a default recreate |
| `next_green_window` | If `decision = "wait"`, the recommended deployment time |
| `score_breakdown` | Per-factor scores contributing to `green_score` |

---

## When the Agent Recommends Waiting

The agent only recommends delaying when **all three conditions** are true:

1. Carbon intensity is **HIGH or VERY HIGH** (> 300 gCO₂/kWh)
2. A greener window exists **within 4 hours**
3. The deployment is not flagged as urgent

If no greener window is available within 4 hours, the agent always recommends deploying immediately — using **canary** to minimise the carbon spike.

---

## Next Steps

- [Setup & Installation](setup.md) — how to run the agent server
- [API Reference](api_reference.md) — all endpoints and request/response schemas
- [Green Score](green_score.md) — how the 100-point explainable score is calculated
