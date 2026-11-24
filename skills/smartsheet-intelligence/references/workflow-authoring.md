# Skill: Workflow Authoring

Triggers: "create automation", "create a workflow", "notify when", "remind when", "when status changes", "move completed rows", "auto-lock rows", "approval workflow", "list workflows", "disable/enable/delete workflow"

Scope: native sheet workflows (automations). Not Bridge, not Zapier/Make, not sprint/bulk row operations.

## Tool discovery (required first step)

Workflow tools live in the `automations` toolset. `tools/list` shows only the gateway tools (`search_tools` and the `execute_automations_*` executors); the workflow tools themselves (`get_workflows`, `create_workflow`, ...) are discovered and called through them:

1. `search_tools(toolset="automations", access_type="read"|"write")` → tool schemas
2. `execute_automations_read(tool_name="get_workflows", arguments={...})`
3. `execute_automations_write(tool_name="create_workflow"|"enable_workflow"|"disable_workflow"|"delete_workflow", arguments={...})`

Always `search_tools` before the first executor call. Never call the tools directly.

## Standard chains

- **Create**: `get_columns` (verify titles, types, picklist options) → preview to user → confirm → `create_workflow` → verify via `get_workflows`
- **List/inspect**: `get_workflows(sheet_id)` — each item's `root_node` is the same tree format `create_workflow` accepts; read it to explain or replicate a workflow
- **Enable/disable/delete**: `get_workflows` → confirm target by name + id → write tool with the id STRING exactly as returned

## create_workflow arguments

| Field | Value |
|-------|-------|
| `name` | Display name (≤255 chars) |
| `sheet_id` | Numeric sheet ID as shown in Smartsheet |
| `frequency` | `RIGHT_AWAY` \| `ONCE_PER_HOUR` \| `ONCE_PER_DAY` \| `ONCE_PER_WEEK` |
| `root_node` | `{"type": "ROOT_NODE", "trigger": {...}, "childNode": {...}}` — childNode is REQUIRED |

Tree keys are camelCase (`childNode`, `columnTitle`, `destinationSheetId`). Columns are referenced by TITLE (case-sensitive), people by EMAIL. Type discriminants are UPPER_SNAKE. Unknown/misspelled keys are rejected with the exact key name — fix the typo, don't remove the field.

## Triggers

| Type | Fields |
|------|--------|
| `ROW_CHANGE_TRIGGER` | `added`, `changed`, `deleted` (all required booleans); optional `criteria` list to narrow: `{"type": "CELL_CHANGE_CRITERIA", "columnTitle": "...", "values": [{"displayValue": "..."}]}` or `ATTACHMENT_CHANGE_CRITERIA`/`COMMENT_CHANGE_CRITERIA` with added/changed/deleted. For "changes to any value" omit `values` entirely - do NOT invent placeholder values |
| `ANY_CHANGE_TRIGGER` | none |
| `ROW_DELETED_TRIGGER` | none |
| `SHARING_TRIGGER` | `addedOrUpdated`, `deleted` |
| `DATE_TRIGGER` | `date` (ISO, e.g. `"2026-09-01T08:00:00Z"`), `hourOfDay` (0-23), `daysAfter` |
| `DATE_COLUMN_TRIGGER` | `daysAfter` (required; offset in days relative to the column date), `columnTitle` (DATE column), `hourOfDay` |
| `DATE_RECURRENCE_TRIGGER` | `repeatEvery` (≥1), `repeatFrequency` (`DAILY`/`WEEKLY`/`MONTHLY_DAY`/`MONTHLY_ORDINAL`/`YEARLY` — NO `ONCE`: a one-shot schedule is a `DATE_TRIGGER`), `startsOn` (ISO), `hourOfDay` (required), optional `endsOn`/`endsAfterTimes`/`repeatOn` |

`repeatOn` = `{"frequency": ..., "daysOfWeek": ["monday", ...], "dayOrdinal": "first", "dayOfMonth": N}`. Include `repeatOn` ONLY for `WEEKLY` (set `daysOfWeek`), `MONTHLY_DAY` (set `dayOfMonth`, 1-28), and `MONTHLY_ORDINAL` (set `dayOrdinal` + `daysOfWeek`); for `DAILY` and `YEARLY` OMIT `repeatOn` entirely — it is rejected. When present, set `repeatOn.frequency` to the SAME value as `repeatFrequency`. daysOfWeek/dayOrdinal values are LOWERCASE — the only lowercase enums in the tree. When replicating a tree returned by `get_workflows`, strip zero-value recurrence noise before writing (`endsAfterTimes: 0`, a `repeatOn` with `frequency: "ONCE"` or `dayOfMonth: 0`) — the write side rejects those.

## Actions (inside `{"type": "GENERAL_ACTION_NODE", "action": {...}}`)

| Type | Required fields |
|------|-----------------|
| `EMAIL_RECIPIENT` | `notificationCustomizationJson` (STRING — use `"{}"`), `recipients`; optional `subject`, `message`, `includeAttachments`, `includeComments`, `includedColumnTitles` |
| `UPDATE_REQUEST` | `notificationCustomizationJson` (`"{}"`), `recipients`; set `includedColumnTitles` or the request has no fields to update |
| `CHANGE_CELL` | `columnConfigs`: `[{"columnTitle": "...", "values": ["New"], "appendToMultiselect": false}]` — values are PLAIN STRINGS; `appendToMultiselect` is required |
| `CLEAR_CELL` / `RECORD_A_DATE` | `columnConfigs`: `[{"columnTitle": "..."}]` (RECORD_A_DATE needs DATE columns) |
| `MOVE_ROW` / `COPY_ROW` | `destinationSheetId` (numeric sheet ID) |
| `LOCK_ROW` / `UNLOCK_ROW` | none |
| `ASSIGN_PEOPLE` | `columnConfigs`: `[{"columnTitle": "...", "userEmails": [...], "appendToMultiselect": false}]` — emails must have existing contact records |
| `GENERATE_DOCUMENT` | `config`: `{"mappingTitle": "...", "mappingType": "...", "generationOption": "..."}` — all three required; `mappingTitle` must be an EXISTING mapping (unknown names rejected) |
| `REMINDER` | deprecated — use `EMAIL_RECIPIENT` instead |

**Recipients** (`recipients` entries — `id` is always a STRING):
- All shared users: `{"type": "ALL_SHARED_USERS", "id": "0"}`
- Contact column: `{"type": "COLUMN", "id": "<column title>"}`
- Specific person: `{"type": "USER", "id": "<email>"}`

## Conditions (IF_ELSE_NODE)

```json
{"type": "IF_ELSE_NODE",
 "conditionNodes": [{"type": "CONDITION_NODE", "logicalOperator": "AND",
   "conditions": [{"type": "ONE_VALUE", "columnTitle": "Status", "operator": "EQUAL",
                   "values": [{"displayValue": "Complete"}]}],
   "childNode": {"type": "GENERAL_ACTION_NODE", "action": {"type": "LOCK_ROW"}}}],
 "elseNode": {"type": "ELSE_NODE",
   "childNode": {"type": "GENERAL_ACTION_NODE", "action": {"type": "UNLOCK_ROW"}}}}
```

- Predicate `type` = arity: `ZERO_VALUE` (IS_BLANK, IS_CHECKED, TODAY — no `values`), `ONE_VALUE` (EQUAL, CONTAINS...), `TWO_VALUE` (BETWEEN)
- Condition values are OBJECTS: `{"displayValue": "text"}` or `{"numericValue": 123}` — NOT plain strings (unlike CHANGE_CELL values)
- Every CONDITION_NODE needs a `childNode` with an action — never leave it empty
- Operators (UPPER_SNAKE; complete list — negation spelling varies per operator, do not invent `NOT_X` forms): EQUAL, NOT_EQUAL, GREATER_THAN, GREATER_THAN_OR_EQUAL, LESS_THAN, LESS_THAN_OR_EQUAL, CONTAINS, DOES_NOT_CONTAIN, BETWEEN, NOT_BETWEEN, IS_BLANK, IS_NOT_BLANK, IS_NUMBER, IS_NOT_NUMBER, IS_DATE, IS_NOT_DATE, IS_CHECKED, IS_UNCHECKED, IS_ONE_OF, IS_NOT_ONE_OF, TODAY, NOT_TODAY, PAST, NOT_PAST, FUTURE, NOT_FUTURE, LAST_N_DAYS, NOT_LAST_N_DAYS, NEXT_N_DAYS, NOT_NEXT_N_DAYS, HAS_ANY_OF, HAS_NONE_OF, HAS_ALL_OF, NOT_ALL_OF, MULTI_IS_EQUAL, MULTI_IS_NOT_EQUAL

## Approvals (APPROVAL_ACTION_NODE)

```json
{"type": "APPROVAL_ACTION_NODE",
 "action": {"type": "APPROVAL", "notificationCustomizationJson": "{}",
   "recipients": [{"type": "COLUMN", "id": "Approver"}],
   "approvalColumnTitle": "Approval Status",
   "submittedColumnValue": "Submitted", "approvedColumnValue": "Approved",
   "rejectedColumnValue": "Declined",
   "approveButtonLabel": "Approve", "rejectButtonLabel": "Decline",
   "includedColumnTitles": ["Task Name", "Status"],
   "subject": "Row Approval Required", "message": "Please approve or decline this row change."},
 "ifApprovedNode": {"type": "GENERAL_ACTION_NODE", "action": {"type": "LOCK_ROW"}}}
```

Use a real column for `approvalColumnTitle` and a COLUMN/USER recipient — ALL_SHARED_USERS does not work for approvals. `approveButtonLabel`, `rejectButtonLabel`, and `includedColumnTitles` (real column titles) are REQUIRED even though the schema marks them optional — omitting any of them fails validation.

## Verified examples

**"Email everyone when Status changes to Resolved"** (tested end-to-end on staging; sheet IDs below are placeholders — always use the caller's real IDs):

```json
{"name": "Notify on resolution", "sheet_id": 1111111111111111, "frequency": "RIGHT_AWAY",
 "root_node": {"type": "ROOT_NODE",
  "trigger": {"type": "ROW_CHANGE_TRIGGER", "added": false, "changed": true, "deleted": false,
    "criteria": [{"type": "CELL_CHANGE_CRITERIA", "columnTitle": "Status",
                  "values": [{"displayValue": "Resolved"}]}]},
  "childNode": {"type": "GENERAL_ACTION_NODE",
    "action": {"type": "EMAIL_RECIPIENT", "notificationCustomizationJson": "{}",
      "subject": "Issue resolved", "message": "The issue was marked Resolved.",
      "recipients": [{"type": "ALL_SHARED_USERS", "id": "0"}]}}}}
```

**"Move completed rows to the archive sheet"** (structure verified on staging):

```json
{"name": "Archive completed", "sheet_id": 1111111111111111, "frequency": "ONCE_PER_DAY",
 "root_node": {"type": "ROOT_NODE",
  "trigger": {"type": "ROW_CHANGE_TRIGGER", "added": false, "changed": true, "deleted": false},
  "childNode": {"type": "IF_ELSE_NODE",
    "conditionNodes": [{"type": "CONDITION_NODE", "logicalOperator": "AND",
      "conditions": [{"type": "ONE_VALUE", "columnTitle": "Status", "operator": "EQUAL",
                      "values": [{"displayValue": "Complete"}]}],
      "childNode": {"type": "GENERAL_ACTION_NODE",
        "action": {"type": "MOVE_ROW", "destinationSheetId": 2222222222222222}}}]}}}
```

## Gotchas

- `notificationCustomizationJson` is REQUIRED on EMAIL_RECIPIENT, UPDATE_REQUEST, and APPROVAL — always pass the string `"{}"`. Note: `get_workflows` may return it as `""`; when replicating a returned tree, replace `""` with `"{}"` before creating.
- Workflow `id` values from create/get are STRINGS — pass them unchanged to enable/disable/delete. Never convert to numbers.
- Column titles and picklist values are case-sensitive and validated — always `get_columns` first; never invent columns or picklist values.
- `root_node.childNode` can never be null or omitted.
- Reminder-type workflows are not returned by `get_workflows`.
- Pagination: iterate with the returned `last_key`; trees are large, prefer small `max_items`.

## Safety rules (non-negotiable)

1. **Preview before create**: show a plain-language summary ("When X happens, if Y, then Z — checked {frequency}") and wait for explicit yes. Never create unprompted.
2. **Confirm before delete**: restate name + id + that deletion is permanent; suggest `disable_workflow` as the reversible alternative. Wait for explicit yes.
3. **Verify after create**: call `get_workflows` and confirm the workflow appears with the expected tree; report name, id, and status.
4. One workflow per request — for "and also..." follow-ups, run the full preview cycle again.
