# Automation Configuration

Automations are no-code logic rules defined on object methods via the `jsActionable` field. They execute server-side when trigger events fire on a record.

## Where Automations Live

Automations are **not** separate files. They live inside object JSON files at `objects/<name>.json`, within the `methods[]` array. Any method with a `jsActionable` property is an automation.

## jsActionable Structure

```json
{
  "key": "Auto_Set_Default_Status",
  "event": "create",
  "label": "Auto Set Default Status",
  "action": { "$stub": true },
  "configKey": "method",
  "isDisabled": false,
  "permission": "insecure",
  "$declarator": {
    "key": "myObject__c",
    "configKey": "object"
  },
  "jsActionable": {
    "action": {
      "actionKey": "setValue",
      "toPath": "status__c",
      "literalValue": "new"
    },
    "condition": {
      "condition": {},
      "description": ""
    }
  }
}
```

### jsActionable Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `action` | ActionDefinition | Yes | The root action to execute (see Actions Reference) |
| `condition` | ConditionWrapper | No | Optional condition — automation only runs if matched |

### ConditionWrapper Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `condition` | object | Yes | The condition expression (see Conditions section). Use `{}` to always run |
| `description` | string | No | Human-readable description of the condition |

## Trigger Events

Automations fire based on the `event` field on the method:

| Event | Description | Typical Use Cases |
|-------|-------------|-------------------|
| `create` | Fires when a record is created | Set defaults, create related records |
| `update` | Fires on every record update | Recalculate values, sync fields |
| `commit` | Fires once per transaction (after all updates) | Final calculations, notifications, external calls |
| `delete` | Fires when a record is deleted | Clean up related records |
| `validate` | Fires before save | Validation rules |

## Actions Reference

Every action has an `actionKey` field plus action-specific properties.

### Action Summary

| actionKey | Description | Key Properties |
|-----------|-------------|----------------|
| `setValue` | Set a field to a literal value | `toPath`, `literalValue`, `isAppend` |
| `copyValue` | Copy one field's value to another | `fromPath`, `toPath`, `isAppend` |
| `createRecord` | Create a new record | `objectKey`, `initialValueActions`, `connectToPath`, `isAppend` |
| `removeValue` | Remove a value from a field | `fromPath`, `toPath`, `pathToCompare` |
| `compositeAction` | Execute multiple actions in sequence | `actions` |
| `loop` | Loop over an array and execute actions per item | `recordsPath`, `pathsToLoad`, `actions` |
| `mathExpression` | Evaluate a math expression | `expression`, `targetPath`, `symbolMap` |
| `invokeMethod` | Invoke another method on the same object | `methodKey` |
| `callAction` | Call a method on another object with parameter mapping | `methodKey`, `objectKey`, `parameters` |
| `callApi` | Call an external HTTP API | `url`, `method`, `headers`, `query`, `paths` |
| `sendNotification` | Send a notification via configured channels | `notificationKey`, `notificationTitle`, `notificationBody`, `notificationRecipientPath` |
| `scheduleAutomation` | Schedule an automation to run at a future time | `automationKey`, `schedule`, `automationToRun` |
| `callAiAgent` | Call an AI agent with a prompt | `aiAgentExternalId`, `prompt`, `parameters` |
| `queueWorkflow` | Queue a workflow for execution | `workflowKey`, `input`, `backgroundActionLabel` |
| `markConnectionsDirty` | Re-evaluate rules on connected objects | `paths` |
| `noop` | No operation (placeholder/testing) | — |

### setValue

Sets a field to a literal value.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `actionKey` | `"setValue"` | Yes | |
| `toPath` | string | Yes | Target field path (dot notation for nested) |
| `literalValue` | any | Yes | The value to set (string, number, boolean, array, object) |
| `isAppend` | boolean | No | If true, append to existing array instead of replacing |

```json
{
  "actionKey": "setValue",
  "toPath": "status__c",
  "literalValue": "active"
}
```

### copyValue

Copies a value from one field to another.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `actionKey` | `"copyValue"` | Yes | |
| `fromPath` | string | Yes | Source field path |
| `toPath` | string | Yes | Target field path |
| `isAppend` | boolean | No | If true, append to existing array instead of replacing |

```json
{
  "actionKey": "copyValue",
  "fromPath": "requestedDate__c",
  "toPath": "scheduledDate__c"
}
```

### compositeAction

Executes multiple actions in sequence.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `actionKey` | `"compositeAction"` | Yes | |
| `actions` | ActionDefinition[] | Yes | Array of actions to execute in order |

```json
{
  "actionKey": "compositeAction",
  "actions": [
    {
      "actionKey": "setValue",
      "toPath": "status__c",
      "literalValue": "processing"
    },
    {
      "actionKey": "copyValue",
      "fromPath": "submittedBy__c",
      "toPath": "assignedTo__c"
    }
  ]
}
```

### loop

Loops over an array field and executes actions for each item.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `actionKey` | `"loop"` | Yes | |
| `recordsPath` | string | Yes | Path to the array field to iterate over |
| `pathsToLoad` | string[] | No | Additional paths to load for each record |
| `actions` | ActionDefinition[] | Yes | Actions to execute per iteration |

```json
{
  "actionKey": "loop",
  "recordsPath": "lineItems__c",
  "actions": [
    {
      "actionKey": "setValue",
      "toPath": "processed__c",
      "literalValue": true
    }
  ]
}
```

### createRecord

Creates a new record and optionally connects it to the current record.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `actionKey` | `"createRecord"` | Yes | |
| `objectKey` | string | Yes | The object type to create (e.g. `"task__c"`) |
| `initialValueActions` | ActionDefinition[] | Yes | Actions to set initial field values on the new record |
| `connectToPath` | string | No | Connection field path on the current record to link the new record |
| `isAppend` | boolean | No | If true, append to existing connection array |

```json
{
  "actionKey": "createRecord",
  "objectKey": "auditLog__c",
  "connectToPath": "auditLogs__c",
  "isAppend": true,
  "initialValueActions": [
    {
      "actionKey": "setValue",
      "toPath": "action__c",
      "literalValue": "status_changed"
    },
    {
      "actionKey": "copyValue",
      "fromPath": "status__c",
      "toPath": "newValue__c"
    }
  ]
}
```

### removeValue

Removes a value from a field.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `actionKey` | `"removeValue"` | Yes | |
| `fromPath` | string | No | Source path to match for removal |
| `toPath` | string | Yes | Target field path to remove from |
| `pathToCompare` | string | No | Path used for comparison when removing from arrays |

```json
{
  "actionKey": "removeValue",
  "toPath": "assignees__c",
  "fromPath": "removedAssignee__c"
}
```

### mathExpression

Evaluates a math expression and stores the result.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `actionKey` | `"mathExpression"` | Yes | |
| `expression` | string | Yes | Math.js expression (e.g. `"a * b"`, `"sum(items)"`) |
| `targetPath` | string | Yes | Field path to store the result |
| `symbolMap` | object | No | Map variable names to field paths: `{ "a": "quantity__c", "b": "unitPrice__c" }` |

```json
{
  "actionKey": "mathExpression",
  "expression": "quantity * unitPrice",
  "targetPath": "totalPrice__c",
  "symbolMap": {
    "quantity": "quantity__c",
    "unitPrice": "unitPrice__c"
  }
}
```

### invokeMethod

Invokes another method on the same object.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `actionKey` | `"invokeMethod"` | Yes | |
| `methodKey` | string | Yes | Key of the method to invoke |

```json
{
  "actionKey": "invokeMethod",
  "methodKey": "Recalculate_Totals"
}
```

### callAction

Calls a method on another object with parameter mapping.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `actionKey` | `"callAction"` | Yes | |
| `methodKey` | string | Yes | Key of the method to call |
| `objectKey` | string | Yes | Key of the target object (e.g. `"invoice__c"`) |
| `parameters` | object | No | Map of parameter names to literal values or field path expressions |

```json
{
  "actionKey": "callAction",
  "methodKey": "Create_Invoice_Line",
  "objectKey": "invoice__c",
  "parameters": {
    "amount": { "path": "totalPrice__c" },
    "description": { "literalValue": "Auto-generated line item" }
  }
}
```

### callApi

Calls an external HTTP API.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `actionKey` | `"callApi"` | Yes | |
| `url` | string | Yes | The API endpoint URL |
| `method` | string | Yes | HTTP method: `"GET"`, `"POST"`, `"PUT"`, `"DELETE"` |
| `headers` | object | No | Request headers |
| `query` | object | No | Query string parameters |
| `paths` | string[] | No | Field paths to extract from the record and include in the request body |

```json
{
  "actionKey": "callApi",
  "url": "https://api.example.com/webhook",
  "method": "POST",
  "headers": { "Authorization": "Bearer token123" },
  "paths": ["fullId", "status__c", "customer__c.name"]
}
```

### sendNotification

Sends a notification through configured channels.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `actionKey` | `"sendNotification"` | Yes | |
| `notificationKey` | string | Yes | Unique notification identifier |
| `notificationTitle` | string | Yes | Notification title (supports field interpolation) |
| `notificationBody` | string | Yes | Notification body (supports field interpolation) |
| `notificationRecipientPath` | string | No | Field path to the recipient user/group |
| `notificationToRun` | object | No | Additional notification config |

Available channels: `email`, `inbox`, `toast`, `push`, `sms`. Defaults: `email` and `inbox`.

```json
{
  "actionKey": "sendNotification",
  "notificationKey": "order_approved_notification",
  "notificationTitle": "Order Approved",
  "notificationBody": "Your order has been approved and is ready for processing.",
  "notificationRecipientPath": "requestedBy__c"
}
```

### scheduleAutomation

Schedules another automation to run at a future date/time.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `actionKey` | `"scheduleAutomation"` | Yes | |
| `automationKey` | string | Yes | Key identifying this scheduled automation |
| `schedule` | string or object | Yes | ISO date string, or field path expression for a date field |
| `automationToRun` | string | Yes | Key of the method to invoke when the schedule fires |

```json
{
  "actionKey": "scheduleAutomation",
  "automationKey": "follow_up_reminder",
  "schedule": { "path": "dueDate__c" },
  "automationToRun": "Send_Follow_Up_Notification"
}
```

### callAiAgent

Calls an AI agent with a prompt and parameters.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `actionKey` | `"callAiAgent"` | Yes | |
| `aiAgentExternalId` | string | Yes | External ID of the AI agent to invoke |
| `prompt` | string | Yes | The prompt to send to the agent |
| `parameters` | object | No | Map of parameter names to field paths or literal values |

```json
{
  "actionKey": "callAiAgent",
  "aiAgentExternalId": "agent-classify-v1",
  "prompt": "Classify this support ticket by priority and category.",
  "parameters": {
    "ticketDescription": { "path": "description__c" },
    "ticketSubject": { "path": "subject__c" }
  }
}
```

### queueWorkflow

Queues a workflow for asynchronous execution.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `actionKey` | `"queueWorkflow"` | Yes | |
| `workflowKey` | string | Yes | Key of the workflow to queue |
| `input` | object | No | Input parameters with nested field mapping |
| `backgroundActionLabel` | string | No | Label shown in the UI while the workflow runs |

```json
{
  "actionKey": "queueWorkflow",
  "workflowKey": "generate_report_workflow",
  "backgroundActionLabel": "Generating report...",
  "input": {
    "recordId": { "path": "fullId" },
    "reportType": { "literalValue": "summary" }
  }
}
```

### markConnectionsDirty

Marks connected objects as needing rule re-evaluation.

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `actionKey` | `"markConnectionsDirty"` | Yes | |
| `paths` | string[] | Yes | Array of connection field paths to mark dirty |

```json
{
  "actionKey": "markConnectionsDirty",
  "paths": ["parentOrder__c", "invoice__c"]
}
```

### noop

No operation. Useful as a placeholder or for testing.

```json
{
  "actionKey": "noop"
}
```

## Conditions

Conditions control whether an automation runs. They are optional — omit the `condition` field or use `"condition": {}` to always run.

### Condition Structure

```json
{
  "condition": {
    "condition": {
      "status__c": { "$equals": "approved" }
    },
    "description": "Only run when status is approved"
  }
}
```

The inner `condition` object maps field paths to operator expressions.

### Comparison Operators

| Operator | Description | Example Value |
|----------|-------------|---------------|
| `$equals` | Exact match | `"active"`, `42`, `true` |
| `$notEquals` | Not equal | `"deleted"` |
| `$in` | Value is in array | `["new", "pending"]` |
| `$notIn` | Value is not in array | `["archived", "deleted"]` |
| `$lt` | Less than | `100` |
| `$lte` | Less than or equal | `100` |
| `$gt` | Greater than | `0` |
| `$gte` | Greater than or equal | `1` |
| `$contains` | String contains substring | `"urgent"` |
| `$startsWith` | String starts with | `"INV-"` |
| `$endsWith` | String ends with | `"@example.com"` |
| `$isEmpty` | Field is empty/null | `true` |
| `$isNotEmpty` | Field has a value | `true` |
| `$hasEvery` | Array contains all listed values | `["tag1", "tag2"]` |
| `$hasSome` | Array contains at least one value | `["priority", "urgent"]` |
| `$hasChanged` | Field changed since the triggering event | `true` |

### Multiple Conditions on One Field

```json
{
  "amount__c": { "$gte": 100, "$lt": 1000 }
}
```

### Multiple Field Conditions

All field conditions at the same level are implicitly ANDed:

```json
{
  "status__c": { "$equals": "active" },
  "amount__c": { "$gt": 0 }
}
```

### Logical Operators

Use `$and` and `$or` to combine conditions explicitly:

```json
{
  "$or": [
    { "status__c": { "$equals": "approved" } },
    { "priority__c": { "$equals": "critical" } }
  ]
}
```

```json
{
  "$and": [
    { "status__c": { "$hasChanged": true } },
    { "status__c": { "$equals": "complete" } }
  ]
}
```

### Qualifier Operators (for Connection Fields)

Apply conditions across related records:

| Operator | Description | Example |
|----------|-------------|---------|
| `$all` | Every connected record matches | All line items are shipped |
| `$some` | At least one connected record matches | Any item is overdue |
| `$none` | No connected records match | No items are rejected |

```json
{
  "lineItems__c": {
    "$all": {
      "status__c": { "$equals": "shipped" }
    }
  }
}
```

### Field Comparison Expressions

Compare a field to another field's value (instead of a literal):

```json
{
  "actualDate__c": { "$gt": { "path": "expectedDate__c" } }
}
```

### Date Math in Conditions

Add or subtract time intervals from date comparisons:

```json
{
  "dueDate__c": {
    "$lt": {
      "path": "today",
      "add": { "intervalUnit": "day", "interval": 7 }
    }
  }
}
```

### Case-Insensitive String Matching

```json
{
  "email__c": {
    "$equals": "admin@example.com",
    "caseInsensitive": true
  }
}
```

## Complete Examples

### Example 1: Set Default Value on Create

```json
{
  "key": "Auto_Set_Default_Status",
  "event": "create",
  "label": "Auto Set Default Status",
  "action": { "$stub": true },
  "configKey": "method",
  "isDisabled": false,
  "permission": "insecure",
  "$declarator": { "key": "order__c", "configKey": "object" },
  "jsActionable": {
    "action": {
      "actionKey": "setValue",
      "toPath": "status__c",
      "literalValue": "new"
    }
  }
}
```

### Example 2: Conditional Copy on Update

When status changes to "approved", copy the approver to assignee:

```json
{
  "key": "Auto_Assign_On_Approval",
  "event": "update",
  "label": "Auto Assign On Approval",
  "action": { "$stub": true },
  "configKey": "method",
  "isDisabled": false,
  "permission": "insecure",
  "$declarator": { "key": "request__c", "configKey": "object" },
  "jsActionable": {
    "action": {
      "actionKey": "copyValue",
      "fromPath": "approvedBy__c",
      "toPath": "assignedTo__c"
    },
    "condition": {
      "condition": {
        "$and": [
          { "status__c": { "$hasChanged": true } },
          { "status__c": { "$equals": "approved" } }
        ]
      },
      "description": "When status changes to approved"
    }
  }
}
```

### Example 3: Composite Action with Math

On commit, calculate line total and update status:

```json
{
  "key": "Auto_Calculate_And_Update",
  "event": "commit",
  "label": "Auto Calculate and Update",
  "action": { "$stub": true },
  "configKey": "method",
  "isDisabled": false,
  "permission": "insecure",
  "$declarator": { "key": "lineItem__c", "configKey": "object" },
  "jsActionable": {
    "action": {
      "actionKey": "compositeAction",
      "actions": [
        {
          "actionKey": "mathExpression",
          "expression": "quantity * unitPrice",
          "targetPath": "lineTotal__c",
          "symbolMap": {
            "quantity": "quantity__c",
            "unitPrice": "unitPrice__c"
          }
        },
        {
          "actionKey": "setValue",
          "toPath": "calculated__c",
          "literalValue": true
        }
      ]
    }
  }
}
```

### Example 4: Create Related Record

On create, create an audit log entry connected to the record:

```json
{
  "key": "Auto_Create_Audit_Log",
  "event": "create",
  "label": "Auto Create Audit Log",
  "action": { "$stub": true },
  "configKey": "method",
  "isDisabled": false,
  "permission": "insecure",
  "$declarator": { "key": "order__c", "configKey": "object" },
  "jsActionable": {
    "action": {
      "actionKey": "createRecord",
      "objectKey": "auditLog__c",
      "connectToPath": "auditLogs__c",
      "isAppend": true,
      "initialValueActions": [
        {
          "actionKey": "setValue",
          "toPath": "action__c",
          "literalValue": "created"
        },
        {
          "actionKey": "copyValue",
          "fromPath": "$createdBy",
          "toPath": "performedBy__c"
        }
      ]
    }
  }
}
```

### Example 5: Send Notification on Status Change

```json
{
  "key": "Notify_On_Approval",
  "event": "update",
  "label": "Notify On Approval",
  "action": { "$stub": true },
  "configKey": "method",
  "isDisabled": false,
  "permission": "insecure",
  "$declarator": { "key": "order__c", "configKey": "object" },
  "jsActionable": {
    "action": {
      "actionKey": "sendNotification",
      "notificationKey": "order_approved",
      "notificationTitle": "Order Approved",
      "notificationBody": "Your order has been approved and is being processed.",
      "notificationRecipientPath": "requestedBy__c"
    },
    "condition": {
      "condition": {
        "$and": [
          { "status__c": { "$hasChanged": true } },
          { "status__c": { "$equals": "approved" } }
        ]
      },
      "description": "When status changes to approved"
    }
  }
}
```

## Tips for AI Agents

- Automations live inside object configs in the `methods[]` array — they are **not** separate files
- The `action` field at the method top level must always be `{ "$stub": true }` — the real logic goes in `jsActionable.action`
- Use `compositeAction` when you need multiple actions from one trigger
- Use `loop` to iterate over array fields or connected records
- Conditions are optional — use `"condition": {}` or omit entirely to always run
- Set `"permission": "insecure"` for automations (they run server-side, not user-triggered)
- Set `"isDisabled": false` to enable, or `true` to temporarily disable without deleting
- Method keys should be descriptive: `Auto_Set_Default_Status`, `Calculate_Order_Total`, `Notify_On_Approval`
- The `$declarator` field links the method to its parent object — set `key` to the object key and `configKey` to `"object"`

## Action Keys: The Closed Union

`jsActionable.action.actionKey` is a **closed discriminated union**. Recipe push validation rejects any actionKey not in the accepted set.

**Accepted:** `setValue`, `copyValue`, `createRecord`, `removeValue`, `compositeAction`, `connectToPath`, `invokeMethod`, `sendNotification`

**REJECTED (do not use):** `notify`, `showToast`, `sendEmail`, `mathExpression`, `callAction`, `loop`, `removeRecord`

If you've authored `notify`, `showToast`, or `sendEmail` (the Builder UI surfaces those names), the canonical actionKey is **`sendNotification`** (see next section). For `mathExpression` use `setValue` with a formula-derived field. For `removeRecord` there is no recipe-supported equivalent — delete via API.

## `sendNotification` Action Shape

The `sendNotification` action requires a paired event declaration on the parent object's `events[]` array, plus the action's runtime config:

```json
{
  "actionKey": "sendNotification",
  "notificationConfig": {
    "notificationRecipientPath": "$updatedBy",
    "channels": ["inbox", "toast"],
    "title": "Order {fullId} approved",
    "body": "Customer {customer__c.name} approved order on {$updatedAt}"
  }
}
```

- `notificationRecipientPath` resolves to a user field (e.g., `$updatedBy`, or a custom user ref like `assignedTo__c`).
- `channels` accepts `"inbox"` and `"toast"`. Email is NOT a channel here — for email, use a Rocket Engine workflow.
- `title` and `body` support `{fieldKey}` template tokens (resolved at fire time against the source record). Dot-traversal works (`{customer__c.name}`).

To discover the exact event-declaration shape for your trigger, pull from a Builder UI-configured automation and inspect the event entry — the union is closed but well-defined.

## Event Timing Footgun: `event:"create"` Fires BEFORE Initial Values

When an automation triggers on `event: "create"`, it fires **before** `initialValueActions` are applied to the new record. If your condition reads a connection field that the caller is setting via `json: {parent__c: {id: ...}}`, the condition sees `null` and the action aborts.

**Fix:** trigger on `event: "update"` with a `$hasChanged` condition instead.

```json
{
  "event": "update",
  "condition": {
    "condition": {
      "$and": [
        { "parent__c": { "$hasChanged": true } },
        { "parent__c": { "$equals": null, "$not": true } }
      ]
    }
  }
}
```

This fires once after initial values land, with the parent ref populated.

## Cross-Record Writes vs Reads

| Direction | Supported? | How |
|---|---|---|
| **Write** to another record's field | ✅ Yes | `setValue` / `copyValue` with `toPath: "tractor__c.currentLocation__c"` (dot-traversal through ref) |
| **Read** another record's field in a condition | ❌ No | Cross-record condition reads silently no-fire the action |

**Workaround for cross-record reads:** denormalize the value via a LOOKUP-derived field on the source record, then condition on the local field. The LOOKUP updates reactively when the cross-record value changes.

## Self-Ref Copy: Not Directly Supported

There is no actionKey to write "this record's id" into a child record's ref field. Common workarounds:

1. **Sibling text field** — store the parent's `fullId` (string) in a text field on the child instead of an object ref. Acceptable when you don't need ref-traversal queries.
2. **Rocket Engine workflow** — `workflowUtils/createRecord/1.0` accepts cross-object record creation with explicit ref wiring; use it for the create-with-back-reference case.

## Recipe Lifecycle: What `recipe:push` Does and Doesn't Do

- **Adds and updates:** yes — every local config is upserted to the cookbook.
- **Deletions:** no — `recipe:push` does NOT execute deletions. Configs deleted on the cookbook but still present locally are **silently re-created** on the next push. To delete: `DELETE /api/v2/platformModel/<configType>/<key>` then republish.
- **Whole-folder upload:** `recipe:push` re-uploads the entire local recipe folder. If you removed a file locally hoping to delete the cookbook config, that won't happen — the file's absence is noted but no DELETE is sent.

Run `/bloom-recipe-preflight` before every push to surface these conditions.

## Forward-Ref Staged Push

Cross-object refs and self-refs in a single `recipe:push` batch fail with `BRL-6001 "Unknown object key"`. The server validator checks refs against the **current cookbook state**, not the batch being pushed.

**Fix: two-pass staged push.**

1. **Skeleton pass** — strip the forward refs from local files, push.
2. **Wire-up pass** — re-add refs, push again.

Automate by branching off a "skeleton" copy of the changed objects, pushing, restoring refs, and pushing again.

## Stub-Mixin Drift Unblocker

If `recipe:push` fails with `SDLC-4001: Unknown field X in object Y` because base-recipe drift broke the registry composition, an empty `$custom_<objectName>` mixin file can unblock — the platform's composition only checks for *existence* of the referenced base, not usage. Adding a stub mixin satisfies composition and lets your config push land.

## Debugging: Did the Automation Fire?

Use the `/events` API to verify what ran:

```
GET /api/v2/platformModel/events?recordId=<rid>&boardId=<bid>&objectKey=<okey>&limit=100&orderBy=createdAt&orderByDirection=desc&notInTypes=widget-open,widget-close,widget-skip
```

For the full diagnostic playbook (mutate events, automationResults[], conditionResults, eventual consistency), see `debugging-automations.md`.

For interactive root-cause analysis when an automation didn't fire as expected, run the `/bloom-trace-automation` skill from Claude Code.
