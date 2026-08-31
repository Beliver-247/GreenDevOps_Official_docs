# Deploy Component — Setup & Installation

The Green AI Agent requires Python 3.10+, [Ollama](https://ollama.com) (for the local LLM), and the agent's Python dependencies. It runs as a long-lived Flask server on your deployment machine.

---

## Prerequisites

| Requirement | Notes |
|---|---|
| **Python 3.10+** | Used to run the agent server |
| **Ollama** | Runs the local LLM (`qwen2.5:1.5b`) — free, no internet needed after install |
| **`deployments.db`** | SQLite database written by Component 1 (deployment tracker) — optional but recommended |
| **`training_dataset.csv`** | Historical timing data — optional; agent falls back to heuristics without it |
| **Electricity Maps API key** | Optional — agent estimates carbon from CSV patterns if not provided |

---

## Step 1 — Install Ollama and Pull the Model

```bash
# Install Ollama (Linux)
curl -fsSL https://ollama.com/install.sh | sh

# Pull the model (only needed once — ~1 GB download)
ollama pull qwen2.5:1.5b

# Verify it's running
ollama list
```

Ollama starts automatically as a system service. Verify it is accessible:

```bash
curl http://localhost:11434/api/tags
```

---

## Step 2 — Set Up the Python Environment

Copy the agent source files to your deployment machine (e.g. `/opt/green-agent/`):

```
agent_server.py
agent_tools.py
green_agent.py
carbon_api.py
carbon_calculator.py
carbon_service.py
deployment_tracker.py
energy_calculator.py
profiler.py
db_sync.py
```

Create a virtual environment and install dependencies:

```bash
cd /opt/green-agent
python3 -m venv venv
source venv/bin/activate
pip install flask requests psutil
```

---

## Step 3 — Configure Environment Variables

The agent reads all configuration from environment variables. Create a `.env` file or export them in your shell / systemd service:

```bash
export AGENT_PORT=5002
export OLLAMA_URL=http://localhost:11434
export OLLAMA_MODEL=qwen2.5:1.5b

# Optional — enables live carbon data from Electricity Maps
export ELECTRICITY_MAPS_KEY=your_api_key_here
export CARBON_ZONE=LK   # Sri Lanka — or IN-SO for South India

# Point to your deployment DB and training CSV
export WATCH_DIR=/opt/energy-profiller-hiran
export TRAINING_CSV=/opt/energy-profiller-hiran/training_dataset.csv
```

---

## Step 4 — Start the Agent Server

```bash
cd /opt/green-agent
source venv/bin/activate
python agent_server.py
```

Expected startup output:

```
============================================================
  Green Deployment Agent Server
============================================================
  Port    : 5002
  Model   : qwen2.5:1.5b
  Ollama  : http://localhost:11434

  Main endpoint for Jenkins:
    POST http://localhost:5002/api/check

  Test endpoints:
    GET  http://localhost:5002/api/health
    GET  http://localhost:5002/api/tools/carbon
    GET  http://localhost:5002/api/tools/resources
    GET  http://localhost:5002/api/history
    GET  http://localhost:5002/api/stats
============================================================
```

---

## Step 5 — Verify the Agent is Working

Test the health endpoint:

```bash
curl http://localhost:5002/api/health
```

Test a carbon data read:

```bash
curl http://localhost:5002/api/tools/carbon
```

Send a test deployment decision request:

```bash
curl -X POST http://localhost:5002/api/check \
  -H "Content-Type: application/json" \
  -d '{"job_name": "test-pipeline", "build_number": "1"}'
```

---

## Running as a systemd Service (Recommended for Production)

Create `/etc/systemd/system/green-agent.service`:

```ini
[Unit]
Description=GreenDevOps Green AI Agent
After=network.target ollama.service

[Service]
Type=simple
User=hiran
WorkingDirectory=/opt/green-agent
ExecStart=/opt/green-agent/venv/bin/python agent_server.py
Environment=AGENT_PORT=5002
Environment=OLLAMA_URL=http://localhost:11434
Environment=OLLAMA_MODEL=qwen2.5:1.5b
Environment=WATCH_DIR=/opt/energy-profiller-hiran
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable green-agent
sudo systemctl start green-agent
sudo systemctl status green-agent
```

---

## Jenkins Integration

In your `Jenkinsfile`, the agent is called in the **Confirm Deploy Strategy** stage via the `greenCheck()` shared library function. The function sends a `POST /api/check` request and reads back the strategy.

The agent URL is set in the pipeline environment block:

```groovy
environment {
    GREEN_AGENT_URL = 'http://172.17.0.1:5002'
}
```

!!! note "Docker host IP"
    `172.17.0.1` is the Docker bridge gateway — the IP that Jenkins (running inside a container) uses to reach services running on the host machine. Adjust this if your network topology differs.

---

## Fallback Behaviour

If Ollama is unavailable (e.g. not installed, model not pulled, or service down), the agent automatically switches to **score-driven fallback mode**:

- The same five tools are called directly
- The Green Score is computed as normal
- The decision is made using score thresholds (≥70 → deploy, <50 → wait if window exists)
- The response includes `"method": "score_fallback"` so you can identify fallback decisions in `/api/stats`

No manual intervention is needed — the agent degrades gracefully.
