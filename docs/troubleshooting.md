# Troubleshooting

This page covers the most common errors and how to resolve them.

---

## Build Optimizer Errors

### `ConfigurationError: Could not read config file`

```
Error: Could not read config file /workspace/config/default.yaml: [Errno 2] No such file or directory
```

**Cause:** The optimizer resolved a config path that doesn't exist.

**Fix:** Either create `config/default.yaml` in your project root, or pass `--config` with the correct path. If you want to use built-in defaults, do not pass `--config` at all.

---

### `ConfigurationError: 'modules' must be a list`

**Cause:** Your config file has `modules:` set to a non-list value (e.g. `modules: null` or `modules: auth-service`).

**Fix:**
```yaml
# Correct — empty list for auto-discovery
modules: []

# Correct — explicit list
modules:
  - name: auth-service
    path: auth-service
```

---

### `ConfigurationError: carbon_aware.provider must be 'mock' or 'electricity_maps'`

**Cause:** An invalid value was supplied for `carbon_aware.provider`.

**Fix:** Use exactly `mock` or `electricity_maps` (lowercase):
```yaml
carbon_aware:
  provider: electricity_maps
```

---

### `GitError: ...`

```
Error: fatal: bad object abc123
```

**Cause:** The `--base` or `--head` ref does not exist in the repository. This often happens with shallow clones.

**Fix:** Ensure your Git checkout uses `shallow: false, depth: 0` in the Jenkinsfile:
```groovy
extensions: [[ $class: 'CloneOption', shallow: false, depth: 0, noTags: false ]]
```

For the very first commit in a repository (no `HEAD~1` exists), use the empty tree SHA as the base:
```bash
BASE_SHA=4b825dc642cb6eb9a060e54bf8d69288fbee4904
```

---

### `ImpactAnalysisError`

**Cause:** An internal error in the module-to-file mapping. Usually caused by a module path that doesn't exist on disk.

**Fix:** Check that `modules[].path` in your config matches the actual directory structure relative to `--project-root`.

---

## Carbon Scheduling Warnings

### `⚠  ML predictor unavailable`

```
[GreenOptimizer] ⚠  ML predictor unavailable: No module named 'lightgbm'
[GreenOptimizer]    Using rule-based scheduling engine instead.
```

**Cause:** The `lightgbm` or `scikit-learn` Python package is not installed in the environment running the optimizer.

**Impact:** The optimizer falls back to the rule-based `SchedulingDecisionEngine`. This is **not a failure** — scheduling still works, but without the ML model's `green_probability` score.

**Fix (Docker):** The official Docker image `beliver247/build-optimizer-agent:latest` bundles all dependencies including `lightgbm`. If you're running locally, install the full requirements:
```bash
pip install -r requirements.txt
```

---

### `⚠  Could not fetch current carbon intensity`

```
[GreenOptimizer] ⚠  Could not fetch current carbon intensity: 401 Client Error: Unauthorized
Using last known value from history store.
```

**Cause:** The Electricity Maps API key is invalid, expired, or not set.

**Fix:**
1. Verify the key at [app.electricitymaps.com](https://app.electricitymaps.com/).
2. In Jenkins, go to **Manage Jenkins → Credentials** and update the `electricity-maps-key` secret.
3. Confirm the env var is injected in your Jenkinsfile:
   ```groovy
   env.ELECTRICITY_MAPS_API_KEY = credentials('electricity-maps-key')
   ```

---

### `⚠  History backfill failed`

```
[GreenOptimizer] ⚠  History backfill failed: 403 Forbidden. Proceeding with partial history.
```

**Cause:** Your Electricity Maps plan may not include historical data access.

**Impact:** The ML model uses fewer hours of history than ideal. If fewer than `min_history_hours` are available, the rule-based engine is used automatically.

**Fix:** Set `backfill_on_empty: false` to suppress the backfill attempt, or upgrade your Electricity Maps plan.

---

## Jenkins Pipeline Issues

### Optimizer JSON not found in output

**Symptom:** `env.OPTIMIZER_STATUS` is `unknown` and all downstream stages are skipped.

**Cause:** The JSON line was not found in the optimizer output. This can happen if the `sh` step output is captured but the optimizer printed an error to `stderr` instead of `stdout`.

**Fix:** Look at the raw `output` variable in Jenkins logs. If the optimizer failed with a `ConfigurationError` or `GitError`, fix the underlying error first. Then ensure the Groovy parsing uses:
```groovy
def jsonLine = output.readLines().find { it.startsWith('{"') }
```

---

### `git config --global --add safe.directory /work`

**Symptom:** Docker stage fails with:
```
fatal: detected dubious ownership in repository at '/work'
```

**Cause:** Git ≥ 2.35.2 refuses to operate in directories owned by a different user. Inside the container, `/work` is owned by the host's Jenkins user UID.

**Fix:** The reference Jenkinsfile already includes the `git config --global --add safe.directory /work` line before running the optimizer. If you're seeing this error, confirm it is present in your `bash -lc '...'` block.

---

### Maven build fails with `Module not found`

**Symptom:**
```
[ERROR] The project com.example:billing-service ... does not exist
```

**Cause:** The optimizer generated a `-pl` command referencing a module that doesn't exist at that path, or the Maven project has not been built yet (no `target/` directories).

**Fix:** Check that `modules[].path` is correct, and that `modules[].artifact_id` matches the `<artifactId>` in the module's `pom.xml`.

---

## Exit Code Reference

| Code | Meaning | Is it a failure? |
|---|---|---|
| `0` | Success | No |
| `1` | Error or Maven build failed | **Yes** |
| `10` | No changed files detected | No |
| `20` | Documentation-only changes | No |
| `30` | No affected modules found | No |

Codes `10`, `20`, and `30` mean the optimizer made a deliberate decision to skip the build. Gate downstream stages on `env.OPTIMIZER_STATUS == 'success'` rather than the exit code.

---

## Getting More Diagnostic Information

Run the optimizer with verbose output to see exactly what it's doing:

```bash
python3 -m optimizer \
  --project-root . \
  --dry-run true \
  --output-format json \
  --carbon-aware
```

The human-readable summary printed to stdout includes:
- The exact git refs used (`git diff base..head`)
- The config file path that was loaded
- Every changed file
- Which modules were directly vs. transitively affected
- The exact Maven commands that would run

See [CLI Reference](cli/reference.md) for all available flags.
