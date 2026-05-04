# Bloom Recipes

This directory contains Rose Rocket platform configuration files managed by the [Bloom CLI](https://github.com/aspect-build/bloom-CLI). Each subdirectory represents an organization's recipe configs pulled from the Rose Rocket platform.

## Folder Structure

```
bloom-recipes/
  README.md                           ← you are here
  documentation/                      ← detailed reference guides
  <orgId>/                            ← organization UUID
    manifest.json                     ← auto-managed tracking file (do not edit)
    .raw/
      configs.json                    ← baseline snapshot (do not edit)
    objects/                          ← data model definitions
    boards/                           ← list/table view configs
    detailViews/                      ← single-record detail screen configs
    groups/                           ← permission group configs
    widgets/                          ← interactive UI component configs
    widgetSequences/                  ← multi-step widget flow configs
```

## Config Types

| Type | Directory | Purpose |
|------|-----------|---------|
| Object | `objects/` | Define data entities, their fields, events, methods, and automations (via jsActionable) |
| Board | `boards/` | Configure list/table views with columns, filters, and action buttons |
| Detail View | `detailViews/` | Configure single-record screens with sections and fields |
| Group | `groups/` | Define permission roles controlling access to boards, objects, fields, and widgets |
| Widget | `widgets/` | Build interactive UI components with fields, actions, and conditions |
| Widget Sequence | `widgetSequences/` | Chain multiple widgets into ordered multi-step flows |

## Key Rules

- **Do not edit** `manifest.json` or anything in `.raw/` — these are auto-managed by Bloom CLI
- Custom config keys use a `__c` suffix (e.g. `bike__c`, `chairBoard__c`)
- Filenames are kebab-case versions of the config key (e.g. `custom-order.json` for `customOrder__c`)
- Each JSON file has a `configKey` field identifying its type and a `key` field as its unique identifier

## Detailed Documentation

See the `documentation/` folder for comprehensive reference on each config type, naming conventions, field types, and examples.

## Deploying Changes

After editing config files, push changes back to the platform:

```bash
bloom recipe:push
```

Use `bloom recipe:status` to see what has changed and `bloom recipe:diff` to review differences before pushing.
