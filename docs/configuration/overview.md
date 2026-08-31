# Configuration Overview

The GreenDevOps optimizer is driven by a single YAML (or JSON) configuration file.
Almost every behaviour — from which Maven modules to build, to how the carbon scheduler connects to the Electricity Maps API — is controlled through this file.

---

## Config File Lookup Order

The optimizer resolves the config file in the following order of priority:

```
1. --config <path>          (CLI flag, absolute or relative)
2. config/default.yaml      (conventional location, relative to --project-root)
3. Built-in defaults        (used when no file is found)
```

This means you can drop a `config/default.yaml` next to your root `pom.xml` and the optimizer will pick it up automatically — no extra flags required.

> [!TIP]
> If you supply a relative path to `--config`, the optimizer first checks it relative to the **current working directory**, then relative to `--project-root`. Absolute paths are used as-is.

---

## File Format

Both **YAML** and **JSON** are supported. The file extension (`.yaml`, `.yml`, or `.json`) determines the parser.

`PyYAML` is used if installed. If it is not, the optimizer falls back to a built-in minimal YAML parser that handles dictionaries, lists, booleans, nulls, and quoted strings — sufficient for all documented config keys.

### Minimal YAML Example

```yaml
maven:
  group_id: com.example

carbon_aware:
  provider: electricity_maps
  electricity_maps_api_key: null   # Set via env var ELECTRICITY_MAPS_API_KEY
  electricity_maps_zone: GB
```

### Equivalent JSON

```json
{
  "maven": {
    "group_id": "com.example"
  },
  "carbon_aware": {
    "provider": "electricity_maps",
    "electricity_maps_api_key": null,
    "electricity_maps_zone": "GB"
  }
}
```

---

## How CLI Flags Override Config Values

The following CLI flags, when provided, **always override** the corresponding config file setting:

| CLI Flag | Config key | Notes |
|---|---|---|
| `--base <ref>` | `git.base_ref` | Also auto-read from `$GIT_PREVIOUS_SUCCESSFUL_COMMIT` |
| `--head <ref>` | `git.head_ref` | Also auto-read from `$GIT_COMMIT` |
| `--dry-run` | `dry_run` | Accepts `true`/`false` or bare flag |
| `--output-format` | `output.format` | `json`, `key-value`, or `none` |

> [!NOTE]
> `$GIT_PREVIOUS_SUCCESSFUL_COMMIT` and `$GIT_COMMIT` are set automatically by Jenkins when using the Git plugin. The optimizer reads them as a convenience so you don't have to pass `--base` / `--head` explicitly in most pipelines.

---

## Environment Variable Precedence

For sensitive values, environment variables take priority over the config file:

| Environment variable | Config key |
|---|---|
| `ELECTRICITY_MAPS_API_KEY` | `carbon_aware.electricity_maps_api_key` |

---

## Next Steps

- [Full Config Reference](reference.md) — every available key with type, default, and description
- [Annotated Examples](examples.md) — copy-paste ready configs for common scenarios
