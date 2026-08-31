# Configuration Examples

Copy-paste ready config files for the most common GreenDevOps scenarios.

---

## 1. Minimal Config (Auto-Discovery)

No configuration needed for a standard Maven monorepo. Drop a `config/default.yaml` at your project root and leave `modules` empty:

```yaml
# config/default.yaml
maven:
  group_id: com.example
```

The optimizer will:
- Auto-discover modules from `<modules>` in `pom.xml`, or by scanning top-level directories.
- Use `HEAD~1..HEAD` as the diff range (Jenkins populates this automatically via `$GIT_COMMIT`).
- Skip builds when only documentation files change.

---

## 2. Multi-Service Monorepo with Explicit Modules

Use explicit module declarations when your module directories have non-standard names, or when `artifactId` differs from the directory name:

```yaml
# config/default.yaml
maven:
  group_id: com.acme.platform

modules:
  - name: user-service
    path: services/user
    artifact_id: user-service

  - name: order-service
    path: services/order
    artifact_id: order-service

  - name: notification-service
    path: services/notifications
    artifact_id: notification-service

  - name: shared-models
    path: libs/shared-models
    artifact_id: shared-models

shared_modules:
  - shared-models
```

With `shared_modules: [shared-models]`, any change to `libs/shared-models/` triggers a rebuild of all three services.

---

## 3. Carbon-Aware Scheduling with Electricity Maps

Enable real carbon data for a production pipeline:

```yaml
# config/default.yaml
maven:
  group_id: com.example

carbon_aware:
  provider: electricity_maps

  # API key — set as a Jenkins credential and injected via env var.
  # The ELECTRICITY_MAPS_API_KEY env var overrides this setting.
  electricity_maps_api_key: null

  # Your grid zone — find yours at https://www.electricitymaps.com/map
  electricity_maps_zone: GB

  # Persist the history on the Jenkins agent (outside the workspace so it survives clean builds)
  history_store_path: /var/jenkins_home/.greendevops/carbon_history.json

  # Use ML model after 6 hours of history (more cautious than the 3h default)
  min_history_hours: 6
  backfill_on_empty: true
```

**Jenkins credential setup:**

```groovy
environment {
    ELECTRICITY_MAPS_API_KEY = credentials('electricity-maps-key')
}
```

---

## 4. Build Only, Skip Tests

For a fast feedback pipeline where tests are run separately (e.g. on a nightly schedule):

```yaml
maven:
  group_id: com.example
  build_goals:
    - clean
    - package          # Use package instead of install to avoid polluting the local repo
  run_build: true
  run_tests: false     # No test action emitted
```

---

## 5. Customising the Doc-Only Skip Rules

By default, changes to `.md`, `.png`, `.pdf`, etc. are skipped. If your project uses `.adoc` source files that actually affect the build output (e.g. generated documentation artifacts), remove them from the skip list:

```yaml
rules:
  skip_non_code_changes: true
  doc_only_extensions:
    - .md
    - .txt
    - .png
    - .gif
    - .jpg
    # Note: .adoc removed — AsciiDoc sources trigger builds in this project

  global_trigger_paths:
    - pom.xml
    - .mvn/
    - mvnw
    - settings.xml
    - docker-compose.yml    # Adding compose file as a global trigger
```

---

## 6. Dry-Run Analysis Mode

Run the optimizer in analysis-only mode to preview what *would* be built, without executing anything:

```yaml
dry_run: true

output:
  format: json

carbon_aware:
  provider: mock       # No API key needed for dry-run analysis
```

Or pass `--dry-run` on the CLI (overrides the config):

```bash
python3 -m optimizer \
  --project-root . \
  --dry-run true \
  --output-format json \
  --carbon-aware
```

---

## 7. CI Container (Minimal Dependencies)

When running in a lean CI container without `PyYAML` or `lightgbm` installed, the optimizer degrades gracefully:

```yaml
carbon_aware:
  provider: mock                  # No API calls needed
  min_history_hours: 0            # Rule-based engine always used
  backfill_on_empty: false        # No backfill API calls

output:
  format: key-value               # No JSON parser needed in shell scripts
```

The built-in YAML parser handles this file without `PyYAML`.
