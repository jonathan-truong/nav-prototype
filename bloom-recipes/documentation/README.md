# Bloom Recipes Configuration Reference

Documentation for the JSON configuration files in the `bloom-recipes/` directory. Optimized for AI agent consumption.

## Repository Structure

```
bloom-recipes/
  <orgId>/                          # UUID identifying the organization
    org-meta.json                   # Org details (name, subdomain, etc.)
    manifest.json                   # Tracks all configs with hashes
    .raw/
      configs.json                  # All configs in a single flat array
    objects/                        # Data model definitions
      <objectName>.json
    boards/                         # Table/grid UI configs
      <boardName>.json
    detailViews/                    # Single-record detail screen configs
      <detailViewName>.json
    groups/                         # Permission group configs
      <groupName>.json
    widgets/                        # Interactive UI component configs
      <widgetName>.json
    widgetSequences/                # Multi-step widget flow configs
      <sequenceName>.json
```

## Documentation Index

| Document | Description |
|----------|-------------|
| [conventions.md](conventions.md) | Naming, filenames, ordering, configKey values, mixin pattern, audit fields, CWD rule, text-cache → LOOKUP refactor |
| [objects.md](objects.md) | Object definitions: fields, field types, events, methods, **API write shapes**, derivation types (formula + LOOKUP), connection rules |
| [automations.md](automations.md) | Automation rules: triggers, actions, conditions, **closed actionKey union**, sendNotification shape, event timing, recipe lifecycle |
| [boards.md](boards.md) | Board configs: columns, filters, buttons, security filters, **kind vs type**, Create-button required-fields footgun |
| [detail-views.md](detail-views.md) | Detail view configs: sections, detail fields, **platform widgets catalog**, buttons-broken warning |
| [groups.md](groups.md) | Permission groups: **the 4-permission visibility chain**, board/object/field/detailView/widget permissions, dual-entry pattern |
| [widgets.md](widgets.md) | Widget configs: fields, actions, conditions, **derived-fields-forbidden rule**, embedded-board parent ref pattern |
| [widget-sequences.md](widget-sequences.md) | Widget sequence configs: ordered widget references |
| [manifest.md](manifest.md) | Manifest tracking and .raw/configs.json format |
| [api-platformmodel.md](api-platformmodel.md) | REST API for record CRUD: endpoints, write shapes, auth token location |
| [api-board-views.md](api-board-views.md) | Saved board views API: filter operators, **SelectField in:[] gotcha**, charts |
| [debugging-automations.md](debugging-automations.md) | /events API for "why didn't my automation fire?" |
| [workflows-rocket-engine.md](workflows-rocket-engine.md) | Rocket Engine workflows: trigger vs node syntax, $merge gotcha, cross-object create |
| [document-generation.md](document-generation.md) | Custom document types, templates, generation workflow |
| [view-builder-guide.md](view-builder-guide.md) | Detail-view authoring: panelGroup/panel/widget, layout recipes |
| [platform-objects-and-mixins.md](platform-objects-and-mixins.md) | Don't recreate platform objects; extend via mixins |

## Workspace-Root Files (Auto-Generated)

In addition to this `bloom-recipes/` tree, bloom generates two other surfaces in the workspace root that you should know about:

- **`<workspace>/CLAUDE.md`** — auto-loaded into every Claude Code session in this directory. Contains the top-10 footguns and an at-a-glance pointer to the docs and skills below. Always overwritten on `recipe:pull`/`recipe:docs`; do not hand-edit.
- **`<workspace>/.claude/skills/bloom-*.md`** — invocable from Claude Code:
  - **Diagnostic:** `/bloom-check-perms`, `/bloom-verify-board-writable`, `/bloom-recipe-preflight`, `/bloom-diagnose-create-button`, `/bloom-trace-automation`
  - **Generative:** `/bloom-generate-sample-records` (dry-run; prod-cautious), `/bloom-generate-board-view`, `/bloom-generate-chart-view` (bar / metric / donut), `/bloom-generate-detail-view`
- **`bloom recipe:doctor`** — deterministic audit command (perms / board-writable / drift). Skills shell out to this.

User-authored skills in `.claude/skills/` are never modified or deleted by bloom — only `bloom-*` files are managed.

## Quick Reference: What to Edit

| Task | Config Type | File Location |
|------|-------------|---------------|
| Add a new data entity | `objects/` | Create `<name>.json` with `configKey: "object"` |
| Add a field to an entity | `objects/` | Add entry to `fields[]` in the object file |
| Create a list/table view | `boards/` | Create `<name>-board.json` with `configKey: "board"` |
| Add another board for the same object | `boards/` | Create a second board file with a unique `key` but the same `objectKey` |
| Create a record detail screen | `detailViews/` | Create `<name>-detail-view.json` with `configKey: "detailView"` |
| Set up role permissions | `groups/` | Edit `$custom_<role>.json` to add board/object/field permissions |
| Add an interactive component | `widgets/` | Create `<name>.json` with `configKey: "widget"` |
| Chain widgets into a flow | `widgetSequences/` | Create `<name>.json` with `configKey: "widgetSequence"` |

## Safety: Org Verification

> **WARNING:** Never switch the active org without explicit user confirmation.

Before running any `recipe:push`, `recipe:promote`, or destructive recipe command:
1. Run `bloom status` to verify the currently active org
2. Confirm the org name and subdomain match the intended target
3. If the wrong org is active, **ask the user** before switching — do not switch silently

## Agent Usage: Non-Interactive Mode

Agents should **always** use non-interactive (headless) mode. Pass commands as arguments to `bloom` instead of launching the interactive REPL.

```
bloom <command> [args] [--yes]
```

The `--yes` (or `-y`) flag auto-approves confirmation prompts. Without it, commands that require confirmation will abort in headless mode.

### Command Reference

| Command | Example | Description |
|---------|---------|-------------|
| `status` | `bloom status` | Check auth and active org |
| `login` | `bloom login --yes` | Authenticate via browser |
| `org:switch` | `bloom org:switch <uuid>` | Switch org by UUID (preferred) |
| `org:switch` | `bloom org:switch acme` | Switch org by name/substring |
| `recipe:pull` | `bloom recipe:pull --yes` | Pull configs from active org (refuses to overwrite local edits) |
| `recipe:pull` | `bloom recipe:pull --yes --force` | Pull and overwrite any local edits |
| `recipe:push` | `bloom recipe:push --yes` | Push local changes to org (publishes immediately) |
| `recipe:push` | `bloom recipe:push --dry-run` | Preview the push without sending |
| `recipe:push` | `bloom recipe:push --yes --force` | Push regardless of remote drift or empty changeset |
| `recipe:promote` | `bloom recipe:promote <sourceOrgId> --dry-run` | Promote configs from another org (preview) |
| `recipe:promote` | `bloom recipe:promote <sourceOrgId> --yes` | Promote configs from another org |
| `recipe:lock` | `bloom recipe:lock --yes` | Lock the org so direct `recipe:push` is rejected |
| `recipe:unlock` | `bloom recipe:unlock --yes` | Unlock the org |
| `recipe:status` | `bloom recipe:status` | Show recipe sync state |
| `recipe:diff` | `bloom recipe:diff` | Show detailed config diffs |
| `recipe:docs` | `bloom recipe:docs` | Regenerate documentation files |

Aliases work in headless mode (e.g., `bloom rp --yes` for `recipe:pull`). Run `bloom help` for the full command list.

`org:switch` accepts an org UUID (recommended for scripted use) or a name/substring as a positional argument. Without args it launches an interactive picker which is not supported headlessly.

### Recipe Lifecycle: Pull → Edit → Push

1. **`bloom recipe:pull --yes`** — Downloads all configs from the active org as JSON files into `bloom-recipes/<orgId>/`. Always start here to establish a baseline. Refuses if local has uncommitted edits — pass `--force` to overwrite.
2. **Edit JSON files** — Modify, add, or remove config files in the local `bloom-recipes/<orgId>/` directories (objects, boards, groups, detailViews, widgets, etc.)
3. **`bloom recipe:push --dry-run`** *(optional)* — Print the diff that would be sent without POSTing. Useful before a real push.
4. **`bloom recipe:push --yes`** — Uploads the local snapshot to the org and publishes it. Refuses if the server has changed since your last pull (run `recipe:pull` first to reconcile, or pass `--force` to overwrite). Refuses on locked orgs — use `recipe:promote` from a source org instead.

Agents should always follow this order. Skipping `pull` means there is no baseline to diff against.

## Handling Validation Errors

Recipe pushes may fail on the first attempt due to assumption mismatches between the local config and the server's expectations.

**Expected behavior for agents:**
1. Run `bloom recipe:push --yes`
2. If the push fails with validation errors, **read the error output carefully**
3. Correct the offending config file(s) based on the error messages
4. Re-push: `bloom recipe:push --yes`
5. Repeat until the push succeeds or the error is clearly not fixable

**After resolving all errors:**
- Generate a report listing every validation error encountered and how it was resolved
- Encourage the user to email the report to **ryan.m@roserocket.com** so the documentation can be improved for future agents
