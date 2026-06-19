# speckit.workspace-materialize.materialize

Materialize missing workspace repositories from an approved workspace topology.

## Purpose

Read an approved workspace topology and create or populate workspace directories for each component marked `action: materialize`. This command is the single point where application repositories enter the workspace.

## Usage

```
/speckit.workspace-materialize.materialize [options]
```

### Options

| Option | Description |
|--------|-------------|
| `--from <path>` | Path to a specific topology file |
| `--dry-run` | Report what would happen without making changes (default) |
| `--apply` | Execute materialization (requires explicit opt-in) |
| `--force` | Skip safety checks (non-empty targets, unapproved topology) |
| `--project <slug>` | Materialize only components matching the project slug |
| `--set key=value` | Supply a value for an unresolved input |
| `--mode copy|clone|template` | Override the default materialization mode |

Default mode is `--dry-run`. No files are created, modified, or deleted in dry-run mode.

## Inputs

| File | Required | Description |
|------|----------|-------------|
| `AGENTS.md` | No | Agent capabilities and constraints |
| `.specify/extensions/workspace-materialize/workspace-materialize-config.yml` | No | Materialization configuration |
| `.specify/extensions/workspace-materialize/workspace-materialize-config.template.yml` | No | Configuration defaults |
| `.opencode/blueprints.yml` | Yes | Project-local blueprint catalog |
| `.opencode/workspace.yml` | No | Current materialized workspace manifest |
| `specs/<feature>/workspace-topology.yml` | No | Approved feature topology |
| `specs/<feature>/workspace-topology.proposed.yml` | No | Proposed topology (if no finalized topology exists) |

## Outputs

In `--apply` mode, creates or updates:

```
.opencode/workspace.yml
specs/<feature>/workspace-topology.yml
workspace/<area>/<project>/
workspace/<area>/<project>/.blueprint-provenance.yml
```

## Behavior

### 1. Resolve Topology Input

- If `--from <path>` is provided, use that topology file.
- Otherwise, look for `specs/<feature>/workspace-topology.yml` (finalized).
- Fall back to `specs/<feature>/workspace-topology.proposed.yml` (proposed).

### 2. Status Gate

- Refuse to materialize if the topology `status` is not `approved`, unless `--force` is provided.
- Print a clear error explaining that the topology must be reviewed and approved first.

### 3. Unresolved Inputs Gate

- Refuse to continue if required unresolved inputs remain in the topology.
- Print a list of required inputs and their questions.
- Accept `--set key=value` pairs to resolve inputs at the command line.

### 4. Blueprint Resolution

- Resolve blueprint references from `.opencode/blueprints.yml` using the blueprint `id` field (e.g., `frontend/ui-apollo`).
- Read `default_ref`, `ref_type`, `repository`, and `repository_url` from the resolved blueprint.
- Refuse to materialize if a referenced blueprint is not found in the catalog.
- Enforce pinned refs when `catalog.require_pinned_refs` is true in config:
  - Refuse if `ref_type` is `branch` and the ref is a branch name (not a pinned tag or SHA).
  - Refuse if `default_ref` is a placeholder (`<pin-tag-or-sha>`, `main`, `latest`, etc.).
  - Require exact tags or commit SHAs.

### 5. Component Processing

For each component with `action: materialize`:

1. **Compute target path** from `blueprint.workspace.path_template` and component parameters.
   - Also resolve `blueprint.workspace.command_project_template` for use in provenance.
   - Also resolve `blueprint.workspace.repository_name_template` for repository naming.

2. **Verify target path** is inside the configured workspace root (`workspace/` by default).
   - Refuse if the path escapes the workspace root.

3. **Verify target path does not already contain unmanaged files**:
   - If the path exists but has no `.blueprint-provenance.yml`, consider it unmanaged.
   - Refuse to proceed unless `--force` is provided and `materialization.allow_non_empty_target` or `materialization.allow_overwrite` is true in config.

4. **Resolve materialization mode**:
   - From `--mode` argument, or
   - From component-level `materialization_mode` in topology, or
   - From `blueprint.materialization.default_mode`.
   - Mode must be in `blueprint.materialization.allowed_modes`. Refuse otherwise.
   - Default to `copy` if no mode is specified anywhere.

### 6. Materialization Modes

#### `copy`
- Copy blueprint repository contents into the target path.
- Remove the upstream `.git` directory so the project becomes standalone.
- Document the action; do not assume GitHub write permissions.

#### `clone`
- Clone the blueprint repository at the pinned ref into the target path.
- Detach the clone from the blueprint upstream (remove or replace `origin` remote).
- Document the action; do not assume GitHub write permissions.

#### `template`
- Create a project repository from a template, if the runtime supports it.
- If template creation is not available, report the required manual action.
- Do not assume GitHub API access or write permissions.

### 7. Write Provenance

After materializing each component, write `.blueprint-provenance.yml` at the component root:

```yaml
schema_version: "1.0"
blueprint:
  repository: "reytech-dev/java-graphql-blueprint"
  repository_url: "https://github.com/reytech-dev/java-graphql-blueprint"
  ref: "v0.1.0"
  ref_type: "tag"
  blueprint_key: "backend/java-graphql"
materialized:
  at: "<timestamp>"
  by: "speckit.workspace-materialize.materialize"
  mode: "copy"
project:
  id: "<project>-api"
  area: "backend"
  path: "workspace/backend/<project>-api"
  command_project_template: "{project}-api"
parameters:
  project: "<project>"
```

### 8. Update Workspace Manifest

Update `.opencode/workspace.yml` with materialized components.

Create the file if it does not exist. Merge new entries with existing entries. Do not remove entries for components that were not part of the current materialization run.

```yaml
schema_version: "1.0"
workspace:
  root: "workspace"
  status: "materialized"

projects:
  - id: "<project>-api"
    area: "backend"
    path: "workspace/backend/<project>-api"
    blueprint: "backend/java-graphql"
    repository_source: "reytech-dev/java-graphql-blueprint"
    repository_url: "https://github.com/reytech-dev/java-graphql-blueprint"
    ref: "v0.1.0"
    ref_type: "tag"
    runner: "java-runner"
    validation:
      - "./dev/scripts/wrapper.sh backend:test <project>-api"
    status: "materialized"
```

### 9. Update Feature Topology

Update `specs/<feature>/workspace-topology.yml` from the proposed topology, marking materialized components with their result status.

### 10. Dry-Run Output

In dry-run mode, print a summary:

```
Dry-run summary:
  Topology: specs/<feature>/workspace-topology.proposed.yml
  Components to materialize: 2
    - backend/<project>-api from backend/java-graphql (ref: v0.1.0) via copy
    - frontend/<project>-ui from frontend/ui-apollo (ref: v0.2.0) via copy
  Target workspace root: workspace/
  Mode: dry-run (use --apply to execute)
```

### 11. Apply Output

In apply mode, print a summary of actions taken and the next recommended command.

## Safety Rules

- Never overwrite existing files unless `--force` is provided and the target is confirmed safe.
- Never delete existing repositories or workspace directories.
- Never run package install, build, test, plan, or apply during materialization.
- Never run infrastructure apply.
- Never run raw `docker compose`.
- Never run direct project tooling (`pnpm`, `npm`, `node`, `npx`, `gradlew`, `tofu`, `docker`).
- Never write `.env` files.
- Never commit secrets.
- Never create repositories on remote hosting platforms unless explicitly confirmed.

## Next Command

After materialization:

```
/speckit.workspace-materialize.check
```
