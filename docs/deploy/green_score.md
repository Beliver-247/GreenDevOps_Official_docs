# Green Score — Explainable Decision Score

The **Green Score** is a 0–100 composite score computed before every deployment decision. It makes the agent's reasoning transparent and auditable — every factor that influenced the decision is broken down with its individual contribution.

---

## Why a Score?

A plain LLM decision ("deploy with canary") is a black box. The Green Score fixes this by quantifying exactly *why* the agent made its choice, using real measured data from five tool calls. This makes the system suitable for research evaluation, thesis reporting, and operational monitoring.

---

## Score Composition

The total score is the sum of six weighted factors (max 100 points):

| Factor | Max Points | Weight | What it measures |
|---|---|---|---|
| **Carbon Intensity** | 35 | 35% | How dirty the electricity grid is right now |
| **CPU Utilisation** | 20 | 20% | Server headroom for a safe deployment |
| **Memory Utilisation** | 10 | 10% | Memory pressure on the deployment host |
| **Business Time** | 10 | 10% | Traffic context — peak hours constrain strategy |
| **Deployment History** | 15 | 15% | Recent success rate from `deployments.db` |
| **Historical Timing** | 10 | 10% | % of historical deployments at this hour that were "good timing" |

!!! note "Why carbon gets the highest weight (35%)"
    Carbon intensity is the primary research variable — it has the most direct and measurable impact on CO₂ emissions. CPU comes second because high CPU during deployment causes longer execution time, which directly translates to more energy consumed.

---

## Factor Details

### 1. Carbon Score (0–35)

| Grid Intensity | Score | Rating |
|---|---|---|
| < 100 gCO₂/kWh | 35 | Very Low |
| 100–200 | 28 | Low |
| 200–300 | 18 | Medium |
| 300–400 | 8 | High |
| ≥ 400 | 0 | Very High |

### 2. CPU Score (0–20)

| CPU Usage | Score |
|---|---|
| < 20% | 20 — ideal |
| 20–40% | 16 — light load |
| 40–60% | 12 — moderate |
| 60–80% | 6 — heavy, caution |
| ≥ 80% | 0 — unsafe |

### 3. Memory Score (0–10)

| Memory Usage | Score |
|---|---|
| < 40% | 10 |
| 40–60% | 8 |
| 60–75% | 5 |
| 75–85% | 2 |
| ≥ 85% | 0 |

### 4. Business Time Score (0–10)

| Condition | Score | Rationale |
|---|---|---|
| Maintenance window (22:00–05:00) | 10 | Downtime acceptable — any strategy safe |
| Off-peak hours | 7 | Flexible — rolling or recreate fine |
| Business hours (08:00–18:00) | 5 | No downtime allowed — rolling only |
| Peak traffic (09–11, 14–16) | 2 | Canary required — blast radius must be minimal |

### 5. History Score (0–15)

Pulled from `deployments.db` (Component 1's SQLite database):

| Condition | Score |
|---|---|
| DB unavailable (fresh install) | 7 — neutral |
| ≥ 80% recent success rate | 15 |
| 60–79% success rate | 11 |
| < 60% success rate | 6 |

### 6. Timing Score (0–10)

Computed from the historical training CSV (`training_dataset.csv`), which records hourly `Good_Timing` labels:

| Good-timing % at current hour | Score |
|---|---|
| ≥ 80% | 10 |
| 60–79% | 8 |
| 40–59% | 5 |
| < 40% | 2 |
| No CSV data | 5 — neutral |

---

## Grade Thresholds

| Score | Grade | Typical decision |
|---|---|---|
| 85–100 | **Excellent** | Deploy immediately with any strategy |
| 70–84 | **Good** | Deploy — rolling or recreate preferred |
| 50–69 | **Moderate** | Deploy with caution — canary preferred |
| 0–49 | **Poor** | Wait if a green window is available within 4 hours |

---

## Score in the Fallback Mode

If Ollama (the local LLM) is unavailable, the agent switches to a **score-driven fallback** that uses the same Green Score to make decisions — the score thresholds replace the LLM's reasoning. This ensures consistent, auditable behaviour even without the LLM running.

---

## Example Breakdown

From a real Build #191:

```
Carbon Score     : 8/35   — 293 gCO2/kWh (high)
CPU Score        : 20/20  — 12% utilisation
Memory Score     : 10/10  — 38% utilisation
Business Time    :  5/10  — business hours, no downtime allowed
History Score    : 15/15  — 100% recent success rate
Timing Score     :  8/10  — 72% historically good at this hour
──────────────────────────────────
Final Green Score: 66/100  →  Moderate
```

Result: **deploy** with **canary** strategy (moderate score → cautious strategy chosen).
