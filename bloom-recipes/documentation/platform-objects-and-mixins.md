# Platform Objects vs Custom Objects

**Rule of the road:** if Rose Rocket already ships an object, you extend it via
mixin. You do **not** create a parallel `*__c` version. This is the single
most common — and most expensive — mistake on new recipe projects.

A team will see no `customer` object in their freshly-scaffolded recipe folder,
assume it doesn't exist, create `customer__c.json`, and then spend a day
chasing `BRL-6001 Unknown object key` errors when boards, automations, and
cross-references can't decide which "customer" to talk to. There are now two
sources of truth and the platform GUI is wired to the wrong one.

## Platform-shipped objects (do NOT recreate)

Confirmed platform-provided objects you must extend, never duplicate:

- `customer`
- `carrier`
- `contact`
- `location`
- `address`

This list is **not exhaustive**. Broker / shipper orgs ship additional objects
(`load`, `leg`, `stop`, `order`, `shipment`, etc.). The canonical way to
discover what your specific org ships is described in
[How to discover platform objects](#how-to-discover-platform-objects) below.

## Why duplicating breaks everything

Platform-shipped objects are not just data containers. They are deeply wired
into:

- **Boards & detail views** — the default GUI binds to the platform key
- **Automations & events** — built-in triggers fire against the platform key
- **Mixins composed at publish** — base recipes, group permissions, and
  upstream configs all reference the platform key
- **Cross-record references** — `{ "ref": true, "configKey": "object" }` fields
  pointing at `customer` resolve; pointing at `customer__c` they raise
  `BRL-6001 Unknown object key`

Recreate `customer` as `customer__c` and you get two parallel object trees,
duplicate-key validation chaos, and a GUI that shows the platform's customers
while your custom records sit invisible.

## The right pattern: extend via mixin

To add fields, events, or methods to a platform object, drop a mixin file at:

```
bloom-recipes/<orgId>/objects/$custom_<targetKey>.json
```

Example — extending `customer`:

```json
{
  "configKey": "object",
  "key": "$custom_customer",
  "isMixin": true,
  "targetKey": "customer",
  "fields": [
    {
      "key": "loyaltyTier__c",
      "type": { "key": "text", "ref": true, "configKey": "type" },
      "label": "Loyalty Tier",
      "isCustom": true,
      "configKey": "field"
    }
  ]
}
```

The platform's RecipeLoader composes `$custom_customer` onto the base
`customer` at runtime. To you it looks like one object; to the platform the
base remains canonical and your additions ride on top.

## What belongs in a mixin

**Yes:**

- Custom fields (`isCustom: true`, suffix with `__c`)
- Custom events
- Custom methods / actions

**No:**

- A redefinition of the object's identity (`level`, `key`)
- Renames of base fields
- Removal of base fields

A mixin **adds**. It does not replace.

## When you DO want a new custom object

If the entity does not exist on the platform — `product`, `equipment`,
`lead`, `contract`, `driver`, `tractor`, `trailer`, `freightItem` — that's
a legitimate custom object. Create it as:

```
bloom-recipes/<orgId>/objects/<entity>__c.json
```

with `level: "primary"` and `isCustom: true`. The `__c` suffix is the
contract that tells the platform "this is yours, not mine."

## Wrong vs right — quick reference

| Goal | Wrong | Right |
|---|---|---|
| Add a field to `customer` | `customer__c.json` | `$custom_customer.json` (mixin) |
| Add a field to `carrier` | `carrier__c.json` | `$custom_carrier.json` (mixin) |
| Override a platform field's behavior | new `*__c` object | mixin on the base |
| Add a "lead" entity (no platform base) | mixin on `customer` | `lead__c.json`, `level: "primary"` |
| Add a "driver" entity (no platform base on carrier orgs) | mixin on `contact` | `driver__c.json`, `level: "primary"` |
| Add a "freightItem" entity | mixin on `load` | `freightItem__c.json`, primary |

## How to discover platform objects

Don't guess. Pull a baseline org and inspect the objects folder:

```bash
bloom recipe:pull
ls bloom-recipes/<orgId>/objects/
```

Rule:

- Any file **without** `__c` in its key → platform-shipped, extend via mixin
- Any file **with** `__c` → custom, owned by the recipe

For an even safer probe (e.g. a partially-pulled recipe), push a dummy custom
object with a single ref field at the candidate key. If push validates, the
platform has the object; if it errors with `BRL-6001`, it doesn't.

## Mixin gotchas — quick reference

**Stub mixins unblock base-recipe drift.** When a push fails with
`SDLC-4001 / BRL "Unknown field X in object Y"` because the upstream base
recipe drifted, an empty-ish `$custom_<Y>.json` mixin that defines `X` as a
text stub will satisfy composition. The field is never read; it just needs to
exist for the composer. See `automations.md` (drift section) and the
the stub-mixin drift unblocker in automations.md memo for the full pattern.

**Mixins compose at runtime, not at file-write.** You will not see the merged
object in your local files. Composition happens server-side at publish. Your
local repo holds the base and the `$custom_*` deltas separately, and that's
correct.

**One mixin per target per recipe.** Don't ship two
`$custom_customer.json`-shaped files; consolidate fields into one mixin file
keyed by `targetKey: "customer"`.

**Custom board mixins need `buttons: []`.** A common composition error —
`source.buttons is not iterable` — is fixed by adding `"buttons": []` to
every `$custom_*Board__c` mixin. Cheap, mandatory.

---

**TL;DR:** before you create `<thing>__c.json`, run `bloom recipe:pull` and
check whether `<thing>.json` already exists. If it does, write a
`$custom_<thing>.json` mixin instead. You will save yourself a day.
