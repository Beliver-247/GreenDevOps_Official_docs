# GreenDevOps Build Optimizer — Deep Dive

This document explains exactly how the Python engine (`optimizer/core.py`) dynamically maps changed files to Maven modules and calculates the "blast radius" of dependencies to guarantee safe selective builds.

> [!NOTE]
> This page covers the **build analysis** subsystem only.
> For carbon-aware scheduling internals, see [Carbon-Aware Scheduling](concepts/carbon_aware_scheduling.md).

---

## 1. Module Discovery

Before the optimizer can do anything, it needs to understand the structure of your project. It does this completely dynamically by searching for `pom.xml` files — no hardcoded project structure required.

Discovery tries three strategies in order:

### Strategy 1: Explicit Config

If your config file lists modules explicitly, those are used directly:

```yaml
modules:
  - name: auth-service
    path: services/auth-service
```

### Strategy 2: Root `pom.xml` `<modules>` Declaration

If no explicit modules are configured, the optimizer reads `<modules>` from the root `pom.xml`:

```xml
<modules>
  <module>auth-service</module>
  <module>billing-service</module>
</modules>
```

### Strategy 3: Top-Level Directory Scan (Fallback)

If neither strategy succeeds, the optimizer scans all top-level directories for a `pom.xml` file:

```python
top_level_modules = [
    child.name
    for child in sorted(project_root.iterdir())
    if child.is_dir() and (child / "pom.xml").is_file()
]
```

**Why this matters:** This makes the optimizer reusable across virtually any standard Maven repository with zero configuration.

---

## 2. Dependency Graphing

Once modules are discovered, the optimizer builds a **Reverse Dependency Graph** in memory by parsing every `pom.xml`'s `<dependencies>` block.

```
Module A depends on Module B
→ Reverse graph: "If B changes, A must be rebuilt"
```

The graph is implemented as a `dict[module_name, set[dependent_names]]`:

```python
# If module B is changed, reverse_deps["B"] = {"A", "C"} means
# both A and C must also be rebuilt.
reverse_deps: dict[str, set[str]] = {
    "common-lib":     {"auth-service", "billing-service", "api-gateway"},
    "auth-service":   {"api-gateway"},
}
```

Only **internal** dependencies (same `<groupId>`) are tracked. External library dependencies (e.g. Spring, Jackson) are ignored.

---

## 3. Impact Analysis (Git Diff Mapping)

The optimizer fetches the `git diff` between `--base` and `--head` and maps every changed file to the module that owns it based on the directory path.

### Documentation-Only Detection

Files matching the doc-only rules (e.g. `.md`, `.png`, `.txt`) are tracked for display but excluded from impact calculation.

If **all** changed files are doc-only and `rules.skip_non_code_changes: true`, the optimizer exits with code `20` (`documentation_only`) — no build needed.

### Global Trigger Paths

If a **global trigger** file changes (e.g. root `pom.xml`, `.mvn/`, `settings.xml`), **all modules** are marked as directly affected, forcing a full rebuild.

```python
if is_global_trigger_path(path, rules.global_trigger_paths):
    directly_affected.update(modules.keys())
```

This is the safety net for build configuration changes.

---

## 4. Blast Radius Expansion

Finally, the optimizer merges the directly-affected modules from step 3 with the reverse dependency graph from step 2.

It recursively expands the affected set using a BFS traversal:

```
directly_affected = {"billing-service"}

Expand:
  billing-service → dependents: {"api-gateway"}
  api-gateway     → dependents: {}   (no further dependents)

Final affected_modules = {"billing-service", "api-gateway"}
```

```python
def expand_dependents(start_modules, reverse_deps):
    affected = set(start_modules)
    queue = deque(start_modules)

    while queue:
        module = queue.popleft()
        for dependent in reverse_deps.get(module, set()):
            if dependent not in affected:
                affected.add(dependent)
                queue.append(dependent)

    return affected
```

**Why this matters:** By recursively resolving the graph, the optimizer guarantees that no downstream code breaks from an upstream change — preserving 100% CI/CD reliability while skipping everything that genuinely doesn't need rebuilding.

---

## 5. Maven Command Construction

Once the final `affected_modules` set is known, the optimizer constructs `mvn` commands using Maven's `-pl` (project list) flag:

```
mvn -pl auth-service,api-gateway clean install
mvn -pl auth-service,api-gateway test
```

These commands are placed in the `actions` array of the JSON output, ready to be parsed and executed by your Jenkinsfile.

---

## End-to-End Example

Given this repository:

```
my-project/
├── pom.xml               ← root, declares modules
├── common-lib/
├── auth-service/         ← depends on common-lib
├── billing-service/      ← depends on common-lib
└── api-gateway/          ← depends on auth-service + billing-service
```

And this commit: *"Fix null check in AuthController.java"* (only `auth-service/` changed)

| Step | Result |
|---|---|
| 1. Discover | 4 modules: `common-lib`, `auth-service`, `billing-service`, `api-gateway` |
| 2. Graph | `auth-service → {api-gateway}`, `common-lib → {auth-service, billing-service, api-gateway}` |
| 3. Git diff | `auth-service/src/main/java/AuthController.java` → directly affects `auth-service` |
| 4. Expand | `auth-service` → also affects `api-gateway` |
| 5. Commands | `mvn -pl auth-service,api-gateway clean install` |

**Result:** Only 2 of 4 modules rebuilt. `common-lib` and `billing-service` are completely skipped.
