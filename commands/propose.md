# speckit.workspace.propose

Propose a workspace topology for the current Spec Kit feature.

## Purpose

Generate a proposed workspace topology file that describes which blueprint repositories should be materialized, which existing workspace components can be reused, and which decisions remain unresolved.

This command does **not** clone, copy, delete, or modify application repositories. It produces a side-effect-free topology proposal.

## Usage

```
/speckit.workspace.propose
```

No arguments are required. The command resolves the current feature directory automatically.

## Inputs

The command reads (in priority order, latest wins):

| File | Required | Description |
|------|----------|-------------|
| `AGENTS.md` | No | Agent capabilities and constraints |
| `.specify/memory/constitution.md` | No | Project constitution |
| `.specify/extensions/workspace-materialize/workspace-materialize-config.yml` | No | Materialization configuration |
| `.specify/extensions/workspace-materialize/workspace-materialize-config.template.yml` | No | Configuration defaults |
| `.opencode/blueprints.yml` | No | Project-local blueprint catalog |
| `.specify/extensions/workspace-materialize/blueprints.yml` | No | Extension fallback catalog |
| `specs/<feature>/spec.md` | Yes | Feature specification |
| `specs/<feature>/plan.md` | No | Implementation plan |
| `.opencode/workspace.yml` | No | Current materialized workspace manifest |
| `.opencode/workspace.local.yml` | No | Local workspace overrides |

## Output

Creates or updates:

```
specs/<feature>/workspace-topology.proposed.yml
```

## Behavior

1. Resolve the current feature directory from the active Spec Kit context.

2. Load configuration from `workspace-materialize-config.yml` (with template defaults as fallback).

3. Load the blueprint catalog from `.opencode/blueprints.yml` (with extension fallback). The catalog uses a nested structure: top-level `blueprints` grouped by area (`frontend`, `backend`, `infrastructure`), each containing named blueprints keyed by their `id` (e.g., `frontend/ui-apollo`). The catalog may also include `project_profiles`, `selection_rules`, and `inter_blueprint_dependencies`.

4. Load the existing workspace manifest from `.opencode/workspace.yml`, if present.

5. Read the feature specification (`spec.md`) to understand functional requirements.

6. Read the implementation plan (`plan.md`), if present, to understand technical strategy.

7. Infer required areas using `selection_rules` from the catalog as the primary mechanism:
   - Match `selection_rules.<area>.select_when` conditions against spec and plan content.
   - If no `selection_rules` are defined in the catalog, fall back to manual inference:
     - `frontend` — UI, user-facing interactions, client-side logic
     - `backend` — API, services, data processing
     - `infrastructure` — provisioning, networking, platform config
     - `e2e` — end-to-end testing
     - `environment` — development environment configuration
   - Check `project_profiles` for a matching profile (e.g., `fullstack-web-app`, `local-mvp`, `infrastructure-only`). If a profile matches, use its `components` list as the starting template.
   - Record which profile (if any) matched and which specific rules triggered each area selection.

8. Reuse existing materialized components whenever they satisfy the need:
   - Match existing workspace projects by area and capability
   - Prefer reuse over new materialization
   - Surface conflicts when an existing component does not satisfy the spec

9. Propose new components only when the feature or plan introduces a new bounded component:
   - Select blueprints from the catalog using the `id` key (e.g., `frontend/ui-apollo`).
   - Apply naming conventions from configuration.
   - Generate workspace paths from `blueprint.workspace.path_template`.
   - Use `blueprint.materialization.default_mode` as the preferred mode.
   - Resolve required `blueprint.parameters` from command-line `--set` values or mark as unresolved.
   - Check `blueprint.dependencies` and `inter_blueprint_dependencies` for additional components that may need to be included.
   - For infrastructure blueprints, consult `exports`/`imports` to understand value flow between components.

10. Do **not** create repositories.

11. Do **not** invent repository names. Use placeholders and unresolved inputs when needed:
    - `<project>` for the project slug
    - `<name>` for unspecified names
    - Mark unresolved inputs explicitly

12. Mark output `status` as `proposed` or `needs-review`:
    - `proposed`: all inputs resolved, ready for review
    - `needs-review`: unresolved inputs or low-confidence selections remain

13. Include confidence levels for all proposed components:
    - `high`: spec and plan clearly indicate this component
    - `medium`: reasonably inferred from spec or plan
    - `low`: speculative, needs human confirmation

14. Include reasons for each selected or proposed blueprint.

15. Compose validation commands for each component. Since `runner` and `validation` are no longer defined on blueprints, derive commands from project conventions:
    - Use `blueprint.workspace.command_project_template` to construct the project name portion.
    - Use only wrapper scripts:
      - `./dev/scripts/wrapper.sh`
      - `./dev/scripts/exec.sh`
    - Example: `./dev/scripts/wrapper.sh frontend:test {project}-ui`

16. Include unresolved inputs for:
    - Project name
    - Package name
    - Target domain
    - Provider choice
    - Ambiguous ownership

17. Never recommend infrastructure apply in validation commands or notes.

## Output Shape

```yaml
schema_version: "1.0"
feature: "<feature-id>"
status: "needs-review"
source_artifacts:
  spec: "specs/<feature>/spec.md"
  plan: "specs/<feature>/plan.md"

summary:
  required_areas:
    - "frontend"
    - "backend"
  confidence: "medium"
  profile: "local-mvp"
  selection_rules_triggered:
    - "frontend: feature contains user-visible pages"
    - "backend: feature requires GraphQL API schema"
  notes:
    - "Plan indicates UI and API changes."

components:
  - id: "frontend-main"
    area: "frontend"
    action: "materialize"
    blueprint: "frontend/ui-apollo"
    existing_workspace_id: null
    project: "<project>-ui"
    path: "workspace/frontend/<project>-ui"
    materialization_mode: "copy"
    runner: "node-runner"
    validation:
      - "./dev/scripts/wrapper.sh frontend:test <project>-ui"
      - "./dev/scripts/wrapper.sh frontend:build <project>-ui"
    confidence: "medium"
    reason: "User-visible UI flow requires frontend implementation."

unresolved_inputs:
  - key: "project"
    question: "What project slug should be used for materialized repositories?"
    default: "<project>"

policies:
  infrastructure_apply_allowed: false
  direct_tooling_allowed: false
```

## Action Values

| Action | Meaning |
|--------|---------|
| `reuse` | Existing materialized component satisfies the need |
| `materialize` | New component should be materialized from blueprint |
| `defer` | Deliberately deferred to a later phase |
| `manual` | Requires manual setup; cannot be automated |
| `profile` | Use a project profile as-is without changes |

## Safety Rules

- Do not clone, copy, delete, or modify any application repository.
- Do not write to `.opencode/workspace.yml`.
- Do not write to `specs/<feature>/workspace-topology.yml` (only the `.proposed.yml` variant).
- Do not run validation commands; only include them in the output.
- Do not overwrite existing files unless the command is explicitly re-run for the same feature.

## Next Command

After review and approval of the topology:

```
/speckit.workspace.materialize --dry-run
```
