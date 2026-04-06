# Record-Triggered Flows (for Apex Developers)

> **When to read this:** When building any flow that fires automatically when a record is created, updated, or deleted -- including choosing between before-save, after-save, scheduled paths, and asynchronous execution -- or when deciding whether a record-triggered flow or an Apex trigger is the right tool for the job.

## Rules

- When you only need to update fields on the triggering record, always use a before-save flow because it consumes zero DML operations and runs up to 10x faster than an after-save flow. Think of it as the declarative equivalent of a `before insert` / `before update` trigger that modifies `Trigger.new` in-place.
- Never use a before-save flow to create, update, or delete related records because before-save flows only support Assignment, Decision, Loop, Get Records, and Formula elements -- no DML elements are available. If you need cross-object DML, use an after-save flow or an Apex trigger.
- When you need to create child records, send emails, post to Chatter, invoke Apex, or launch subflows, always use an after-save flow (Actions and Related Records) because only after-save flows have the full element library.
- Never put Get Records, Create Records, Update Records, or Delete Records elements inside a loop because each execution counts against the 100 SOQL or 150 DML per-transaction limit. This is the Flow equivalent of putting SOQL/DML inside a `for` loop in Apex -- same governor limits, same consequences.
- When configuring entry conditions on an update-triggered flow, always use "Only when a record is updated to meet the condition requirements" when you want the flow to fire only on the transition (not every subsequent save where the condition is already true) because this prevents redundant re-execution. This is analogous to checking `Trigger.oldMap.get(id).Status__c != Trigger.newMap.get(id).Status__c` in Apex.
- When you have multiple record-triggered flows on the same object, always set explicit Trigger Order values in flow properties because Salesforce chooses an arbitrary order when no order is specified. Apex developers: this is like having multiple unordered trigger handlers with no framework controlling sequence.
- Never assume a before-save flow has access to the record Id on insert because the Id has not been assigned yet at that point in the save cycle -- same as `Trigger.new[0].Id` being null in a `before insert` trigger.
- When using scheduled paths, always configure the org's Default Workflow User (Setup > Process Automation Settings) before activating the flow because scheduled paths run as this user and fail silently without it.
- Record-triggered flows always run in system context without sharing. Never rely on them to enforce record-level security -- they bypass sharing rules, org-wide defaults, and role hierarchy. This is equivalent to an Apex trigger's implicit `without sharing` behavior, except you cannot change it. In Apex, at least you can call a `with sharing` helper class.
- When both a before-save flow and a before Apex trigger exist on the same object and modify the same field, the Apex trigger always wins because it executes after the before-save flow (step 5 vs step 4 in the order of execution). Never rely on a flow to set a field that a trigger also sets.

## How It Works

A record-triggered flow fires automatically when a DML event (insert, update, delete, or undelete in some cases) occurs on the configured object. The flow does not require user interaction -- it runs behind the scenes as part of the record save transaction. You configure the trigger in the Start element by choosing the object, the trigger event (create, update, create or update, delete), and the optimization (Fast Field Updates for before-save, or Actions and Related Records for after-save).

**Before-save flows** execute early in the order of execution, before Apex before triggers and before custom validation rules. They modify the triggering record's field values in memory -- conceptually identical to modifying `Trigger.new` records in a `before insert` / `before update` trigger. Because these changes are written to the record before the database save, no additional DML statement is consumed. This makes before-save flows extremely efficient -- Salesforce benchmarks them at roughly 10x faster than after-save flows for the same field-update work. The tradeoff is a restricted element palette: only Assignment, Decision, Loop, Get Records, and Formula resources are available. You cannot create related records, send notifications, or call external services.

**After-save flows** execute later in the order of execution, after the record has been written to the database (but before the transaction is committed). They have access to the full element library: Create Records, Update Records, Delete Records, Get Records, Decision, Assignment, Loop, Send Email, Post to Chatter, Custom Notification, Invoke Apex, Launch Subflow, and all action elements. Because the record already exists in the database at this point, the record Id is available and you can create child records that reference it -- same as an `after insert` trigger where `Trigger.new[0].Id` is populated. However, any update to the triggering record in an after-save flow requires a separate DML statement that counts against the 150 DML limit.

**Bulkification**: When a batch of records triggers the flow (e.g., a Data Loader insert of 200 records), Salesforce creates one flow interview per record. All interviews run in the same transaction. The runtime batches interviews together at bulkifiable elements (DML and SOQL elements), executing those operations as a single bulk call. This means a Get Records inside a loop does not necessarily issue 200 separate SOQL queries -- the runtime batches them. However, you should still design as if bulkification might not fully optimize every scenario. The maximum batch size is 200 records per transaction -- identical to the `Trigger.new` batch size in Apex triggers.

## Order of Execution: Where Flows and Apex Triggers Interleave

This is the critical sequence that every Apex developer must internalize. When a record is saved, every automation in this list shares a single transaction and a single set of governor limits.

| Step | What Executes | Type |
|---|---|---|
| 1 | Load original record from database (update) or initialize new record (insert) | System |
| 2 | Load new field values from the request | System |
| 3 | System validation (first pass): field types, string lengths, schema-required fields | System |
| **4** | **Before-save record-triggered flows** (Fast Field Updates) | **Flow** |
| **5** | **Before Apex triggers** (`before insert`, `before update`, `before delete`) | **Apex** |
| 6 | System validation (second pass): re-check required fields after custom logic | System |
| 7 | Custom validation rules | Declarative |
| 8 | Duplicate rules | Declarative |
| 9 | Record saved to database (Id assigned on insert), but transaction NOT committed | System |
| **10** | **After Apex triggers** (`after insert`, `after update`, `after delete`) | **Apex** |
| 11 | Assignment rules (Lead/Case) | Declarative |
| 12 | Auto-response rules | Declarative |
| 13 | Workflow rules (legacy). If workflow field updates fire, they re-execute steps 5 and 10 (before/after triggers). Flows do NOT re-fire. | Legacy |
| 14 | Escalation rules | Declarative |
| **15** | **After-save record-triggered flows** (Actions and Related Records) | **Flow** |
| 16 | Entitlement rules | Declarative |
| 17 | Roll-up summary field calculations | System |
| 18 | Cross-object workflow rules | Legacy |
| 19 | Criteria-based sharing rules | System |
| 20 | Transaction committed | System |
| 21 | Post-commit: emails sent, `@future` methods run, queueable jobs start, async flow paths execute, platform events fire | Async |

### What This Means for Apex Developers

**Before-save flows run BEFORE your before triggers (step 4 vs step 5).** If both modify the same field, your trigger wins because it runs second. But this also means a before-save flow can set a field value that your `before insert` trigger logic then reads -- and the value will be whatever the flow set, not the user's original input (unless you check `Trigger.old`).

**After triggers run BEFORE after-save flows (step 10 vs step 15).** An after trigger that creates child records or updates related objects will have those changes visible to an after-save flow's Get Records elements. Useful pattern: trigger does the heavy computation, flow does lightweight follow-up like notifications.

**After-save flow updates to the triggering record restart the FULL cycle.** If an after-save flow uses Update Records on the triggering record, the entire order of execution (steps 1-20) runs again -- including your Apex triggers. Salesforce limits after-save flow re-entry to two executions per record per transaction. Your trigger handler must be recursion-safe.

**Your Apex triggers share governor limits with all flows.** A before-save flow that runs 10 Get Records elements before your trigger even fires has consumed 10 of your 100 SOQL queries. Always audit the full automation stack on an object before adding triggers or flows.

## Before-Save vs After-Save vs Asynchronous: Comparison

| Aspect | Before-Save (Fast Field Updates) | After-Save (Actions & Related Records) | Run Asynchronously |
|---|---|---|---|
| **When it runs** | Before Apex before triggers, before validation rules | After DML, after Apex after triggers, before commit | After commit, in a separate transaction |
| **DML cost to update triggering record** | Zero (modifies in memory) | 1 DML statement per Update Records call | 1 DML statement per Update Records call |
| **Available elements** | Assignment, Decision, Loop, Get Records, Formula | Full element library (Create, Update, Delete, Email, Apex, Subflow, etc.) | Full element library |
| **Record Id available on insert?** | No | Yes | Yes |
| **Can create/update related records?** | No | Yes | Yes |
| **Can send email / notifications?** | No | Yes | Yes |
| **Governor limits context** | Same transaction as the triggering save | Same transaction as the triggering save | Separate transaction (fresh limits) |
| **Scheduled paths available?** | No | Yes | No |
| **Apex trigger equivalent** | `before insert` / `before update` modifying `Trigger.new` | `after insert` / `after update` performing DML | `@future` or `Queueable` launched from trigger |
| **Performance** | ~10x faster than after-save | Standard | Deferred; no user-perceived latency |
| **Use case** | Field defaulting, field stamping, simple validation-like logic | Cross-object updates, child record creation, notifications | External callouts, heavy processing, non-blocking work |

## When to Use a Record-Triggered Flow vs an Apex Trigger

This is the question every Apex developer asks. The answer depends on complexity, volume, and organizational context.

### Decision Matrix

| Factor | Favor Flow | Favor Apex Trigger |
|---|---|---|
| **Logic complexity** | Simple field defaulting, status stamping, single-object conditionals | Complex multi-object logic, dynamic SOQL, aggregate queries, intricate business rules |
| **Record volume** | UI-scale saves (1-10 records at a time) | Batch-scale operations (200+ records via Data Loader, API, or batch Apex) |
| **Automation density** | 1-2 automations on the object | 5+ automations already firing on the object (consolidate in a trigger framework) |
| **Maintainer skill set** | Admins maintain the logic; developers are not always available | Dedicated developer team maintaining the codebase |
| **Data operations needed** | Standard CRUD on a few objects | Dynamic SOQL, aggregate queries (`GROUP BY`, `HAVING`), SOSL, 50k+ row queries |
| **Error handling** | Simple pass/fail with Custom Error element | Complex partial-success scenarios, `Database.SaveResult` inspection, retry logic |
| **Callout requirements** | No callouts (or simple ones via async path) | Complex HTTP callout chains, certificate-based auth, retry/backoff patterns |
| **Unit testing** | Flow testing is limited -- no programmatic assertion of intermediate states | Full Apex test framework with `System.assert`, `Test.startTest`, mock callouts |
| **Governor limit sensitivity** | Low overall consumption, plenty of headroom | Already near governor limits; need precise control over every query and DML |
| **Deployment** | Deploys as metadata (FlowDefinition); admins can activate/deactivate | Deploys as ApexClass + ApexTrigger; requires 75%+ test coverage |

### The Hybrid Pattern

The most common production pattern is a hybrid: use flows for straightforward declarative logic and Apex for the parts that flows handle poorly.

**Example**: When an Opportunity closes, a before-save flow stamps `Closed_Quarter__c` (zero DML, admin-maintainable). An after-save flow fires and calls an `@InvocableMethod` that generates a PDF quote, emails it, and creates a Task (complex logic in Apex, triggered by Flow). The flow handles the simple part; Apex handles the hard part.

**Rule of thumb**: If you can describe the logic in one sentence ("set Region based on BillingState"), use a flow. If describing the logic requires a paragraph with conditionals and edge cases, use Apex.

## The $Record and $Record__Prior Variables

Every record-triggered flow automatically provides two global variables -- think of these as the flow equivalents of `Trigger.new` and `Trigger.old`:

- **$Record** -- Contains the current field values of the triggering record. In a before-save flow, `$Record` reflects the values as submitted (analogous to `Trigger.new`). In an after-save flow, `$Record` reflects the values as they exist after the database save. Use `{!$Record.FieldName}` to reference fields.
- **$Record__Prior** -- Contains the field values of the record before the current save (analogous to `Trigger.old`). Only available when the trigger event includes "update" (not on insert, since there is no prior state). Use `{!$Record__Prior.FieldName}` to compare old vs new values.

### Formula functions for entry conditions and decision elements

These functions work in flow formulas and entry condition formulas:

- **ISNEW()** -- Returns TRUE if the record is being created (insert). Equivalent to `Trigger.isInsert` in Apex.
- **ISCHANGED({!$Record.FieldName})** -- Returns TRUE if the specified field's value differs from its prior value. Equivalent to `Trigger.oldMap.get(id).Field != Trigger.newMap.get(id).Field` in Apex.
- **PRIORVALUE({!$Record.FieldName})** -- Returns the prior value of the field. Equivalent to `Trigger.oldMap.get(id).Field` in Apex.

**Important**: ISCHANGED() is not available in the entry condition formula when the run condition is set to "Only when a record is updated to meet the condition requirements." Use it in Decision elements or in entry conditions with "Every time a record is updated and meets the condition requirements."

## Entry Conditions

The Start element of a record-triggered flow has three condition-logic options:

1. **All Conditions Are Met (AND)** -- Every condition row must be true. Simplest option for straightforward filters.
2. **Any Condition Is Met (OR)** -- At least one condition row must be true.
3. **Formula Evaluates to True** -- Write a custom formula that returns TRUE/FALSE. Use this for complex logic combining AND/OR, ISCHANGED(), ISNEW(), PRIORVALUE(), and cross-field comparisons.

### When to run for updated records

When the trigger event includes updates, you choose one of:

- **Every time a record is updated and meets the condition requirements** -- The flow runs on every save where the conditions are satisfied, even if the record already met them before. Similar to an Apex trigger with no `oldMap` comparison.
- **Only when a record is updated to meet the condition requirements** -- The flow runs only when the record transitions from NOT meeting the conditions to meeting them. This is the "isChanged-like" behavior at the entry level. Required for scheduled paths. Equivalent to checking `Trigger.oldMap.get(id).Status != 'Closed' && Trigger.newMap.get(id).Status == 'Closed'` in Apex.

## Scheduled Paths

Scheduled paths let you defer part of a record-triggered flow to run at a future time. They are only available on after-save flows configured with "Only when a record is updated to meet the condition requirements."

**Configuration:**
- Time Source: A date or datetime field on the triggering record, or the time the flow triggered
- Offset: Number of minutes, hours, or days before or after the time source
- Batch size: Scheduled paths process up to 200 records per batch

**Key behaviors:**
- The Default Workflow User runs the scheduled path. Set this in Setup > Process Automation Settings.
- If the record no longer meets the entry conditions when the scheduled path fires, the path does not execute.
- If the record is deleted before the scheduled path fires, the path does not execute.
- Scheduled paths run in a separate transaction with fresh governor limits.
- For testing, set the offset to 0 minutes. The path executes within ~1 minute. Reset the offset before deploying to production.

### Scheduled Paths vs Scheduled Apex

| Aspect | Scheduled Paths (Flow) | Scheduled Apex (`Schedulable` + `Database.Batchable`) |
|---|---|---|
| **Trigger** | Record-level: fires when a specific record meets conditions | Org-level: runs on a cron schedule regardless of individual records |
| **Granularity** | Per-record offset from a date/datetime field | Fixed schedule (e.g., daily at 2 AM) |
| **Use case** | "Send reminder 3 days before CloseDate" | "Recalculate all account health scores every night" |
| **Batch size** | 200 records per batch (automatic) | Configurable scope via `Database.executeBatch(job, scope)` |
| **Error handling** | Limited -- path silently skips if record no longer qualifies | Full Apex `try/catch`, `Database.SaveResult`, custom error logging |
| **Governor limits** | Fresh transaction per batch | Fresh transaction per `execute()` batch |
| **Cancellation** | Automatic if record no longer meets conditions | Must be managed in code or via Setup > Scheduled Jobs |
| **Dynamic timing** | Yes -- offset is relative to a field value that can change | No -- cron schedule is static unless reprogrammed |
| **Monitoring** | Setup > Flow Trigger Explorer; limited debug visibility | Setup > Scheduled Jobs; full Apex debug logs; `AsyncApexJob` queries |

**Rule of thumb**: Use scheduled paths when the timing is relative to individual records ("3 days before this record's CloseDate"). Use Scheduled Apex when the timing is absolute and org-wide ("every night at midnight, process all records matching criteria").

## Run Asynchronously Path

The Run Asynchronously path executes after the transaction commits. Salesforce queues the work and processes it shortly after (usually within seconds, but no SLA is guaranteed).

**Key behaviors:**
- Runs in a separate transaction with fresh governor limits (new 100 SOQL, 150 DML limits).
- Any values or queries in the async path reflect changes committed by the synchronous path.
- The async path does not begin until all immediate-path work completes and the transaction commits.
- Use this for callouts to external systems, heavy processing, or work that should not slow down the user's save experience.
- You cannot guarantee exact execution order among multiple async paths.

### Run Asynchronously vs @future / Queueable Apex

| Aspect | Run Asynchronously (Flow) | `@future` Method | `Queueable` Apex |
|---|---|---|---|
| **Invocation** | Automatic from record-triggered flow config | Called from trigger/Apex code | `System.enqueueJob()` from trigger/Apex |
| **Chaining** | No chaining -- single async execution | No chaining | Can chain: one Queueable enqueues another (depth limit: 5 in test, no hard limit in prod but governed) |
| **Parameters** | Full `$Record` context available | Only primitive types (`Id`, `String`, `List<Id>`, etc.) | Full object state via constructor -- can pass sObjects, collections, complex state |
| **Callouts** | Supported (full element library) | Requires `callout=true` annotation | Implement `Database.AllowsCallouts` |
| **Monitoring** | Limited Flow debug tools | Apex debug logs, `AsyncApexJob` | Apex debug logs, `AsyncApexJob`, can query job status |
| **Error handling** | Fault paths in flow; limited retry capability | `try/catch` in the method; no built-in retry | `try/catch`; can implement custom retry by re-enqueueing |
| **Governor limits** | Fresh async limits | Fresh async limits (200 SOQL, 12 MB heap) | Fresh async limits (200 SOQL, 12 MB heap) |
| **Complexity ceiling** | Moderate -- limited by Flow element capabilities | Low -- only primitives, no state | High -- full Apex with complex state, chaining, batching |
| **Best for** | Simple post-save work: send notification, create related record, call subflow | Quick fire-and-forget: update a field, make a simple callout | Complex async work: multi-step processing, callout chains, retry logic, stateful operations |

**Rule of thumb**: Use Run Asynchronously for simple declarative work that should not block the user's save. Use `@future` for quick fire-and-forget Apex operations (field updates, simple callouts). Use `Queueable` for anything that needs state, chaining, or complex error handling.

## Decision Elements and Branching

Decision elements evaluate conditions and route the flow down different paths (outcomes). Each outcome has its own condition logic (AND, OR, or formula). The flow evaluates outcomes in order from top to bottom; the first matching outcome wins. An unmatched scenario falls to the Default Outcome.

**Best practice:** If you find yourself placing a Decision element immediately after the Start element to branch into completely different business logic paths, split the logic into multiple record-triggered flows instead. Use the Flow Trigger Explorer to manage execution order. Each flow should have focused entry conditions. This improves maintainability and debugging. An Apex developer would recognize this as the single-responsibility principle -- one flow per concern, just as you would separate trigger handler methods by concern.

## Flow Trigger Explorer

When you have multiple record-triggered flows on the same object, use Flow Trigger Explorer to visualize and control execution order.

**Navigation:** Setup > Flows > (open any flow) > Flow Trigger Explorer (button in toolbar), or Setup > Object Manager > [Object] > Flow Triggers

The explorer groups flows by execution phase:
1. Before-save flows
2. After-save flows
3. Run Asynchronously flows
4. Scheduled paths

Within each group, you can drag flows to set execution order or set the Trigger Order property (numeric value, 1-2000) in flow properties. Lower numbers run first. Flows without an explicit order run in an undefined sequence after those with explicit ordering. This is roughly analogous to using a trigger framework that controls handler method execution order -- except it is point-and-click.

## Governor Limits (Per Transaction)

All elements in the immediate (synchronous) paths of a record-triggered flow share the same transaction as the triggering DML and any Apex triggers on the same object. These limits are cumulative across ALL automation in the transaction:

| Limit | Value | Apex Developer Note |
|---|---|---|
| SOQL queries | 100 | Same limit your triggers share. A flow's Get Records elements count against this. |
| DML statements | 150 | Same limit. A flow's Create/Update/Delete elements count against this. |
| Records retrieved by SOQL | 50,000 | Shared ceiling. A flow retrieving 40,000 records leaves 10,000 for your triggers. |
| Records modified by DML | 10,000 | Shared ceiling. |
| CPU time | 10,000 ms (10 seconds) | Flow element evaluation consumes CPU time. Complex formulas in flows add up. |
| Heap size | 6 MB (synchronous) | Large collections in flows consume heap. |
| Callouts | 100 (after-save only; not available in before-save) | Same pool. If your trigger's `@future` method uses 50, the flow has 50 left. |

**Before-save flows are free**: Updating fields on `$Record` in a before-save flow consumes zero DML and zero SOQL (unless you use Get Records). This is why before-save is always preferred when you only need to set field values on the triggering record -- same reason Apex developers modify `Trigger.new` in-place rather than issuing `update` DML in a before trigger.

## Recursion Prevention and Re-Entry

Salesforce has built-in recursion prevention for record-triggered flows:

- A record-triggered flow on a given object runs at most **two times** per record per transaction for after-save flows (the initial execution plus one re-entry if the record is updated again within the same transaction). If a third re-entry would occur, the flow does not execute again.
- Before-save flows similarly have recursion guards, but the specifics depend on the trigger event.
- When an after-save flow updates the triggering record, that update re-invokes the full order of execution (including before-save flows and Apex triggers). Design entry conditions to be idempotent to prevent unnecessary re-execution.

**How this interacts with Apex trigger recursion**: If an after-save flow updates the triggering record, your Apex trigger fires again. If your trigger also updates the record (or a related record that cascades), you can get a chain reaction. The flow's built-in 2-execution limit helps, but your Apex trigger has no such automatic guard. Always use static recursion guards (`static Set<Id> processedIds`) in your trigger handler.

**Best practices for preventing unwanted recursion:**
1. Use entry conditions that compare `$Record` to `$Record__Prior` so the flow only fires when the relevant field actually changed.
2. Use "Only when a record is updated to meet the condition requirements" to fire only on transition.
3. Avoid updating fields on the triggering record in an after-save flow when a before-save flow would suffice -- after-save updates cause a re-save cycle that re-fires your Apex triggers.
4. If you must update the triggering record in an after-save flow, ensure the update does not change any field that your flow entry conditions or Apex trigger conditions evaluate.

## Configuration Examples

### Example 1: Before-Save -- Auto-Populate Region Based on State

**Object:** Account
**Trigger:** Create or Update
**Optimization:** Fast Field Updates (before-save)
**Entry Conditions (Formula):**
```
OR(
  ISNEW(),
  ISCHANGED({!$Record.BillingState})
)
```

**Flow logic:**
1. Decision element: Check `{!$Record.BillingState}`
   - Outcome "West": BillingState IN ("CA", "OR", "WA", "NV", "AZ")
   - Outcome "East": BillingState IN ("NY", "NJ", "CT", "MA", "PA")
   - Default Outcome: "Other"
2. Assignment element (per outcome): Set `{!$Record.Region__c}` to "West", "East", or "Other"

No DML consumed. Runs before validation rules. **Apex equivalent**: a `before insert, before update` trigger with a `switch` on BillingState, but the flow is admin-maintainable without a deployment.

### Example 2: After-Save -- Create Follow-Up Task When Opportunity Closes

**Object:** Opportunity
**Trigger:** Update
**Optimization:** Actions and Related Records (after-save)
**Entry Conditions:**
- Condition: `{!$Record.StageName}` Equals "Closed Won"
- When to run: Only when a record is updated to meet the condition requirements

**Flow logic:**
1. Create Records element: Create a Task
   - Subject: "Follow up on closed opportunity: {!$Record.Name}"
   - WhoId: `{!$Record.ContactId}` (primary contact)
   - OwnerId: `{!$Record.OwnerId}`
   - ActivityDate: `{!$Flow.CurrentDate}` + 7 (formula for 7 days from now)
   - Status: "Not Started"

Consumes 1 DML statement. Record Id is available for the WhoId lookup. **When to use Apex instead**: If the Task creation has complex conditional logic (different task types, different assignees based on role hierarchy, bulk insert optimization), put it in an `@InvocableMethod` called from this flow, or move the entire thing to an after trigger.

### Example 3: Scheduled Path -- Send Reminder 3 Days Before Close Date

**Object:** Opportunity
**Trigger:** Create or Update
**Optimization:** Actions and Related Records (after-save)
**Entry Conditions:**
- Condition: `{!$Record.StageName}` NOT Equals "Closed Won" AND NOT Equals "Closed Lost"
- When to run: Only when a record is updated to meet the condition requirements

**Scheduled Path:**
- Time Source: `{!$Record.CloseDate}`
- Offset: 3 Days Before
- Actions: Send Email to Opportunity Owner with reminder

If the Opportunity closes before the scheduled path fires, the record no longer meets entry conditions and the path is skipped. **When to use Scheduled Apex instead**: If you need to process thousands of records in a nightly batch with complex aggregation logic, use `Schedulable` + `Database.Batchable`. Scheduled paths are better for per-record, time-relative actions.

### Example 4: Entry Condition Formula -- Complex Multi-Field Logic

```
AND(
  NOT(ISNEW()),
  ISCHANGED({!$Record.Status__c}),
  TEXT({!$Record.Status__c}) = "Escalated",
  TEXT(PRIORVALUE({!$Record.Status__c})) <> "Escalated",
  {!$Record.Priority__c} = "High"
)
```

This fires only when: the record is updated (not new), the Status field changed, the new status is "Escalated", the prior status was not already "Escalated" (prevents re-entry), and the priority is High. **Apex equivalent**: `if (!Trigger.isInsert && Trigger.oldMap.get(id).Status__c != 'Escalated' && rec.Status__c == 'Escalated' && rec.Priority__c == 'High')`.

### Example 5: Hybrid Pattern -- Flow Triggers Invocable Apex

**Object:** Purchase_Order__c
**Trigger:** Update
**Optimization:** Actions and Related Records (after-save)
**Entry Conditions:** Status__c changed to "Approved" (transition-based)

**Flow logic:**
1. Apex Action: `GeneratePurchaseOrderPdfAction` (invocable method)
   - Input: `{!$Record.Id}`
   - Output: `Result` wrapper with `isSuccess`, `errorMessage`, `pdfContentVersionId`
2. Decision: Check `{!result.isSuccess}`
   - Yes: Send Custom Notification to owner
   - No: (Log error -- flow ends; the save is not rolled back unless you add a Custom Error)

The flow handles the orchestration; the Apex handles the PDF generation, email, and complex logic. Neither tool does work it is bad at.

## Testing Record-Triggered Flows

### Flow Tests (Summer '25+)

Salesforce now supports declarative Flow Tests for record-triggered flows. You can create test cases directly in Flow Builder that:
- Define input record values (the triggering record)
- Define expected output values (what the record should look like after the flow runs)
- Assert on intermediate values at specific flow elements
- Run automatically as part of deployment validation

**Key capability (Summer '25):** The "Has Error" assertion operator allows you to test fault paths -- verify that your flow correctly handles error scenarios (e.g., Custom Error elements fire when expected).

**Limitations compared to Apex tests:**
- Cannot assert on related records created by the flow (only the triggering record)
- Cannot test complex multi-object cascading scenarios
- Cannot mock external callouts
- Cannot test timing-dependent scheduled paths
- Cannot programmatically generate test data in bulk

**When to supplement with Apex tests:** If your record-triggered flow calls an `@InvocableMethod`, write Apex tests for the invocable method separately. The Apex test can exercise the method in bulk (200+ records), test edge cases, and assert on all downstream effects. The Flow test covers the orchestration; the Apex test covers the logic.

### Debugging Record-Triggered Flows

**Debug logs:** Record-triggered flows appear in Apex debug logs under the `FLOW_START_INTERVIEWS` event. Each flow interview logs:
- The flow version and API name
- Entry condition evaluation result
- Each element executed, with variable values
- Any faults or errors

**Debug flow (Flow Builder):** Open the flow in Flow Builder → click Debug → choose a record to simulate. This runs the flow in a debug transaction and shows the path taken through each element. Useful for isolated testing, but does not show interactions with Apex triggers (those do not fire in debug mode).

**Flow Trigger Explorer:** For understanding the full execution landscape on an object: Setup > Object Manager > [Object] > Flow Triggers. Shows all record-triggered flows with their execution order, grouped by phase (before-save, after-save, async, scheduled).

**Apex developers:** When debugging an issue that involves both flows and triggers, use the Apex debug log with `FLOW` log category set to `FINE` or `FINER`. The log will interleave flow execution events with your Apex trigger execution, showing the exact order of operations and where values change.

## Common Mistakes

1. **Using after-save to update the triggering record's own fields** -- This wastes a DML statement and triggers a re-save cycle that re-invokes the entire order of execution, including your Apex triggers. Fix: Use a before-save flow instead. If you need both before-save field updates and after-save related-record work, create two separate flows.

2. **Putting DML or SOQL elements inside a loop** -- A Data Loader operation of 200 records fires 200 flow interviews in one transaction. A Get Records inside a loop can hit the 100 SOQL limit instantly -- same as putting SOQL inside a `for` loop in Apex. Fix: Collect records into a collection variable inside the loop, then use a single Create/Update/Delete Records element after the loop.

3. **Not checking for null before operating on Get Records results** -- If Get Records returns no results and you reference a field from the result, the flow faults. Fix: Add a Decision element after Get Records to check whether the result variable Is Null. Apex developers: this is the equivalent of not checking `queryResult.isEmpty()` before accessing `queryResult[0]`.

4. **Using "Every time a record is updated" when "Only when updated to meet the condition" is appropriate** -- This causes the flow to fire on every save, wasting governor limits and potentially causing side effects (duplicate emails, duplicate child records). Fix: Use the transition-based option and design entry conditions around the specific field change.

5. **Forgetting to set Default Workflow User before using scheduled paths** -- Scheduled paths fail silently if no Default Workflow User is configured. Fix: Setup > Process Automation Settings > Default Workflow User. Set it to an active user with appropriate permissions.

6. **Assuming record-triggered flows enforce sharing** -- Record-triggered flows run in system context without sharing. They can read and modify any record regardless of the running user's permissions. Fix: Do not rely on flow execution context for data security. Use validation rules, sharing rules, or Apex with `with sharing` for security enforcement.

7. **Creating a single flow with a Decision element right after Start to handle multiple unrelated scenarios** -- This creates a monolithic flow that is hard to debug and maintain. Fix: Split into multiple focused flows, one per business scenario. Use Flow Trigger Explorer to control execution order.

8. **Not accounting for recursion when an after-save flow updates the triggering record** -- The update re-invokes the order of execution. If entry conditions are not tight enough, the flow fires again. Worse, your Apex triggers fire again too -- and they may not have recursion guards. Fix: Ensure entry conditions use ISCHANGED() or "Only when updated to meet the condition" so the re-save does not re-trigger the flow. Add static recursion guards in your Apex trigger handler.

9. **Testing scheduled paths with long delays** -- Waiting days for a scheduled path to fire during testing is impractical. Fix: Set the offset to 0 minutes during testing (fires within ~60 seconds). Reset to the real offset before deploying to production.

10. **Ignoring the order of execution for before-save flows** -- Before-save flows run before custom validation rules. If your before-save flow populates a required field, validation passes. But if you expect validation to reject the record before your flow runs, it will not -- the flow runs first. Fix: Understand the full order of execution and design accordingly.

11. **Adding a record-triggered flow to an object without auditing existing Apex triggers** -- The new flow shares governor limits with existing triggers. A flow that uses 30 SOQL queries on an object where triggers already use 65 will crash at 101 during batch operations. Fix: Before adding any flow, audit all automation on the object (triggers, flows, workflow rules). Profile total SOQL/DML consumption. Only add automation if headroom exists. See the automation inventory template in [Flow + Apex Coexistence](./flow-apex-coexistence.md).

12. **Expecting a before-save flow value to persist when a before trigger also sets the same field** -- The trigger runs at step 5 (after the flow at step 4) and silently overwrites the flow's value. An admin changes the flow, tests it in an environment without the trigger, deploys -- and it does not work. Fix: Never have both a before-save flow and a before trigger modify the same field. Document field ownership.

## See Also

- [Flow + Apex Coexistence Patterns](./flow-apex-coexistence.md)
- [Flow Elements & Resources Reference](./flow-elements-and-resources.md)
- [Invocable Methods -- Flow-to-Apex Bridge](../apex-fundamentals/invocable-methods.md)
- [Apex Fundamentals (triggers, sharing, testing)](../apex-fundamentals/apex-fundamentals.md)
