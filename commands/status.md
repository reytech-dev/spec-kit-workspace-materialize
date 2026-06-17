# speckit.workspace.status

Report the current materialized workspace state and unresolved decisions.

## Purpose

Print a human-readable overview of the workspace materialization state. Entirely read-only; makes no changes to files or directories.

## Usage

```
/speckit.workspace.status
```

No arguments are required. The command resolves the current feature directory automatically.

## Inputs

| File | Required | Description |
|------|----------|-------------|
| `.opencode/blueprints.yml` | No | Project-local blueprint catalog |
| `.opencode/workspace.yml` | No | Current materialized workspace manifest |
| `.opencode/workspace.local.yml` | No | Local workspace overrides |
| `specs/<feature>/workspace-topology.yml` | No | Active feature topology (if a feature is active) |

## Output

The command prints a structured status report. The exact format depends on available data; missing sections are omitted.

```
=== Workspace Status ===

Workspace root: workspace/
Catalog source: .opencode/blueprints.yml
Catalog name: Reytech opencode Blueprint Catalog
Known blueprint count: 5
Project profiles: 3
Selection rules: 4 areas defined
Materialized project count: 3

Projects by area:
  backend (1):
    <project>-api          [materialized]  workspace/backend/<project>-api

  frontend (1):
    <project>-ui           [materialized]  workspace/frontend/<project>-ui

  infrastructure (1):
    <project>-bootstrap-storage  [materialized]  workspace/infrastructure/<project>-bootstrap-storage

Unresolved topology decisions: 0

Validation commands (wrapper-only):
  <project>-api:
    ./dev/scripts/wrapper.sh backend:test <project>-api
  <project>-ui:
    ./dev/scripts/wrapper.sh frontend:install <project>-ui
    ./dev/scripts/wrapper.sh frontend:test <project>-ui
    ./dev/scripts/wrapper.sh frontend:build <project>-ui
  <project>-bootstrap-storage:
    ./dev/scripts/wrapper.sh infrastructure:validate <project>-bootstrap-storage
    ./dev/scripts/wrapper.sh infrastructure:plan <project>-bootstrap-storage

Recommended next command:
  /speckit.workspace.check
```

### Active Feature Topology

If a feature topology is active, the command includes a feature-specific section:

```
Active feature topology: specs/my-feature/workspace-topology.yml
  Status: approved
  Components: 4
    frontend-main      [reuse]        frontend/ui-apollo
    backend-main       [materialize]  backend/java-graphql
    infra-storage      [reuse]        infra/bootstrap-storage
    infra-dns          [defer]        infra/cloudflare
  Unresolved inputs: 0
```

### Empty Workspace State

If no workspace manifest or catalog exists, the command reports:

```
=== Workspace Status ===

Workspace root: workspace/ (configured default)
Catalog source: not found
Catalog name: n/a
Known blueprint count: 0
Project profiles: 0
Materialized project count: 0

Projects by area:
  (none)

Recommended next command:
  /speckit.workspace.propose
```

## Safety Rules

- Do not modify files.
- Do not create or delete directories.
- Do not run any project tooling.
- Do not run validation commands; only display them.
- Do not change workspace state.
