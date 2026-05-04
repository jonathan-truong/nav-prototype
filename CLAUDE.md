<!-- bloom:start -->
# Working with bloom in this folder

You are an AI agent helping a user manage a Rose Rocket org via the **bloom** CLI. This file is auto-loaded into every Claude Code session in this directory and exists to surface the highest-frequency footguns before they bite.

## Orientation

- Recipe configs live in `bloom-recipes/<orgId>/`
- Long-form reference docs live in `bloom-recipes/documentation/` — search there first
- Bloom-shipped skills available via `/`:
  - **Diagnostic:** `/bloom-check-perms`, `/bloom-verify-board-writable`, `/bloom-recipe-preflight`, `/bloom-diagnose-create-button`, `/bloom-trace-automation`
  - **Generative:** `/bloom-generate-sample-records`, `/bloom-generate-board-view`, `/bloom-generate-chart-view`, `/bloom-generate-detail-view`
- Deterministic audits: `bloom recipe:doctor` (used internally by the diagnostic skills)
- Always run bloom from THIS directory (workspace root) — running from a subfolder treats it as a separate empty workspace
- Use non-interactive mode for all bloom commands: `bloom <command> --yes`

## Top-10 footguns

These are the most common mistakes when authoring Rose Rocket recipes. Each one has wasted hours on real customer projects.

1. **Read-only board columns** → `ObjectFieldPermission` entries are missing for a group. Every code-pushed field needs entries on EVERY `$custom_<role>.json` (`admin`, `super-admin`, `manager`, `operations`, `csr`, `sales`, `driver`, `partner`, `customer`, `external`, `guest`) — system fields like `$createdAt` included. The Builder UI auto-syncs these; code pushes don't. Run `/bloom-check-perms`.

2. **Create button doesn't fire** → the button's `json` payload is missing values for required fields. Every `isRequired: true` field on the target object must have a value in `json`. Empty `json: {}` fails silently from the user's perspective. Run `/bloom-diagnose-create-button`.

3. **Embedded-board Create button creates orphans** → platform doesn't thread `parentRecordId` to embedded boards. The button's `action` must extract the parent UUID from `window.location.href` and pass it as a ref.

4. **`multiCurrencyMoney` field type is broken** → renders read-only, returns 500 on write. Use `money` with `{amount, currencyCode}` instead.

5. **`dateTime` write shape** → `{dateTimeInLocation: "YYYY-MM-DDTHH:MM:SS", dateTimeInLocationEnd: null}`. **No `Z` suffix.** Bare ISO with Z returns 400.

6. **JSON detail-view buttons render but don't fire** — confirmed broken. Use a CustomWidget action button or move the action to the board level.

7. **Don't recreate platform objects** as `*__c` parallels — `customer`, `carrier`, `contact`, `location`, `address` are platform-shipped. Extend via `$custom_<name>` mixin instead. BRL-6001 chaos otherwise.

8. **Widgets cannot declare derived/LOOKUP fields** — widget save does a full PUT on every declared field; derived fields aren't writable. Move LOOKUPs to the detail view's `config-RecordDataTable`.

9. **`event:"create"` fires BEFORE initial values are applied** — conditions reading connection fields set by initialValueActions see null and abort. Use `event:"update"` with a `$hasChanged` condition instead.

10. **`bloom recipe:push` re-uploads the WHOLE folder** — deleted-on-cookbook configs whose local files still exist get silently re-created. Configs you wanted deleted come back. Run `/bloom-recipe-preflight` before every push.

## Always-on rules

- **Verify the active org** with `bloom status` before any `recipe:push`, `recipe:promote`, or `recipe:lock`
- **Never push to GTI-Canada** (UUID `632c4a4e-5843-4626-9535-6e5dfcd2d55f`) — read-only reference org
- For production orgs: dry-run first; if a generative skill needs to POST, require the user to retype the org name before any live write, and lower default sample-record counts
- After validation errors: read the error message literally — Rose Rocket echoes the offending field/value back to you. Fix the file and re-push.
- **Bloom config is increasingly standalone JSON.** Reach for the platform TypeScript recipe path only when JSON can't express the thing (e.g., `DocumentsEmbeddedBoard` widget today). Lead with JSON in every recommendation.

## Permissions chain (read this when adding a new object)

A new custom object needs FOUR permission types per group, or the corresponding UI surface vanishes:

| Permission | Gates | Missing → |
|---|---|---|
| `ObjectPermission` | create/delete + per-field perms | record CRUD blocked or columns hidden |
| `BoardPermission` | sidebar visibility | board doesn't appear |
| `DetailViewPermission` | detail page render — needs both `key` AND `detailView` | 404 "failed to find detail view for role" |
| `WidgetSequencePermission` | widget render — DUAL ENTRIES required (sequence-keyed AND object-keyed) | widgets invisible |

Add all four to every `$custom_<role>.json`. Then run `/bloom-check-perms`.

## Quick reference: where to find what

| When you need… | Look at |
|---|---|
| API write shapes (money, dateTime, refs, measurement) | `bloom-recipes/documentation/api-platformmodel.md` |
| Saved board views (filters, columns, charts) | `bloom-recipes/documentation/api-board-views.md` |
| Custom document generation | `bloom-recipes/documentation/document-generation.md` |
| Rocket Engine workflows | `bloom-recipes/documentation/workflows-rocket-engine.md` |
| Detail-view authoring (panels, embedded boards) | `bloom-recipes/documentation/view-builder-guide.md` |
| Permission cookbook (the 4-chain) | `bloom-recipes/documentation/groups.md` |
| "Why didn't my automation fire?" | `bloom-recipes/documentation/debugging-automations.md` |
| Mixins vs custom objects | `bloom-recipes/documentation/platform-objects-and-mixins.md` |
| Naming, file rules, CWD | `bloom-recipes/documentation/conventions.md` |

> The bloom-managed block (delimited by the HTML comment markers around this section) is regenerated by `bloom recipe:pull` and `bloom recipe:docs`. Add your own project notes outside the fence — they're preserved across regenerations.
<!-- bloom:end -->
