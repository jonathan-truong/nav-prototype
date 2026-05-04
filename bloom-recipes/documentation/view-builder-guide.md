# Detail View Builder Guide

This is the long-form how-to for authoring **custom detail views** via the JSON-config recipe path. The companion doc `detail-views.md` covers the basic schema and the auto-generated Overview tab; this guide covers the full **`views`** array — multi-tab layouts, panel grids, embedded child boards, and the platform-provided widgets that snap into them.

If you have never authored a detail view before, read `detail-views.md` first. Then come back here when you need to build a tabbed dashboard, embed a child board, or compose Overview + sidebar widgets + a related-records grid into a single screen.

---

## The schema in plain English

A detail view is a record-page layout. Inside its `views` array you can declare additional **tabs** (alongside the auto-generated Overview), and each tab is composed of a tree:

```
detailView
└── views[]
    └── view (one tab)
        └── panelGroups[]
            └── panelGroup (orientation: horizontal | vertical)
                └── panels[]
                    └── panel (layout: tabs | stack)
                        └── widgets[]
                            └── widgetDisplay
                                ├── widgetKey: "config-..." or "<custom>__c"
                                └── runtimeConfig: { ...widget-specific config... }
```

Each level has a `configKey` discriminator (`view`, `panelGroup`, `panel`, `widgetDisplay`) and a `key`. **Every key in this tree must end in `__c`.** The platform validator enforces `.*__c$` on `view.key`, `panelGroup.key`, `panel.key`, and `widgetDisplay.key`. Recipe push will accept keys without the suffix; **publish will fail** with `String does not match the pattern of ".*__c$"`. Add `__c` to every single key, including throwaway prototype keys.

### Required fields surfaced by validation

Three fields are commonly missed and will fail `recipe:validate` with `BRLD-1002 Missing required field`:

- `view.label` (string) — every entry in `views[]` must have a non-empty `label` (used as the tab label in the detail-view UI). Carrier106 patterns: `"Profile"`, `"Documents"`, `"Activity"`. An empty string `""` is currently accepted by validation but produces a missing tab label in the UI; pick a real word.
- `panel.widgets` (array) — the array must be present even if empty. The field is `widgets`, NOT `widgetDisplay` (that is the `configKey` of the *items* inside this array, not the array name itself).
- `panel.layout` — must be one of `"tabs"` or `"stack"` (see "Panel layout values" below). `"single"` is intuitive but rejected.

A minimal one-tab view looks like:

```json
{
  "configKey": "view",
  "key": "documentsView__c",
  "label": "Documents",
  "panelGroupsDirection": "horizontal",
  "panelGroups": [{
    "configKey": "panelGroup",
    "key": "documentsGroup__c",
    "panelsDirection": "horizontal",
    "isCollapsible": true,
    "panels": [{
      "configKey": "panel",
      "key": "documentsPanel__c",
      "layout": "tabs",
      "isCollapsible": true,
      "widgets": [{
        "configKey": "widgetDisplay",
        "key": "documentsWidget__c",
        "label": "Documents",
        "widgetKey": "config-ConfigurableEmbeddedBoard",
        "runtimeConfig": {
          "boardKey": "documentsEmbeddedBoard__c",
          "connectedPageFieldPath": "documents__c",
          "connectedBoardFieldPath": "$id"
        }
      }]
    }]
  }]
}
```

That JSON renders as a "Documents" tab on the parent record's detail page, showing a child board filtered to the parent's related documents.

---

## Platform-provided widgets catalog

These widgets are pre-built by the platform and referenced from a view by `widgetKey`. They are the **only** widgets you can use in a JSON-only recipe path; anything richer (custom upload buttons, maps, canvases with bound actions) needs a TypeScript recipe.

### `config-ConfigurableEmbeddedBoard`

Embeds a child board into a tab, optionally filtered to records connected to the parent record.

**Accepted `runtimeConfig` keys (Zod-locked — anything else is silently dropped):**

| Key | Required | Purpose |
|---|---|---|
| `boardKey` | yes | Key of the embedded board to render. |
| `label` | no | Override the tab/header label. |
| `connectedPageFieldPath` | no | Field on the **parent** that lists connected child records (e.g., `legs__c`). Drives the filter. |
| `connectedBoardFieldPath` | no | Field on the **child** record to match against — almost always `$id`. |

Behavior is hardcoded:
- `useSimpleHeader: true`
- `canDeleteItems: false`
- `hideViewCreation: true`

**Things you cannot configure** (we tried; the schema swallows them silently): `hideSearchBar`, `hideFilter`, `hideButtons`, `hideBoardViews`, `hideFooter`, `hideColumnCustomization`, `hideGroupBy`, `canExportCSV`, `columnKeys`, `appliedFilters`, `filterKeys`, `stickyColumnKeys`, `height`, `emptyStateConfig`. If you need any of these, you need a TypeScript-side widget built on `GenericEmbeddedBoard` — out of scope for this guide; see your TS recipe path.

### `config-RecordDataTable`

Renders the **standard Overview field table** — the same key/value layout the auto-generated Overview tab uses. No `runtimeConfig`; reads `data.fields` from the parent detailView config.

Use this when you want the Overview to live inside a custom tab (e.g., next to a sidebar, or above an embedded board) rather than as the default first tab. This is the canonical "details" panel — pair it with the detail view's full `data.fields` list to get the full record summary.

### `config-WidgetStack`

Renders the standard widget sidebar — the stack of `CustomWidget` instances normally pinned to the right of a record page. No `runtimeConfig`; reads the object's `widgetSequence` config.

Use this when you want the sidebar to appear inside a specific tab. It cannot host an embedded board (widget sequences silently filter out `config-ConfigurableEmbeddedBoard` references at render time — push and publish accept them, the UI ignores them).

### `config-RecordCanvas`

Renders the freeform record canvas (rich text / notes). No `runtimeConfig`. Drop it into a panel for a "Notes" tab.

### Settings-only widgets (don't use in record detail views)

These widgets exist in the platform's widget library but are scoped to the global `config` settings object. They will not render meaningful content inside a custom record detail view; ignore them unless you are authoring an admin/settings page:

`config-EmbeddedAccessorialBoard`, `config-EmbeddedAssetTypesSettingBoard`, `config-EmbeddedComplianceSettingBoard`, `config-EmbeddedLocationsBoard`, `config-EmbeddedTaxRateBoard`, `config-EmbeddedZoneSettingBoard`, `config-WebhookTriggerSettingsPanel`, `config-AiAgentSettings`, `config-ArchivedRecords`, `config-ChartOfAccounts`, `config-EmailSettings`, `config-CustomDocumentType`, `config-DocumentsBranding`, `config-DocumentsGeneral`.

### Custom widgets (referenced by key)

Any of your own `CustomWidget` instances (`<key>__c`, `component: "CustomWidget"`) can be referenced by `widgetKey` from a panel. The widget's declared `fields` array renders as a form. See `widgets.md` for widget authoring; see the **rule reminder** further down for the most common pitfall.

---

## panelGroup orientations — when to pick which

`panelGroup.panelsDirection` controls how the panels **inside that group** stack:

- **`horizontal`** — panels render side by side. Use for "main content + sidebar" layouts, or for two related child boards displayed in parallel (e.g., Legs next to Stops).
- **`vertical`** — panels render stacked top to bottom. Use for "summary above, detail below" or when you want a long single-column reading flow.

`view.panelGroupsDirection` controls how multiple **panelGroups** stack inside the view. It accepts the same `horizontal`/`vertical` values.

**Nesting orientations is how you build grids.** A horizontal outer with vertical inner groups makes a row of columns. A vertical outer with horizontal inner groups makes a stack of rows. Reference matrix:

| Layout | view.panelGroupsDirection | # groups | each panelGroup.panelsDirection | panels per group |
|---|---|---|---|---|
| Single panel | horizontal | 1 | horizontal | 1 |
| Side-by-side (2 panels) | horizontal | 1 | horizontal | 2 |
| Stacked (2 panels) | horizontal | 1 | vertical | 2 |
| 2×2 grid | vertical | 2 | horizontal | 2 each |
| 3-column row | horizontal | 1 | horizontal | 3 |
| 3-row column | horizontal | 1 | vertical | 3 |
| 3×2 grid | vertical | 2 | horizontal | 3 each |

`isCollapsible: true` is supported on both `panelGroup` and `panel`; users can collapse the section in the UI.

---

## Panel layout values

`panel.layout` accepts exactly two values per server-side validation (BRLD-1001 if you send anything else, including `"single"`):

- **`tabs`** — multiple widgets render as a tab strip inside the panel. Each widget's `label` becomes the tab name. With one widget, the tab strip is hidden — `tabs` is the safe default for any panel.
- **`stack`** — widgets render stacked vertically with no tab chrome.

> **Common pitfall:** `"single"` is **not** a valid value (it's intuitive but rejected). Carrier106's reference detail views all use `"tabs"`, even for single-widget panels. Use `tabs` unless you specifically want stacked rendering.

---

## Embedded board connection filtering

The killer feature of `config-ConfigurableEmbeddedBoard` is **parent-scoped filtering**: show only the child records connected to *this* parent record.

```json
{
  "widgetKey": "config-ConfigurableEmbeddedBoard",
  "runtimeConfig": {
    "boardKey": "legsEmbeddedBoard__c",
    "connectedPageFieldPath": "legs__c",
    "connectedBoardFieldPath": "$id"
  }
}
```

How this resolves at render time:
- The platform reads the parent's `legs__c` field — typically an inverse-array ref pointing at child leg records.
- It filters the embedded `legsEmbeddedBoard__c` to only those rows whose `$id` (literal record id) appears in the parent's `legs__c` array.

`connectedBoardFieldPath` is almost always `$id` (filter by record id). Use a different field only if you have a non-id link key on the child.

**Without the connection keys** — omit both `connectedPageFieldPath` and `connectedBoardFieldPath` — the embedded board shows **all** records on that board (root view). Useful for settings pages; rarely what you want for a record-detail tab.

**Pattern recommendation.** Build a dedicated embedded-board variant per child object (e.g., `legsEmbeddedBoard__c` separate from `legBoard__c`) with: trimmed columns (5–10 most relevant), minimal filters, no chart-view tabs, no extra buttons. The main board is too noisy inside a record-detail context.

---

## Layout recipe gallery

JSON snippets for the three most common layouts. All keys end in `__c`; substitute your own object/board keys.

### Side-by-side

One panelGroup, horizontal, two panels. Useful for "Legs on the left, Accessorials on the right".

```json
{
  "configKey": "view",
  "key": "legsAndAccessorialsView__c",
  "label": "Legs + Accessorials",
  "panelGroupsDirection": "horizontal",
  "panelGroups": [{
    "configKey": "panelGroup",
    "key": "sideBySideGroup__c",
    "panelsDirection": "horizontal",
    "isCollapsible": true,
    "panels": [
      {
        "configKey": "panel",
        "key": "legsPanel__c",
        "layout": "tabs",
        "isCollapsible": true,
        "widgets": [{
          "configKey": "widgetDisplay",
          "key": "legsWidget__c",
          "label": "Legs",
          "widgetKey": "config-ConfigurableEmbeddedBoard",
          "runtimeConfig": {
            "boardKey": "legsEmbeddedBoard__c",
            "connectedPageFieldPath": "legs__c",
            "connectedBoardFieldPath": "$id"
          }
        }]
      },
      {
        "configKey": "panel",
        "key": "accessorialsPanel__c",
        "layout": "tabs",
        "isCollapsible": true,
        "widgets": [{
          "configKey": "widgetDisplay",
          "key": "accessorialsWidget__c",
          "label": "Accessorials",
          "widgetKey": "config-ConfigurableEmbeddedBoard",
          "runtimeConfig": {
            "boardKey": "accessorialsEmbeddedBoard__c",
            "connectedPageFieldPath": "accessorials__c",
            "connectedBoardFieldPath": "$id"
          }
        }]
      }
    ]
  }]
}
```

### 2×2 grid

One outer view (vertical panelGroupsDirection), two inner panelGroups, each horizontal with two panels. Quadrants: top-left, top-right, bottom-left, bottom-right.

```json
{
  "configKey": "view",
  "key": "loadDashboardView__c",
  "label": "Dashboard",
  "panelGroupsDirection": "vertical",
  "panelGroups": [
    {
      "configKey": "panelGroup",
      "key": "topRow__c",
      "panelsDirection": "horizontal",
      "isCollapsible": true,
      "panels": [
        {
          "configKey": "panel", "key": "legsP__c", "layout": "tabs", "isCollapsible": true,
          "widgets": [{
            "configKey": "widgetDisplay", "key": "legsW__c", "label": "Legs",
            "widgetKey": "config-ConfigurableEmbeddedBoard",
            "runtimeConfig": { "boardKey": "legsEmbeddedBoard__c", "connectedPageFieldPath": "legs__c", "connectedBoardFieldPath": "$id" }
          }]
        },
        {
          "configKey": "panel", "key": "stopsP__c", "layout": "tabs", "isCollapsible": true,
          "widgets": [{
            "configKey": "widgetDisplay", "key": "stopsW__c", "label": "Stops",
            "widgetKey": "config-ConfigurableEmbeddedBoard",
            "runtimeConfig": { "boardKey": "stopsEmbeddedBoard__c", "connectedPageFieldPath": "stops__c", "connectedBoardFieldPath": "$id" }
          }]
        }
      ]
    },
    {
      "configKey": "panelGroup",
      "key": "bottomRow__c",
      "panelsDirection": "horizontal",
      "isCollapsible": true,
      "panels": [
        {
          "configKey": "panel", "key": "accP__c", "layout": "tabs", "isCollapsible": true,
          "widgets": [{
            "configKey": "widgetDisplay", "key": "accW__c", "label": "Accessorials",
            "widgetKey": "config-ConfigurableEmbeddedBoard",
            "runtimeConfig": { "boardKey": "accessorialsEmbeddedBoard__c", "connectedPageFieldPath": "accessorials__c", "connectedBoardFieldPath": "$id" }
          }]
        },
        {
          "configKey": "panel", "key": "invP__c", "layout": "tabs", "isCollapsible": true,
          "widgets": [{
            "configKey": "widgetDisplay", "key": "invW__c", "label": "Invoices",
            "widgetKey": "config-ConfigurableEmbeddedBoard",
            "runtimeConfig": { "boardKey": "invoicesEmbeddedBoard__c", "connectedPageFieldPath": "invoices__c", "connectedBoardFieldPath": "$id" }
          }]
        }
      ]
    }
  ]
}
```

### Stacked (Overview + Widgets + Embedded Board, all in one tab)

The "killer combo" — recreate the default record layout in a single tab and add a related-records board below it.

```json
{
  "configKey": "view",
  "key": "fullView__c",
  "label": "Full View",
  "panelGroupsDirection": "vertical",
  "panelGroups": [
    {
      "configKey": "panelGroup", "key": "overviewGroup__c", "panelsDirection": "vertical", "isCollapsible": true,
      "panels": [{
        "configKey": "panel", "key": "overviewPanel__c", "layout": "tabs", "isCollapsible": true,
        "widgets": [{
          "configKey": "widgetDisplay", "key": "overviewWidget__c", "label": "Record",
          "widgetKey": "config-RecordDataTable"
        }]
      }]
    },
    {
      "configKey": "panelGroup", "key": "sidebarGroup__c", "panelsDirection": "vertical", "isCollapsible": true,
      "panels": [{
        "configKey": "panel", "key": "sidebarPanel__c", "layout": "tabs", "isCollapsible": true,
        "widgets": [{
          "configKey": "widgetDisplay", "key": "sidebarWidget__c", "label": "Widgets",
          "widgetKey": "config-WidgetStack"
        }]
      }]
    },
    {
      "configKey": "panelGroup", "key": "legsGroup__c", "panelsDirection": "vertical", "isCollapsible": true,
      "panels": [{
        "configKey": "panel", "key": "legsPanel__c", "layout": "tabs", "isCollapsible": true,
        "widgets": [{
          "configKey": "widgetDisplay", "key": "legsW__c", "label": "Legs",
          "widgetKey": "config-ConfigurableEmbeddedBoard",
          "runtimeConfig": { "boardKey": "legsEmbeddedBoard__c", "connectedPageFieldPath": "legs__c", "connectedBoardFieldPath": "$id" }
        }]
      }]
    }
  ]
}
```

---

## CRITICAL — JSON detail-view buttons render but DON'T fire

**This is the single biggest source of wasted hours for new view-builder authors. Read it twice.**

If you author a button at the **detailView level** via JSON (in `detailView.buttons[]`), it **renders in the UI but clicks do nothing.** Confirmed broken across two schemas (board-button-style `action: "async ({context}) => {...}"` and TS-style `onClick: "async (record, ctx) => {...}"`). Recipe push accepts both, publish succeeds, the button paints — every click silently no-ops.

Hypothesis: detail-view buttons are TypeScript-compiled in the platform's typed context. Serialized string handlers shipped via JSON are not evaluated by the detail-view button runtime. (Board-level buttons evidently *do* support serialized strings — but the parser differs.)

**Workarounds, in order of preference:**

1. **Move the action to a CustomWidget** — author a `CustomWidget` with an `actions: []` array containing the button, and place that widget in a panel via the View Builder. Widget action buttons fire correctly.
2. **Move the action to the board level** — board-level "Create X" / bulk-action buttons work fine in JSON. If the action is contextual to the board (not the single record), this is the cleanest answer.
3. **Author the button in a TypeScript recipe** — outside the JSON path entirely. Required if neither workaround above fits.

Until the platform-side fix lands, assume `detailView.buttons[]` is cosmetic-only when authored from JSON. Don't ship buttons there.

---

## Embedded-board Create button — parent-ref pattern

`config-ConfigurableEmbeddedBoard` does **not** thread a `parentRecordId` into its embedded board's button context. Inside a button `action` running in an embedded-board context, `context.recordId`, `context.parentRecordId`, `context.connectedRecordId`, and `context.parentRecord` are all `undefined`. You only get `context.boardId`, `context.createRecord`, `context.refetch`, and `context.navigateToRecord`.

To create a child record linked to the current parent, **extract the parent UUID from `window.location.href`** with a regex on the route segment. Rose Rocket SPA URLs use hash routing, so the route lives in `location.hash` — match against `href` to cover both:

```js
async ({ context }) => {
  const url = (typeof window !== 'undefined') ? (window.location.href || '') : '';
  const m = url.match(/load__c\/([0-9a-f-]{36})/i);
  const parentId = m && m[1];
  const baseJson = { name: 'New Leg', status__c: 'planned' };
  const json = parentId
    ? Object.assign({}, baseJson, { load__c: { id: parentId } })
    : baseJson;
  await context.createRecord({
    boardId: context.boardId,
    objectKey: 'leg__c',
    json,
  });
  await context.refetch();
}
```

Notes:
- Match the parent's **objectKey** in the regex (`load__c` here). Different parent objects need different patterns.
- The route uses 36-char hex-with-dashes UUIDs; the pattern above is loose enough to catch them.
- **Do not call `navigateToRecord` after the create** — for inline-row creation, `refetch()` is enough and avoids a 404 if the child object's `DetailViewPermission` for the current role isn't set up.
- Bloom's search endpoint can briefly lag indexing the new row. `refetch()` works most of the time; a manual page refresh always works.

---

## Widget rule reminder — derived/LOOKUP fields can't go in `widget.fields[]`

CustomWidgets PUT every field declared in their `fields[]` array on save. **Derived fields (LOOKUP, formula, anything with `isDerived: true`) are not writable** — including one in a widget's field list causes the entire save to return `400 Bad Request`, even if the user only edited a different field.

**Symptom:** widget save fails 400 with no obvious error tying it to a specific field. Check the field list for any LOOKUPs.

**Rule of thumb:**
- Widgets = **editable** fields only. Refs, scalars (text, number, select, dateTime, money, toggle, etc.) are fine.
- Derived/LOOKUP fields go in the **detail view's `config-RecordDataTable`** (the Overview field list), not in a widget. The detail view renders them as read-only and does not PUT them on save.
- If you want contextual display (e.g., show the picked facility's city next to the facility ref picker), accept the trade-off: place the LOOKUP field in the adjacent RecordDataTable section, not inside the widget.

There is no documented per-field `isReadOnly` / `displayOnly` flag that overrides this; if one exists, it's undocumented.

See `widgets.md` for the full widget authoring reference.

---

## Recovery — detail view broke after a bad widget config

**Symptom:** `bloom recipe:publish` returns 500, and the affected records start returning 500 from their detail-view fetch too. Usually triggered by a custom widget whose serialized JS action references a typed enum that didn't resolve (`WidgetActionType.Read`, `ButtonType.Primary`, etc.) — sometimes by a malformed view tree (missing `__c`, invalid direction).

**Fix — delete the bad widget directly from cookbook, republish, then resync:**

```bash
TOKEN=$(jq -r '.auth.accessToken' ~/Library/Preferences/bloom-cli-nodejs/config.json)
ORG="<orgId>"
HOST="https://<sub>.roserocket.com/api/v2/platformModel"

# Identify the bad widget and DELETE it from cookbook
curl -s -X DELETE -H "Authorization: Bearer $TOKEN" \
  "$HOST/cookbook/${ORG}ObjectBuilderMixin/configs/widget/<badWidgetKey>"

# Republish — should now succeed
curl -s -X PATCH -H "Authorization: Bearer $TOKEN" \
  "$HOST/cookbook/${ORG}ObjectBuilderMixin/publish"

# Resync local recipe files
bloom recipe:pull --yes
```

Records start working again as soon as the bad config is gone from cookbook and publish succeeds. If `recipe:doctor` is available in your CLI build, run it after the resync to catch any drift between local and remote.

The same recovery applies if a malformed view tree blocks publish: delete the offending detailView's `views` entry (or the whole detailView config), publish, fix the JSON locally, then push again.

---

## Cross-references

- **`detail-views.md`** — basic schema, the auto-generated Overview tab, `data.fields`, `widgetSequence`, and the legacy `panels` array.
- **`widgets.md`** — CustomWidget authoring: `fields`, `actions`, `buttons`, the rule against derived fields.
- **`groups.md`** — `DetailViewPermission` shape; every detail view must have a permission entry per group, or roles see "failed to find detail view for role" 404s.

For TypeScript-side widgets (custom embedded boards with rich actions, `Map`, `Canvas`, `DocumentsEmbeddedBoard` with upload buttons), see your platform-components TS recipe path — those cannot be authored from JSON.
