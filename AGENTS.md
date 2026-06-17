# AGENTS.md

## File Correlation Invariants

When changing any template or config file, update the corresponding schema and example in the same commit.

| Config / Template | Schema | Example |
|-------------------|--------|---------|
| `blueprints.template.yml`, `blueprints.yaml` | `schemas/blueprints.schema.json` | `examples/blueprints.yml` |
| *workspace manifest* | `schemas/workspace.schema.json` | `examples/workspace.yml` |
| *feature topology* | `schemas/workspace-topology.schema.json` | `examples/workspace-topology.yml` |

### Rules

1. **Add, remove, or rename a field in a template/config?** Update the corresponding JSON schema and example in the same commit.
2. **Add a validation constraint to a schema?** Ensure the corresponding example satisfies it.
3. `workspace-materialize-config.template.yml` and `extension.yml` have no schemas yet. If their structure grows, create a schema and example for each.
