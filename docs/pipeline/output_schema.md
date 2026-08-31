# Output Schema

The optimizer emits a structured JSON payload at the end of every run.
In Jenkins, this is parsed from stdout using `readLines().find { it.startsWith('{"') }`.

---

## Top-Level Fields

| Field | Type | Description |
|---|---|---|
| `status` | `string` | Final status. See [Exit Codes](../cli/reference.md#exit-codes) for all values. |
| `exit_code` | `int` | Process exit code: `0`, `1`, `10`, `20`, or `30`. |
| `elapsed_seconds` | `float` | Wall-clock time the optimizer took to run (seconds). |
| `base_ref` | `string` | The resolved "from" Git ref used for the diff. |
| `head_ref` | `string` | The resolved "to" Git ref used for the diff. |
| `config_path` | `string \| null` | Absolute path of the config file that was loaded. `null` if built-in defaults were used. |
| `changed_files` | `list[string]` | All files detected as changed by `git diff`. Empty when `status = "no_changes"`. |
| `directly_affected_modules` | `list[string]` | Modules that *directly* contain changed files (sorted). |
| `affected_modules` | `list[string]` | All modules that must be rebuilt, including transitive dependents (sorted). |
| `actions` | `list[Action]` | Maven actions to execute. Empty when nothing needs building. |
| `scheduling` | `Scheduling \| null` | Carbon-aware scheduling recommendation. Present only when `--carbon-aware` was passed. |
| `error` | `string` | Human-readable error message. Present only when `status = "error"`. |

---

## `Action` Object

Each entry in `actions` describes one Maven command to run:

| Field | Type | Description |
|---|---|---|
| `name` | `string` | Action type: `"build"` or `"test"`. |
| `command` | `list[string]` | The full Maven command as a list of tokens, e.g. `["mvn", "-pl", "auth-service", "clean", "install"]`. |
| `dry_run` | `bool` | Whether the command was printed but *not* executed (mirrors the `--dry-run` flag). |

**Example:**
```json
{
  "name": "build",
  "command": ["mvn", "-pl", "auth-service,api-gateway", "clean", "install"],
  "dry_run": false
}
```

**Parsing in Groovy:**
```groovy
def buildCmds = result.actions.findAll { it.name == 'build' }
                               .collect { it.command.join(' ') }
env.MAVEN_BUILD_COMMANDS = buildCmds.join('|||')  // '|||' as separator
```

---

## `Scheduling` Object

Present only when `--carbon-aware` is passed. Contains the carbon-aware scheduling recommendation.

| Field | Type | Description |
|---|---|---|
| `action` | `string` | `"execute_now"` — deploy immediately. `"schedule"` — delay to a greener hour. |
| `engine` | `string` | Decision engine used: `"lgbm"` (ML model) or `"rule_based"` (threshold fallback). |
| `green_probability` | `float \| null` | ML confidence (0–1) that now is a green window. Present only when `engine = "lgbm"`. |
| `current_intensity` | `float` | Carbon intensity at time of decision (gCO₂eq/kWh). |
| `scheduled_hour` | `int \| null` | Recommended hour (0–23, local time) to reschedule to. `null` when `action = "execute_now"`. |
| `target_intensity` | `float \| null` | Forecasted carbon intensity at `scheduled_hour`. `null` when `action = "execute_now"`. |
| `carbon_history` | `list[HistoryPoint]` | Recent carbon intensity readings used for the ML decision (for dashboard display). |
| `carbon_forecast` | `list[ForecastPoint]` | 24-hour carbon intensity forecast. |

### `HistoryPoint` Object

| Field | Type | Description |
|---|---|---|
| `timestamp` | `string` | ISO-8601 timestamp of the reading. |
| `intensity` | `float` | Carbon intensity value (gCO₂eq/kWh). |

### `ForecastPoint` Object

| Field | Type | Description |
|---|---|---|
| `hour` | `int` | Hour of day (0–23, local time). |
| `intensity` | `float` | Forecasted carbon intensity (gCO₂eq/kWh). |

---

## Full Example Payload

### Status: `success`, action: `execute_now`

```json
{
  "status": "success",
  "exit_code": 0,
  "elapsed_seconds": 3.142,
  "base_ref": "abc1234",
  "head_ref": "def5678",
  "config_path": "/workspace/config/default.yaml",
  "changed_files": [
    "auth-service/src/main/java/AuthController.java"
  ],
  "directly_affected_modules": ["auth-service"],
  "affected_modules": ["api-gateway", "auth-service"],
  "actions": [
    {
      "name": "build",
      "command": ["mvn", "-pl", "api-gateway,auth-service", "clean", "install"],
      "dry_run": false
    },
    {
      "name": "test",
      "command": ["mvn", "-pl", "api-gateway,auth-service", "test"],
      "dry_run": false
    }
  ],
  "scheduling": {
    "action": "execute_now",
    "engine": "lgbm",
    "green_probability": 0.87,
    "current_intensity": 132.4,
    "scheduled_hour": null,
    "target_intensity": null,
    "carbon_history": [
      {"timestamp": "2026-08-31T10:00:00", "intensity": 185.2},
      {"timestamp": "2026-08-31T11:00:00", "intensity": 160.1},
      {"timestamp": "2026-08-31T12:00:00", "intensity": 132.4}
    ],
    "carbon_forecast": [
      {"hour": 13, "intensity": 128.0},
      {"hour": 14, "intensity": 145.5},
      {"hour": 15, "intensity": 190.3}
    ]
  }
}
```

### Status: `documentation_only`

```json
{
  "status": "documentation_only",
  "exit_code": 20,
  "elapsed_seconds": 0.412,
  "base_ref": "abc1234",
  "head_ref": "def5678",
  "config_path": "/workspace/config/default.yaml",
  "changed_files": ["README.md", "docs/architecture.png"],
  "directly_affected_modules": [],
  "affected_modules": [],
  "actions": []
}
```

### Status: `no_changes`

```json
{
  "status": "no_changes",
  "exit_code": 10,
  "elapsed_seconds": 0.201,
  "base_ref": "abc1234",
  "head_ref": "abc1234",
  "config_path": null,
  "changed_files": [],
  "directly_affected_modules": [],
  "affected_modules": [],
  "actions": []
}
```

---

## Reading the Payload in Groovy

```groovy
// Find the JSON line in the raw optimizer output
def jsonLine = output.readLines().find { it.startsWith('{"') }
if (!jsonLine) {
    error "No JSON output found from optimizer"
}
def result = new groovy.json.JsonSlurper().parseText(jsonLine)

// Status & module list
env.OPTIMIZER_STATUS   = result.status ?: 'unknown'
env.AFFECTED_MODULES   = (result.affected_modules ?: []).join(',')

// Build and test commands (separated by ||| to survive env var serialisation)
def buildCmds = result.actions.findAll { it.name == 'build' }.collect { it.command.join(' ') }
def testCmds  = result.actions.findAll { it.name == 'test'  }.collect { it.command.join(' ') }
env.MAVEN_BUILD_COMMANDS = buildCmds.join('|||')
env.MAVEN_TEST_COMMANDS  = testCmds.join('|||')

// Carbon scheduling
if (result.scheduling) {
    env.CARBON_INTENSITY  = result.scheduling.current_intensity?.toString() ?: ''
    env.GREEN_PROBABILITY = result.scheduling.green_probability?.toString() ?: ''
    env.SCHEDULING_ACTION = result.scheduling.action ?: ''
    env.SCHEDULING_ENGINE = result.scheduling.engine ?: ''
}
```
