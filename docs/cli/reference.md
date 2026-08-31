# CLI Reference

The optimizer is invoked as a Python module:

```bash
python3 -m optimizer [OPTIONS]
```

Or inside the Docker image:

```bash
docker run --rm beliver247/build-optimizer-agent:latest python3 -m optimizer [OPTIONS]
```

---

## Flags

| Flag | Type | Default | Description |
|---|---|---|---|
| `--project-root <path>` | `string` | `.` (current directory) | Root of the Maven repository to analyse. |
| `--config <path>` | `string` | `config/default.yaml` (if present) | Path to a YAML or JSON config file. Absolute, or relative to CWD / project-root. |
| `--base <ref>` | `string` | `HEAD~1` (or `$GIT_PREVIOUS_SUCCESSFUL_COMMIT`) | Git ref to use as the "from" side of the diff. |
| `--head <ref>` | `string` | `HEAD` (or `$GIT_COMMIT`) | Git ref to use as the "to" side of the diff. |
| `--dry-run [true\|false]` | `bool` | `false` | Print Maven commands without executing them. Can be passed as a bare flag (`--dry-run`) or with an explicit value (`--dry-run true`). |
| `--output-format <fmt>` | `string` | `json` | Machine-readable output format. One of `json`, `key-value`, or `none`. |
| `--carbon-aware` | `flag` | off | Run the carbon-aware scheduling subsystem after the build analysis and include the recommendation in the output. |

---

## Environment Variables

The optimizer reads the following environment variables automatically.
CLI flags always take priority over environment variables.

| Variable | Equivalent flag | Notes |
|---|---|---|
| `GIT_PREVIOUS_SUCCESSFUL_COMMIT` | `--base` | Set by Jenkins Git plugin after a successful build. |
| `GIT_PREVIOUS_COMMIT` | `--base` | Fallback if `GIT_PREVIOUS_SUCCESSFUL_COMMIT` is not set. |
| `GIT_COMMIT` | `--head` | Set by Jenkins Git plugin on every build. |
| `ELECTRICITY_MAPS_API_KEY` | `carbon_aware.electricity_maps_api_key` | Takes priority over the config file value. |

---

## Exit Codes

The optimizer exits with one of the following codes.
In Jenkins, a non-zero exit code from `sh` will fail the stage unless you wrap it in `returnStatus: true`.

| Code | Constant | Status field | Meaning |
|---|---|---|---|
| `0` | `EXIT_SUCCESS` | `success` | Build analysis complete; Maven commands were (or would be) executed successfully. |
| `1` | `EXIT_ERROR` | `error` / `maven_failed` | An error occurred. See `stderr` and the `error` field in the JSON payload. |
| `10` | `EXIT_NO_CHANGES` | `no_changes` | No changed files were detected between `--base` and `--head`. |
| `20` | `EXIT_DOCS_ONLY` | `documentation_only` | All changed files are doc-only (e.g. `.md`, `.png`). Build intentionally skipped. |
| `30` | `EXIT_NO_AFFECTED_MODULES` | `no_affected_modules` | Files changed, but none could be mapped to a known Maven module. |

> [!TIP]
> Codes `10`, `20`, and `30` are **not failures** — they mean "nothing needs to be built right now." In your Jenkinsfile, gate downstream stages on `env.OPTIMIZER_STATUS == 'success'` rather than the exit code.

---

## Output Formats

### `json` (default)

A single-line JSON object is printed after the human-readable summary. Parse it in Groovy with `JsonSlurper`:

```groovy
def jsonLine = output.readLines().find { it.startsWith('{"') }
def result = new groovy.json.JsonSlurper().parseText(jsonLine)
```

### `key-value`

One `optimizer_<key>=<value>` pair per line. Useful for shell-based `eval` parsing:

```
optimizer_status=success
optimizer_exit_code=0
optimizer_elapsed_seconds=1.234
optimizer_affected_modules=auth-service,billing-service
optimizer_actions={"command":["mvn","-pl","auth-service","clean","install"],"dry_run":false,"name":"build"}
```

### `none`

Suppresses machine-readable output. The human-readable summary (changed files, affected modules, actions taken) is always printed regardless of this setting.

---

## Human-Readable Summary

Before the structured output, the optimizer always prints a human-readable summary to stdout:

```
Optimizer configuration:
  - git diff: abc123..def456
  - config: /workspace/config/default.yaml
Changed files:
  - auth-service/src/main/java/AuthController.java
Directly affected modules:
  - auth-service
Affected modules:
  - auth-service
  - api-gateway         ← pulled in via reverse dependency graph
Actions taken:
  - build: run (mvn -pl auth-service,api-gateway clean install)
  - test:  run (mvn -pl auth-service,api-gateway test)
Structured output:
{"actions":[...],"affected_modules":["api-gateway","auth-service"],...}
```

The JSON line always starts with `{"` and can be reliably extracted with `.find { it.startsWith('{"') }`.

---

## Full Usage Example

```bash
python3 -m optimizer \
  --project-root /workspace \
  --config /workspace/config/default.yaml \
  --base abc123def \
  --head 789abcdef \
  --dry-run false \
  --output-format json \
  --carbon-aware
```

Inside the Docker image (as used in production Jenkinsfiles):

```bash
tar -C "$PWD" -cf - . | docker run --rm -i \
  -e ELECTRICITY_MAPS_API_KEY="${ELECTRICITY_MAPS_API_KEY}" \
  -e GIT_PREVIOUS_SUCCESSFUL_COMMIT="${GIT_PREVIOUS_SUCCESSFUL_COMMIT}" \
  -e GIT_COMMIT="${GIT_COMMIT}" \
  beliver247/build-optimizer-agent:latest \
  bash -lc '
    mkdir -p /work && tar -xf - -C /work && cd /work
    git config --global --add safe.directory /work
    python3 -m optimizer \
      --project-root /work \
      --output-format json \
      --carbon-aware
  '
```

See [Full Jenkinsfile Walkthrough](../pipeline/full_jenkinsfile.md) for the complete production pipeline.
