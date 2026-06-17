# spec-kit-workspace-materialize

Propose, materialize, validate, and report multi-repository opencode workspaces from blueprint catalogs.

## Purpose

This Spec Kit extension enables opencode agents to work across multiple project repositories within a single workspace. It creates and maintains workspace topology artifacts that describe which project repositories exist, where they live, which blueprint they came from, and which validation commands are safe to run.

The extension does **not** implement application features. It produces topology and provenance files that other extensions (such as `spec-kit-workspace-map`) consume.

## Canonical Blueprint Catalog

The primary blueprint catalog lives at:

```
.opencode/blueprints.yml
```

This is the project-local catalog that pins real tags or commit SHAs for each blueprint repository. The extension ships a template at `blueprints.template.yml` with placeholder refs; each project must copy this template and pin concrete versions.

## Key Concepts

### Blueprint Catalog

The catalog defines **available** blueprints — ready-made project templates organized by area (frontend, backend, infrastructure). Each blueprint includes `task_markers` for automated task routing, `workspace` path templates, `materialization` mode configuration, `parameters` schemas, and `dependencies` on other blueprints.

The catalog also includes:
- **`project_profiles`** — quick-start templates that bundle blueprint selections (e.g., `fullstack-web-app`, `local-mvp`, `infrastructure-only`).
- **`selection_rules`** — automated area inference rules for matching features to required blueprints.
- **`inter_blueprint_dependencies`** — dependency declarations between blueprints for dependency-aware materialization.

Blueprints are not materialized; they are sources that can be instantiated.

### Workspace Manifest

The manifest (`.opencode/workspace.yml`) records **materialized** workspace projects — projects that have been checked out or copied into the workspace. Each entry records the source blueprint, the pinned ref, and the validation commands.

### Feature Topology

A feature topology (`specs/<feature>/workspace-topology.yml`) maps a specific feature's needs onto the workspace. It declares which workspace components should be reused, which should be materialized, and which decisions remain unresolved.

### Difference Summary

| Artifact | Location | Contains |
|----------|----------|----------|
| Blueprint catalog | `.opencode/blueprints.yml` | Available blueprint definitions |
| Workspace manifest | `.opencode/workspace.yml` | Materialized project instances |
| Feature topology | `specs/<feature>/workspace-topology.yml` | Feature-specific component plan |

## Recommended Lifecycle

```
/speckit.specify                          # Write the feature spec
/speckit.plan                             # Create the implementation plan
/speckit.workspace.propose                # Propose workspace topology
/speckit.workspace.materialize --dry-run  # Preview materialization
/speckit.workspace.materialize --apply    # Materialize repos (requires approval)
/speckit.workspace.check                  # Validate everything
/speckit.tasks                            # Generate tasks
/speckit.workspace-map.generate           # Generate workspace map
```

## Commands

| Command | File | Description |
|---------|------|-------------|
| `/speckit.workspace.propose` | `commands/propose.md` | Propose workspace topology from spec and plan |
| `/speckit.workspace.materialize` | `commands/materialize.md` | Materialize repos from approved topology |
| `/speckit.workspace.check` | `commands/check.md` | Validate catalog, manifest, and paths |
| `/speckit.workspace.status` | `commands/status.md` | Print current workspace state |

## Safety Model

This extension enforces strict safety rules to prevent accidental damage:

### Hard Rules

1. Do not patch `.specify/scripts/*`.
2. Do not patch installed core Spec Kit commands.
3. Do not modify `spec.md`, `plan.md`, or `tasks.md`.
4. Do not run raw `docker compose`.
5. Do not run direct project tooling (`pnpm`, `npm`, `node`, `npx`, `gradlew`, `tofu`, `docker`).
6. Do not run `tofu apply` or any infrastructure apply.
7. Do not commit secrets.
8. Do not write `.env` files.
9. Do not overwrite non-empty directories by default.
10. Do not delete repositories or workspace directories.
11. Do not invent repository names.
12. Do not silently ignore unresolved inputs.
13. Do not materialize anything unless explicitly run in apply mode.
14. Dry-run must be the default for materialize.
15. Generated artifacts must mark low-confidence assumptions.

### Validation Commands

All validation commands use only the wrapper scripts through the orchestrator:

```
./dev/scripts/wrapper.sh
./dev/scripts/exec.sh
```

Agents must never run project tooling directly. Tooling commands are routed through these scripts, which the orchestrator controls.

## Example Blueprint Catalog

See `examples/blueprints.yml` for a populated example with:
- `frontend/ui-apollo` — React/Apollo frontend with Apollo Client, Vite, TypeScript, Vitest, Playwright
- `backend/java-graphql` — Java Spring GraphQL backend with WebFlux, R2DBC, Flyway, Testcontainers
- `infrastructure/bootstrap-storage` — OpenTofu S3-compatible remote state backend bootstrap
- `infrastructure/digitalocean` — DigitalOcean compute/network infrastructure
- `infrastructure/cloudflare` — Cloudflare DNS and edge configuration

Includes 3 project profiles (`fullstack-web-app`, `local-mvp`, `infrastructure-only`), selection rules for automatic area inference, inter-blueprint dependency declarations, task markers for routing, and full parameter schemas per blueprint.

## Example Workspace Manifest

See `examples/workspace.yml` for a three-project workspace spanning backend, frontend, and infrastructure.

## Example Feature Topology

See `examples/workspace-topology.yml` for a proposed topology that reuses existing components for an auth feature.

## Schemas

JSON schemas are provided for validation tooling:

| Schema | File |
|--------|------|
| Blueprint catalog | `schemas/blueprints.schema.json` |
| Workspace manifest | `schemas/workspace.schema.json` |
| Feature topology | `schemas/workspace-topology.schema.json` |

## Configuration

Copy the template and customize for your project:

```
cp workspace-materialize-config.template.yml .specify/extensions/workspace-materialize/workspace-materialize-config.yml
cp blueprints.template.yml .opencode/blueprints.yml
```

Edit `.opencode/blueprints.yml` to pin real tags or SHAs instead of placeholders.

## Integration with spec-kit-workspace-map

This extension produces artifacts that `spec-kit-workspace-map` consumes:

- `.opencode/workspace.yml` — the canonical workspace manifest
- `.opencode/blueprints.yml` — the blueprint catalog
- `specs/<feature>/workspace-topology.yml` — feature-specific topology

Run `/speckit.workspace-map.generate` after materialization to produce workspace maps from these artifacts.

## Hook Integration

If the installed Spec Kit version supports hooks:

- **after_plan**: Prompts to run `/speckit.workspace.propose` after `speckit.plan`
- **before_tasks**: Prompts to run `/speckit.workspace.check` before `speckit.tasks`

If hooks are not supported, run these commands manually at the indicated lifecycle points.

## License

MIT
