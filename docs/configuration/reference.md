# Configuration Reference

This page documents every key in the GreenDevOps optimizer configuration file.
All keys are optional — the optimizer runs with built-in defaults if no config file is present.

---

## `git` — Git Ref Settings

Controls which commits are compared to detect changed files.

| Key | Type | Default | Description |
|---|---|---|---|
| `git.base_ref` | `string` | `HEAD~1` | The "from" Git ref. Overridable with `--base` or `$GIT_PREVIOUS_SUCCESSFUL_COMMIT`. |
| `git.head_ref` | `string` | `HEAD` | The "to" Git ref. Overridable with `--head` or `$GIT_COMMIT`. |

```yaml
git:
  base_ref: HEAD~1
  head_ref: HEAD
```

---

## `modules` — Maven Module List

Explicitly declares Maven modules. When this list is empty (the default), the optimizer **auto-discovers** modules by scanning the project for `pom.xml` files.

| Key | Type | Default | Description |
|---|---|---|---|
| `modules` | `list[object]` | `[]` | List of module definitions. Leave empty for auto-discovery. |
| `modules[].name` | `string` | *required* | Human-readable module name (used in log output and `AFFECTED_MODULES`). |
| `modules[].path` | `string` | Same as `name` | Path to the module directory, relative to `--project-root`. |
| `modules[].artifact_id` | `string` | `null` | Maven `<artifactId>` for dependency resolution. Required only when `artifactId` differs from `name`. |

```yaml
modules:
  - name: auth-service
    path: services/auth-service
    artifact_id: auth-service

  - name: billing-service
    path: services/billing-service
```

> [!NOTE]
> Auto-discovery tries three strategies in order:
> 1. Modules explicitly listed here (this setting).
> 2. Modules declared in `<modules>` in the root `pom.xml`.
> 3. Top-level subdirectories that contain a `pom.xml`.

---

## `shared_modules` — Always-Rebuild Modules

```yaml
shared_modules:
  - common-lib
```

| Key | Type | Default | Description |
|---|---|---|---|
| `shared_modules` | `list[string]` | `["common-lib"]` | Modules that trigger a rebuild of **all dependents** when changed. Useful for shared utility libraries. |

---

## `rules` — Change Classification Rules

Controls how changed files are categorised — whether they count as code changes, documentation-only changes, or global triggers.

| Key | Type | Default | Description |
|---|---|---|---|
| `rules.skip_non_code_changes` | `bool` | `true` | When `true`, changes to doc-only files are ignored and the build is skipped entirely. |
| `rules.doc_only_extensions` | `list[string]` | `.adoc .asciidoc .drawio .gif .jpeg .jpg .markdown .md .pdf .png .rst .svg .txt` | File extensions considered documentation-only. |
| `rules.doc_file_names` | `list[string]` | `changelog license readme readme.md` | File names (case-insensitive) considered documentation-only regardless of extension. |
| `rules.global_trigger_paths` | `list[string]` | `pom.xml .mvn/ mvnw mvnw.cmd settings.xml` | Paths that trigger a **full rebuild of all modules** when changed. |

```yaml
rules:
  skip_non_code_changes: true
  doc_only_extensions:
    - .md
    - .txt
    - .png
  doc_file_names:
    - changelog
    - license
    - readme
  global_trigger_paths:
    - pom.xml
    - .mvn/
    - mvnw
```

---

## `maven` — Maven Execution Settings

| Key | Type | Default | Description |
|---|---|---|---|
| `maven.executable` | `string` | `mvn` | Path to or name of the Maven executable. |
| `maven.group_id` | `string \| null` | `null` | Maven `<groupId>` of internal artifacts. Used to filter internal vs external dependencies during dependency graphing. |
| `maven.also_make` | `bool` | `false` | Adds `-am` (`--also-make`) to the Maven command to build upstream dependencies. |
| `maven.also_make_tests` | `bool` | `false` | Adds `-amd` (`--also-make-dependents`) for test commands. |
| `maven.extra_args` | `list[string]` | `[]` | Additional arguments appended to every Maven command (e.g. `-Dskip.integration.tests=true`). |
| `maven.build_goals` | `list[string]` | `["clean", "install"]` | Maven goals for the build action. |
| `maven.test_goals` | `list[string]` | `["test"]` | Maven goals for the test action. |
| `maven.run_build` | `bool` | `true` | Whether to emit a build action. Set to `false` to run tests only. |
| `maven.run_tests` | `bool` | `true` | Whether to emit a test action. Set to `false` to build without testing. |

```yaml
maven:
  executable: mvn
  group_id: com.example
  also_make: true
  extra_args:
    - -Dmaven.test.failure.ignore=false
  build_goals:
    - clean
    - install
  test_goals:
    - test
  run_build: true
  run_tests: true
```

---

## `output` — Structured Output Format

| Key | Type | Default | Description |
|---|---|---|---|
| `output.format` | `string` | `json` | Format for the machine-readable output block. One of `json`, `key-value`, or `none`. |

```yaml
output:
  format: json
```

**`json`** — Emits a single-line JSON object. Easiest to parse in Groovy with `JsonSlurper`.

**`key-value`** — Emits `optimizer_<key>=<value>` pairs, one per line. Useful for shell-based pipelines.

**`none`** — Suppresses machine-readable output entirely (human-readable summary still printed).

See [Output Schema](../pipeline/output_schema.md) for the full list of fields in the JSON payload.

---

## `carbon_aware` — Carbon-Aware Scheduling

Used only when `--carbon-aware` is passed on the CLI.

| Key | Type | Default | Description |
|---|---|---|---|
| `carbon_aware.provider` | `string` | `mock` | Carbon data provider. `"mock"` (no API key, synthetic data) or `"electricity_maps"` (real API). |
| `carbon_aware.electricity_maps_api_key` | `string \| null` | `null` | Electricity Maps API key. Overridden by `$ELECTRICITY_MAPS_API_KEY` env var. |
| `carbon_aware.electricity_maps_zone` | `string` | `LK` | Electricity Maps grid zone code (e.g. `GB`, `DE`, `US-CAL-CISO`). Default is Sri Lanka. |
| `carbon_aware.model_path` | `string \| null` | `null` | Path to a custom LightGBM model file (`.joblib`). `null` uses the model bundled with the plugin. |
| `carbon_aware.history_store_path` | `string` | `~/.greendevops/carbon_history.json` | Path to the rolling 168-hour carbon intensity history file. |
| `carbon_aware.min_history_hours` | `int` | `3` | Minimum hours of history required before the ML model is trusted. Below this, the rule-based engine is used. |
| `carbon_aware.backfill_on_empty` | `bool` | `true` | When `true`, automatically backfills the history store from the API on first run or when stale. |

```yaml
carbon_aware:
  provider: electricity_maps
  electricity_maps_api_key: null   # Set via ELECTRICITY_MAPS_API_KEY env var
  electricity_maps_zone: GB
  model_path: null
  history_store_path: ~/.greendevops/carbon_history.json
  min_history_hours: 3
  backfill_on_empty: true
```

---

## Top-Level Keys

| Key | Type | Default | Description |
|---|---|---|---|
| `dry_run` | `bool` | `false` | When `true`, prints Maven commands but does not execute them. Overridable with `--dry-run`. |

---

## Full Default Config

This is the complete `config/default.yaml` shipped with the plugin, reproduced here for reference:

```yaml
git:
  base_ref: HEAD~1
  head_ref: HEAD

modules: []

shared_modules:
  - common-lib

rules:
  skip_non_code_changes: true
  doc_only_extensions:
    - .adoc
    - .asciidoc
    - .drawio
    - .gif
    - .jpeg
    - .jpg
    - .markdown
    - .md
    - .pdf
    - .png
    - .rst
    - .svg
    - .txt
  doc_file_names:
    - changelog
    - license
    - readme
    - readme.md
  global_trigger_paths:
    - pom.xml
    - .mvn/
    - mvnw
    - mvnw.cmd
    - settings.xml

maven:
  executable: mvn
  group_id: null
  also_make: true
  also_make_tests: false
  extra_args: []
  build_goals:
    - clean
    - install
  test_goals:
    - test
  run_build: true
  run_tests: true

output:
  format: json

carbon_aware:
  provider: mock
  electricity_maps_api_key: null
  electricity_maps_zone: LK
  model_path: null
  history_store_path: ~/.greendevops/carbon_history.json
  min_history_hours: 3
  backfill_on_empty: true
```
