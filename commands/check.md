# speckit.workspace.check

Validate the blueprint catalog, workspace manifest, feature topology, and actual workspace paths against configured policies.

## Purpose

Run a comprehensive validation pass over all workspace materialization artifacts before task generation or implementation. Report violations, warnings, and pass/fail status.

## Usage

```
/speckit.workspace.check [options]
```

### Options

| Option | Description |
|--------|-------------|
| `--feature <feature-id>` | Validate against a specific feature topology |
| `--catalog-only` | Validate only the blueprint catalog |
| `--workspace-only` | Validate only the materialized workspace paths |
| `--strict` | Treat unresolved inputs as failures |

## Inputs

| File | Required | Description |
|------|----------|-------------|
| `AGENTS.md` | No | Agent capabilities and constraints |
| `.opencode/blueprints.yml` | No | Project-local blueprint catalog |
| `.opencode/workspace.yml` | No | Current workspace manifest |
| `.opencode/workspace.local.yml` | No | Local workspace overrides |
| `specs/<feature>/workspace-topology.yml` | No | Approved feature topology |
| `specs/<feature>/workspace-topology.proposed.yml` | No | Proposed topology (if no finalized topology) |
| `.specify/extensions/workspace-materialize/workspace-materialize-config.yml` | No | Materialization configuration |
| `.specify/extensions/workspace-materialize/workspace-materialize-config.template.yml` | No | Configuration defaults |

## Validation Rules

### Catalog Validation

1. Blueprint catalog exists at `.opencode/blueprints.yml` or the configured fallback path.
2. Blueprint refs are pinned (tag or SHA) if `catalog.require_pinned_refs` is true. Refuse if `ref_type` is `branch` when pinned refs are required.
3. Each blueprint entry defines required fields:
   - `id` — unique identifier in `area/key` format
   - `repository` — source repository (org/repo)
   - `area` — functional area (`frontend`, `backend`, `infrastructure`, etc.)
   - `workspace` with `path_template` and `command_project_template`
   - `materialization` with `default_mode` and `allowed_modes`
   - `parameters` — at least one parameter defined
4. `materialization.allowed_modes` contains only valid values: `copy`, `clone`, `template`.
5. `materialization.default_mode` is in `materialization.allowed_modes`.
6. Blueprint `id` values referenced in `project_profiles` components, `dependencies` lists, and `inter_blueprint_dependencies` must exist in the catalog.
7. `selection_rules` entries (if present) each have a non-empty `select_when` list.
8. Note: `runner` and `validation` are no longer defined on blueprints. These live on topology components and workspace manifest projects.

### Workspace Manifest Validation

6. Workspace `root` is defined and is a relative path under the repository root.
7. Each project entry defines:
   - `id` — unique project identifier
   - `area` — matches one of the known blueprint areas
   - `path` — relative path under workspace root
8. Project paths are inside the configured `workspace.root`.
9. Project paths match their declared area:
   - `frontend` projects under `workspace/frontend/`
   - `backend` projects under `workspace/backend/`
   - `infrastructure` projects under `workspace/infrastructure/`

### Materialized Path Validation

10. Materialized project directories exist on disk (unless `status` is `planned` or `pending`).
11. Existing materialized directories include `.blueprint-provenance.yml`, unless the manifest marks them as `external`.
12. Project directories are not empty (contain at least one file beyond provenance).

### Feature Topology Validation

13. Feature topology does not contain unresolved inputs in `--strict` mode.
14. All `action` values are valid (`reuse`, `materialize`, `defer`, `manual`, `profile`).
15. Component paths are inside the workspace root.
16. Each component `blueprint` reference exists in the blueprint catalog (by `id`).
17. Validation commands (on components) use only allowed wrappers:
    - `./dev/scripts/wrapper.sh`
    - `./dev/scripts/exec.sh`
18. Validation commands do not include forbidden direct tooling:
    - `pnpm`, `npm`, `npx`, `yarn`
    - `node`
    - `gradlew`, `mvn`, `java`
    - `tofu`, `terraform`
    - `docker`, `docker-compose`
    - `kubectl`, `helm`
    - Or any other direct binary invocation

### Policy Validation

19. Infrastructure components do not recommend apply in their validation commands (on topology or manifest).
    - `infrastructure:plan` is allowed.
    - `infrastructure:apply` is forbidden.
20. No raw `docker compose` commands appear in any validation list (topology or manifest).
21. No secrets are included in generated manifests (no API keys, tokens, passwords).
22. `policies.direct_tooling_allowed` is false or absent.

## Output

The command prints a structured validation report:

```
Workspace materialization check: pass|warning|fail
```

### Pass
All required artifacts are present and pass all applicable rules. No violations found.

### Warning
All required artifacts are present but low-severity issues were detected:
- Unpinned refs in a catalog that prefers pinned refs (non-strict)
- Missing optional configuration
- Proposed topology exists but is not yet approved

### Fail
One or more required validations failed:
- Missing required blueprint catalog
- Missing pinned refs when strictly required
- Materialized project directory missing or empty
- Missing provenance file in materialized project
- Forbidden tooling in validation commands
- Infrastructure apply in validation commands
- Path outside workspace root
- Unresolved inputs in strict mode

### Sections

```
Catalog
  Blueprint count: 5
  Pinned refs: 5/5
  Project profiles: 3 defined
  Selection rules: 4 areas defined
  Required fields: all present
  Status: pass

Workspace Manifest
  Projects: 3
  Path validity: 3/3 inside workspace root
  Area matching: 3/3
  Status: pass

Feature Topology
  Feature: my-feature
  Components: 4
  Actions: 2 reuse, 2 materialize
  Confidence: 3 high, 1 medium
  Unresolved inputs: 0
  Status: pass

Materialized Paths
  Existing directories: 3/3
  Provenance files: 3/3
  Non-empty: 3/3
  Status: pass

Policy Violations
  Infrastructure apply: 0
  Direct tooling: 0
  Raw docker compose: 0
  Secrets: 0
  Status: pass

Overall: pass

Next Actions:
  /speckit.tasks
  /speckit.workspace.status
```

## Safety Rules

- Do not modify files.
- Do not create or delete directories.
- Do not run validation commands; only check them.
- Do not change workspace state.
