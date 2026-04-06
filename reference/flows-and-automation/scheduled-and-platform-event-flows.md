# Scheduled, Autolaunched & Platform Event Flows -- Apex Developer's Guide

> **When to read this:** When building automation that runs on a schedule, when publishing or subscribing to platform events from flows or Apex, when invoking a flow from Apex or REST API, or when deciding between a declarative flow and an Apex async pattern (Schedulable, Queueable, @future, Batch, platform event trigger).

## Rules

- When you need to process a batch of records on a recurring schedule (daily, weekly), always evaluate schedule-triggered flows first because they require no code, no test classes, and no deployment -- but switch to Schedulable + Batch Apex when you need sub-daily frequency, custom cron expressions, processing more than 50,000 records, or complex multi-step orchestration with job chaining.
- Never use a schedule-triggered flow when the timing depends on a record field value (e.g., "3 days after CreatedDate") -- use a scheduled path on a record-triggered flow instead, because scheduled paths evaluate per-record relative to a date field while schedule-triggered flows run at a fixed clock time.
- When building a flow that will be called from Apex, REST API, or another flow, always create an autolaunched flow (no trigger) because this is the only flow type designed for programmatic invocation without a UI.
- Never mark flow input variables as "Available for Input" unless they are genuinely needed by callers, because every input variable exposed via REST API becomes part of the contract and cannot be removed without breaking integrations.
- When publishing platform events from a flow or Apex that modifies records in the same transaction, always use the "Publish After Commit" behavior on the event definition because "Publish Immediately" can cause subscribers to receive the event before the triggering transaction's data is committed, leading to stale reads.
- Never assume exactly-once delivery for platform events -- Salesforce uses at-least-once delivery. Always design platform event-triggered flows and Apex platform event triggers to be idempotent (safe to process the same event message multiple times).
- When subscribing to high-volume platform events, always add condition filters on the Start element of the platform event-triggered flow to reduce unnecessary interview invocations because every matching event message creates a flow interview.
- When building error handling for platform event flows, always subscribe a monitoring flow or Apex trigger to `FlowExecutionErrorEvent` because platform event-triggered flows fail silently by default with no user-facing error.
- When a schedule-triggered flow must process more than 50,000 records, always switch to Batch Apex because the Start element's Get Records query is subject to the 50,000-record SOQL row limit and there is no declarative workaround.
- When using scheduled paths on record-triggered flows, always configure the Default Workflow User in Setup > Process Automation Settings before activation because scheduled paths run as this user and fail silently if it is not set.
- When you need a flow to run at intervals shorter than daily (e.g., every hour, every 15 minutes), use Scheduled Apex to invoke an autolaunched flow because schedule-triggered flows only support Once, Daily, or Weekly frequencies natively. Alternatively, build the logic entirely in Schedulable + Queueable Apex for full control.

## How It Works

### Schedule-Triggered Flows

A schedule-triggered flow runs at a fixed time and frequency. When it fires, the Start element acts as a bulk query: it retrieves all records of the configured object that match the filter conditions, then feeds them through the flow logic in batches of 200. Each batch runs as a separate transaction with fresh governor limits.

**Frequency options:** Once (single execution at a specified date/time), Daily (every day at specified time), or Weekly (selected days at specified time). There is no native hourly, monthly, or custom cron expression support. For monthly patterns, set the flow to run daily and add a Decision element that checks a date formula (e.g., `DAY({!$Flow.CurrentDate}) = 1`).

**Time zone behavior:** The schedule uses the activating user's time zone. During DST transitions, a flow scheduled for 2:00 AM may fire at 1:00 AM or 3:00 AM UTC. Salesforce adjusts to the new offset without skipping or double-firing.

### Autolaunched Flows (No Trigger)

An autolaunched flow has no trigger -- it does not start on a schedule, record change, or platform event. It is invoked explicitly by:

- **Apex code** via `Flow.Interview`
- **REST API** via the Custom Invocable Actions endpoint
- **Another flow** via a Subflow element
- **Agentforce** actions
- **Custom buttons/links**

Autolaunched flows run in system context without sharing. They accept input variables and return output variables, making them function-like building blocks. Any variable marked "Available for Input" can be set by the caller; any variable marked "Available for Output" can be read after the flow completes.

### Platform Events: Publish/Subscribe Model

Platform events implement a publish/subscribe messaging pattern on the Salesforce event bus. A publisher creates an event message, and one or more subscribers receive and process it. Publishers and subscribers are fully decoupled.

Platform events are defined as custom objects with the `__e` suffix (e.g., `Order_Fulfillment__e`). Unlike standard/custom objects, platform events are transient messages that live on the event bus for a retention period, not persistent database records.

**Retention:** High-volume platform events are retained for **72 hours**. After 72 hours, event messages are no longer available for replay. Standard-volume events are not retained.

### Publish Behavior: Immediate vs After Commit

| Behavior | When Event Is Published | Transaction Rollback | DML Limit Impact | Use Case |
|----------|------------------------|---------------------|-----------------|----------|
| **Publish After Commit** | After transaction commits successfully | Event never publishes if transaction rolls back | 1 DML per publish (counts against 150 DML limit) | Default. Use when subscribers need committed data. |
| **Publish Immediately** | Immediately on `EventBus.publish()` or Create Records | Event publishes even if transaction rolls back | Separate 150 immediate publish limit | Logging, audit trails, fire-and-forget patterns. |

### Platform Event-Triggered Flow Execution

When a platform event-triggered flow is active, Salesforce subscribes it to the specified event channel. Each event message creates a flow interview in its own transaction, separate from the publishing transaction. The `{!$Record}` variable contains the event message fields.

Platform event-triggered flows process events **one at a time** by default. If multiple events arrive rapidly, they are queued and processed sequentially.

### Change Data Capture (CDC)

CDC events publish automatically when records of enabled objects are created, updated, deleted, or undeleted. They use the `ChangeEvent` suffix (e.g., `AccountChangeEvent`).

**Critical limitation:** CDC events cannot directly trigger a platform event-triggered flow. CDC events can only be consumed by Apex triggers on the change event object or external subscribers via Pub/Sub API. To invoke a flow from a CDC event, write an Apex trigger that calls `Flow.Interview.YourFlowName`.

### Scheduled Paths on Record-Triggered Flows

Scheduled paths are a feature of after-save record-triggered flows. They defer actions to a future time relative to a record event. The time source can be the flow trigger time or any date/datetime field on the record. The offset is in Minutes, Hours, or Days before or after the time source.

Scheduled paths re-evaluate entry conditions at fire time. If the record no longer meets the conditions, the path does not execute. They process records in batches of 200 with fresh governor limits.

## Apex Alternatives: When to Use Declarative vs Code

This is the core decision framework for Apex developers. Each declarative async pattern has an Apex equivalent with different strengths.

### Scheduled Flow vs Schedulable + Batch Apex

| Factor | Schedule-Triggered Flow | Schedulable + Batch Apex |
|--------|------------------------|------------------------|
| **Setup complexity** | Zero code, no test classes, configure in Flow Builder | Requires two Apex classes, 75%+ test coverage, deployment |
| **Scheduling options** | Once, Daily, Weekly only | Full cron expression support (hourly, every 15 min, first Monday of month, etc.) |
| **Record volume** | Up to 50,000 records per run (SOQL row limit on Start element) | Millions of records via `Database.QueryLocator` (up to 50M rows) |
| **Batch size** | Fixed at 200 records per interview | Configurable: 1-2,000 records per `execute()` call (default 200) |
| **Complex logic** | Limited to flow elements (Decision, Loop, Assignment, Get/Create/Update/Delete Records) | Full Apex: complex calculations, callouts, JSON parsing, multi-object orchestration |
| **Job chaining** | Not supported | Batch Apex can chain to another batch in `finish()` method; Queueable can chain to another Queueable |
| **Callouts** | Cannot make HTTP callouts from schedule-triggered flows | Batch Apex supports callouts with `Database.AllowsCallouts` |
| **Monitoring** | Flow Logging (Spring '26, requires Data Cloud), FlowExecutionErrorEvent | `AsyncApexJob` table, `CronTrigger` table, Setup > Apex Jobs, custom error logging |
| **Admin maintainability** | Admins can modify filters, logic, field mappings without deployment | Requires developer for any change, plus deployment |

**Decision guide:**
- **Use Scheduled Flow** when: the logic is straightforward (field updates, record creation, simple routing), volume is under 50,000 records, and admins need to maintain it.
- **Use Schedulable + Batch Apex** when: you need custom cron timing, volume exceeds 50,000 records, the logic requires callouts or complex orchestration, or you need job chaining.

### Scheduled Path vs @future / Queueable Apex

| Factor | Scheduled Path (on Record-Triggered Flow) | @future / Queueable Apex |
|--------|------------------------------------------|------------------------|
| **Trigger** | Relative to a record event or date field (e.g., "3 days after CreatedDate") | Called programmatically from a trigger, class, or flow |
| **Timing precision** | Minutes, Hours, or Days offset. Batched by platform -- exact fire time not guaranteed to the second | @future: runs "soon" (usually seconds to minutes). Queueable: runs "soon" with job ID tracking |
| **Use case** | Per-record follow-up automation: "send reminder 3 days before Close Date," "escalate if not resolved in 24 hours" | Post-DML async processing: callouts, heavy computation, operations that would exceed synchronous limits |
| **Re-evaluation** | Re-evaluates entry conditions at fire time -- if record no longer matches, path does not execute | No re-evaluation. Once enqueued, the job runs regardless of subsequent record changes (unless you add your own check) |
| **Sharing context** | Runs as the Default Workflow User | @future: runs as the invoking user. Queueable: runs as the invoking user |
| **Chaining** | Not applicable | Queueable can chain to another Queueable (1 chain in synchronous context, up to 5 in async). @future cannot chain. |
| **Bulk handling** | Batched at 200 records with fresh governor limits | @future: one call per invocation (max 50 per transaction). Queueable: one job per `System.enqueueJob()` call |

**Decision guide:**
- **Use Scheduled Paths** when: you need time-relative automation tied to a specific record event ("X days after Y field").
- **Use @future** when: you need a quick async callout or DML from a trigger and don't need chaining or complex inputs (Salesforce recommends Queueable over @future for new development).
- **Use Queueable** when: you need async processing with complex inputs (SObjects, collections), job chaining, or monitoring via `AsyncApexJob`.

### Platform Event Flow vs Apex Platform Event Trigger

| Factor | Platform Event-Triggered Flow | Apex Trigger on Platform Event |
|--------|------------------------------|-------------------------------|
| **Setup** | Declarative in Flow Builder | Apex trigger + handler class, test coverage, deployment |
| **Processing model** | One event at a time (sequential) | Batched: up to 2,000 events per trigger invocation |
| **Performance at scale** | Slower for high-volume bursts due to sequential processing | Significantly faster for high-volume scenarios -- parallel subscriber scaling available |
| **Complex logic** | Limited to flow elements | Full Apex: parsing complex JSON payloads, multi-object DML, callouts, retry logic |
| **CDC support** | Cannot subscribe to CDC events directly | Can subscribe to CDC events via `after insert` trigger on the ChangeEvent object |
| **Error handling** | Silent failures by default; requires FlowExecutionErrorEvent monitor | Full try-catch in Apex; can publish error events, create log records, send alerts |
| **User context** | Runs as the user who published the event (varies by originating user) | Runs as the Automated Process user |
| **Replay management** | Automatic (platform manages replay position) | Can manually manage via `EventBus.RetainedReplay` for advanced recovery scenarios |
| **Admin maintainability** | Admins can modify without deployment | Requires developer for any change |

**Decision guide:**
- **Use Platform Event Flow** when: event volume is low to moderate, logic is straightforward, and admins need to maintain it.
- **Use Apex Platform Event Trigger** when: you process high-volume event streams, need to handle CDC events, need parallel subscriber scaling, or the subscriber logic requires complex Apex (JSON parsing, callouts, multi-step orchestration).

## Invoking Autolaunched Flows from Apex

When Apex needs to delegate logic to a flow (e.g., business logic owned by admins, or logic that changes frequently and benefits from declarative maintenance), use `Flow.Interview`:

```apex
// build input map -- keys must match flow variable API names exactly (case-sensitive)
Map<String, Object> inputs = new Map<String, Object>();
inputs.put('accountId', acct.Id);
inputs.put('actionType', 'recalculate');

// create and start the flow interview
// the class name matches the flow's API Name with dots replaced by underscores
Flow.Interview.Calculate_Account_Health_Score flow =
    new Flow.Interview.Calculate_Account_Health_Score(inputs);
flow.start();

// retrieve output variables after the flow completes
Decimal score = (Decimal) flow.getVariableValue('healthScore');
String status = (String) flow.getVariableValue('healthStatus');
System.debug('// account health score: ' + score + ', status: ' + status);
```

**Key details:**
- Input variable names are **case-sensitive**. Passing `AccountId` when the variable is named `accountId` results in a silent null value.
- The flow runs synchronously within the calling Apex transaction. All governor limits are shared.
- If the flow fails, it throws a `Flow.FlowException` that you should catch in your Apex code.
- The flow runs in system context without sharing, regardless of the calling Apex class's sharing declaration.

### Alternative: Dynamic Flow Invocation

When you do not know the flow name at compile time (e.g., configurable automation):

```apex
// dynamic invocation using the createInterview static method
// useful when the flow name comes from Custom Metadata or a custom setting
String flowApiName = 'Calculate_Account_Health_Score';
Map<String, Object> inputs = new Map<String, Object>();
inputs.put('accountId', acct.Id);

Flow.Interview dynamicFlow = Flow.Interview.createInterview(flowApiName, inputs);
dynamicFlow.start();

Decimal score = (Decimal) dynamicFlow.getVariableValue('healthScore');
```

## Invoking Autolaunched Flows from REST API

External systems can invoke autolaunched flows via the Custom Invocable Actions endpoint:

**Endpoint:** `POST /services/data/v66.0/actions/custom/flow/{Flow_API_Name}`

**Request body:**
```json
{
  "inputs": [
    {
      "accountId": "001xx000003GYSAA2",
      "actionType": "recalculate"
    }
  ]
}
```

**Response:**
```json
[
  {
    "actionName": "Calculate_Account_Health_Score",
    "errors": null,
    "isSuccess": true,
    "outputValues": {
      "healthScore": 85,
      "healthStatus": "Good",
      "Flow__InterviewStatus": "Finished"
    }
  }
]
```

The `inputs` array supports batch invocation -- multiple sets of input variables in one call, with one result per invocation. Input variable names are case-sensitive and must match exactly.

## Flow Builder Navigation

### Creating a Schedule-Triggered Flow

Setup > Flows > New Flow > Schedule-Triggered Flow > Create
- Configure Start element:
  - Object: Select the object to query
  - Conditions: Define filter criteria (field/operator/value rows, or a custom condition formula)
  - Set Schedule: Choose frequency (Once / Daily / Weekly), Start Date, Start Time
  - For Weekly: select specific days (Monday, Tuesday, etc.)

### Creating an Autolaunched Flow (No Trigger)

Setup > Flows > New Flow > Autolaunched Flow (No Trigger) > Create
- No Start element configuration needed (no trigger, no object, no schedule)
- Define input/output variables in the Toolbox > Manager tab > New Resource > Variable
  - Check "Available for Input" and/or "Available for Output" as needed

### Creating a Platform Event-Triggered Flow

Setup > Flows > New Flow > Platform Event-Triggered Flow > Create
- Start element: Select the platform event (e.g., `Order_Fulfillment_Event__e`)
- Optionally add conditions to filter which event messages trigger the flow
- Build flow logic after the Start element

### Defining a Platform Event

Setup > Platform Events > New Platform Event
- Label, API Name (auto-generated with `__e` suffix)
- Publish Behavior: Publish After Commit (recommended) or Publish Immediately
- Add custom fields for the event payload

### Enabling Change Data Capture

Setup > Integrations > Change Data Capture
- Move objects from "Available Entities" to "Selected Entities"
- CDC events publish automatically when records of those objects change

### Adding a Scheduled Path to a Record-Triggered Flow

Setup > Flows > open an after-save record-triggered flow
- Start element must use "Only when a record is updated to meet the condition requirements"
- Click "Add Scheduled Paths (Optional)" on the Start element
- Configure each path: Time Source, Offset Number, Offset Unit (Minutes, Hours, or Days)

## Configuration Examples

### Example 1: Nightly Batch -- Close Stale Opportunities (Schedule-Triggered Flow)

**Object:** Opportunity
**Schedule:** Daily at 2:00 AM
**Start Conditions:** StageName NOT IN ("Closed Won", "Closed Lost") AND CloseDate < TODAY

**Flow Logic:**
1. Assignment: Set `{!$Record.StageName}` = "Closed Lost"
2. Assignment: Append " [Auto-closed by scheduled flow]" to `{!$Record.Description}`

The flow processes all matching records in batches of 200 automatically. No loop element needed -- schedule-triggered flows iterate over queried records inherently.

**When to switch to Apex:** If the org has 100,000+ stale Opportunities, or if the close logic needs to make callouts (e.g., notify an external CRM), use Batch Apex with `Database.AllowsCallouts`.

### Example 2: The Same Logic in Schedulable + Batch Apex

```apex
// the schedulable class -- runs on a cron schedule
public with sharing class CloseStaleOpportunitiesBatch implements
    Database.Batchable<SObject>, Schedulable {

    // schedulable entry point
    public void execute(SchedulableContext sc) {
        Database.executeBatch(new CloseStaleOpportunitiesBatch(), 200);
    }

    // batchable: query stale opportunities
    public Database.QueryLocator start(Database.BatchableContext bc) {
        return Database.getQueryLocator([
            SELECT Id, StageName, Description
            FROM Opportunity
            WHERE StageName NOT IN ('Closed Won', 'Closed Lost')
            AND CloseDate < TODAY
        ]);
    }

    // batchable: process each batch of 200
    public void execute(Database.BatchableContext bc, List<Opportunity> scope) {
        for (Opportunity opp : scope) {
            opp.StageName = 'Closed Lost';
            opp.Description = (opp.Description != null ? opp.Description : '')
                + ' [Auto-closed by batch job on ' + Date.today().format() + ']';
        }
        update scope;
        System.debug('// closed ' + scope.size() + ' stale opportunities in this batch');
    }

    // batchable: post-processing after all batches complete
    public void finish(Database.BatchableContext bc) {
        // optionally send a summary email or chain to another batch
        System.debug('// CloseStaleOpportunitiesBatch completed');
    }
}
```

```apex
// schedule it: daily at 2:00 AM
// cron expression: seconds minutes hours day-of-month month day-of-week year
String cronExpr = '0 0 2 * * ?';
System.schedule('Close Stale Opportunities - Daily', cronExpr, new CloseStaleOpportunitiesBatch());
```

### Example 3: Autolaunched Flow Called from Apex

**Flow:** `Calculate_Account_Health_Score` (Autolaunched, No Trigger)
**Input Variables:** `accountId` (Text, Available for Input)
**Output Variables:** `healthScore` (Number, Available for Output), `healthStatus` (Text, Available for Output)

```apex
// invoke the flow from a trigger handler or service class
public with sharing class AccountHealthService {

    // param: accountIds -- set of Account IDs to score
    // output: map of Account ID to health score
    public static Map<Id, Decimal> calculateHealthScores(Set<Id> accountIds) {
        Map<Id, Decimal> scores = new Map<Id, Decimal>();

        for (Id accId : accountIds) {
            Map<String, Object> inputs = new Map<String, Object>();
            inputs.put('accountId', accId);

            try {
                Flow.Interview.Calculate_Account_Health_Score flow =
                    new Flow.Interview.Calculate_Account_Health_Score(inputs);
                flow.start();

                Decimal score = (Decimal) flow.getVariableValue('healthScore');
                scores.put(accId, score);
            } catch (Exception ex) {
                // flow failures throw Flow.FlowException
                System.debug('// health score flow failed for account ' + accId + ': ' + ex.getMessage());
                scores.put(accId, null);
            }
        }

        return scores;
    }
}
```

**Warning:** Calling a flow in a loop like this is acceptable only for small volumes. Each `flow.start()` consumes governor limits (SOQL, DML) from the calling transaction. For bulk processing, consider redesigning the flow to accept a collection input or moving the logic into Apex entirely.

### Example 4: Publishing Platform Events from Apex

```apex
// publish a platform event from Apex
// the event definition must exist: Order_Fulfillment_Event__e
// with fields: Order_Id__c, Customer_Id__c, Total_Amount__c, Priority__c
List<Order_Fulfillment_Event__e> events = new List<Order_Fulfillment_Event__e>();

for (Order__c order : ordersToFulfill) {
    events.add(new Order_Fulfillment_Event__e(
        Order_Id__c = order.Id,
        Customer_Id__c = order.Account__c,
        Total_Amount__c = order.Total_Amount__c,
        Priority__c = order.Priority__c
    ));
}

// publish all events in one call -- batched for efficiency
List<Database.SaveResult> results = EventBus.publish(events);

// check for publish failures
for (Integer i = 0; i < results.size(); i++) {
    if (!results[i].isSuccess()) {
        for (Database.Error err : results[i].getErrors()) {
            System.debug('// event publish failed: ' + err.getStatusCode() + ' - ' + err.getMessage());
        }
    }
}
```

### Example 5: CDC Event to Flow via Apex Bridge

CDC events cannot trigger flows directly. Use an Apex trigger as a bridge:

```apex
trigger AccountCDCTrigger on AccountChangeEvent (after insert) {
    for (AccountChangeEvent event : Trigger.new) {
        EventBus.ChangeEventHeader header = event.ChangeEventHeader;

        // only process updates, not creates or deletes
        if (header.getChangeType() == 'UPDATE') {
            Map<String, Object> inputs = new Map<String, Object>();
            inputs.put('recordIds', header.getRecordIds());
            inputs.put('changeType', header.getChangeType());
            inputs.put('changedFields', String.join(header.getChangedFields(), ','));

            Flow.Interview.Account_CDC_Handler flow =
                new Flow.Interview.Account_CDC_Handler(inputs);
            flow.start();
        }
    }
}
```

### Example 6: Error Monitoring Flow for Silent Failures

**Flow Type:** Platform Event-Triggered Flow
**Platform Event:** `FlowExecutionErrorEvent` (standard Salesforce event)

**Flow Logic:**
1. Read error details from `{!$Record}`:
   - `{!$Record.FlowApiName}` -- which flow failed
   - `{!$Record.ElementApiName}` -- which element failed
   - `{!$Record.ErrorMessage}` -- the error message
2. Create Records: Create a custom `Flow_Error_Log__c` record with error details
3. Send Email: Notify the admin team

This catches silent failures in platform event-triggered flows and record-triggered flows. Always set this up in any org that uses platform event flows.

## Comparison: Schedule-Triggered Flow vs Scheduled Path

| Aspect | Schedule-Triggered Flow | Scheduled Path (on Record-Triggered Flow) |
|---|---|---|
| **When it runs** | Fixed clock time (daily, weekly, once) | Relative to a record event or date field |
| **Query behavior** | Queries all matching records at runtime | Applies only to the specific triggering record |
| **Use case** | Batch operations: nightly cleanup, weekly reports | Per-record follow-up: reminder 3 days before close date |
| **Re-evaluation** | Queries fresh each run | Re-evaluates entry conditions at fire time |
| **Batch size** | 200 records per interview | 200 records per batch |
| **Default Workflow User** | No (runs as activating user) | Yes -- required, scheduled paths run as this user |

## Governor Limits and Allocations

### Schedule-Triggered Flows

| Limit | Value | Notes |
|---|---|---|
| Max scheduled flow interviews per 24 hours | 250,000 or (licenses x 200), whichever is greater | All schedule-triggered flows combined |
| Batch size | 200 records per interview | Each batch is a separate transaction |
| SOQL queries per transaction | 100 | Per batch of 200 records |
| Start element query rows | 50,000 | If more records match, only first 50,000 are processed |
| DML statements per transaction | 150 | Per batch of 200 records |
| DML rows per transaction | 10,000 | Per batch of 200 records |
| Element executions per interview | 2,000 | Includes all elements |
| Scheduled paths per flow | 10 | Maximum on a single flow's Start element |

### Platform Events

| Limit | Value | Notes |
|---|---|---|
| Event messages published per hour (Unlimited) | 250,000 | Shared across all platform events |
| Event messages published per hour (Enterprise) | 25,000 | Lower editions have lower allocations |
| Event retention (high-volume) | 72 hours | After this, events cannot be replayed |
| DML per transaction (Publish After Commit) | 150 | Each Create Records / EventBus.publish counts as 1 DML |
| Immediate publish calls per transaction | 150 | Separate from DML limit |
| Platform event definitions per org | 200 (high-volume) + 5 (standard-volume) | High-volume recommended |

### Flow Monitoring (Spring '26)

Spring '26 introduces **Flow Persistent Logging** (GA), a native capability that captures run-level and element-level execution data including flow start/completion time, duration, status, and errors. Logs are stored in Data Cloud and can be surfaced via reports and dashboards. This is accessible through the Flow Logs tab in the Automation App. Note: this consumes Data Cloud credits.

For orgs without Data Cloud, the `FlowExecutionErrorEvent` monitoring pattern (Example 6 above) remains the primary error detection mechanism.

## Common Mistakes

1. **Using a schedule-triggered flow for per-record time-based actions** -- A schedule-triggered flow runs at a fixed clock time. "Send an email 3 days after each Case is created" would require complex date arithmetic and would process all Cases every run. Fix: Use a scheduled path on a record-triggered flow. The path fires per-record at the right time automatically.

2. **Not switching to Batch Apex when the schedule-triggered flow hits the 50,000-row limit** -- If the Start element's conditions match more than 50,000 records, the flow silently processes only the first 50,000 and ignores the rest. Fix: If your object regularly exceeds 50,000 matching records, switch to `Database.Batchable` with `Database.QueryLocator` (supports up to 50 million rows).

3. **Building a Schedulable Apex class for a simple nightly field update** -- A developer writes 200+ lines of Apex, a test class, deploys through a CI pipeline, and an admin cannot modify the filter criteria without a developer. Fix: If the logic is a simple field update on < 50,000 records with no callouts, use a schedule-triggered flow. Save Apex for complex scenarios.

4. **Expecting schedule-triggered flows to support hourly or custom cron expressions** -- The UI only offers Once, Daily, or Weekly. Fix: For sub-daily frequency, use `System.schedule()` with a cron expression to invoke an autolaunched flow. For full cron flexibility, build the logic in Schedulable Apex.

5. **Forgetting that autolaunched flow input variable names are case-sensitive in both Apex and REST** -- Passing `AccountId` when the variable is named `accountId` causes a silent null value. Fix: Match the exact API name case as defined in the flow.

6. **Using Publish Immediately when subscribers need committed data** -- A subscriber receives the event and queries the record, but the publishing transaction has not committed yet, so the query returns stale data or no data. Fix: Use "Publish After Commit" for any event where subscribers will query Salesforce data related to the publishing transaction.

7. **Not designing platform event handlers for idempotency** -- At-least-once delivery means a subscriber may receive the same event twice. Processing it twice creates duplicate records or sends duplicate emails. Fix: Include a unique identifier in the event payload (transaction ID or record ID) and check for duplicates before processing.

8. **Using platform events for simple same-object automation** -- A developer publishes a platform event from an Account trigger, then subscribes a flow to update the same Account. This adds unnecessary complexity and latency. Fix: Use a record-triggered flow on Account. Reserve platform events for cross-system integration, decoupled multi-subscriber scenarios, or fire-and-forget patterns.

9. **Not setting the Default Workflow User before using scheduled paths** -- Scheduled paths run as this user. If it is not set, paths fail silently. Fix: Setup > Process Automation Settings > set Default Workflow User to an active user with appropriate permissions.

10. **Calling autolaunched flows in a tight loop from Apex without considering governor limits** -- Each `flow.start()` call consumes SOQL, DML, and CPU from the calling transaction. Calling a flow 200 times in a loop will likely exceed the 100 SOQL query limit. Fix: Redesign the flow to accept collection inputs, or move the logic into Apex if bulk processing is required.

11. **Assuming CDC events can trigger flows directly** -- They cannot. CDC events can only be consumed by Apex triggers on the ChangeEvent object or external subscribers. Fix: Create an Apex trigger on the change event that invokes an autolaunched flow, passing event data as input variables (see Example 5).

12. **Not monitoring for silent failures in platform event flows** -- Platform event-triggered flows that encounter unhandled errors fail silently with no error email to the admin. Fix: Always create a monitoring flow or Apex trigger that subscribes to `FlowExecutionErrorEvent` and logs errors or sends notifications (see Example 6).

## See Also

- [Flow + Apex Coexistence Patterns](./flow-apex-coexistence.md) -- order of execution, error handling, automation density decisions
- [Screen Flows](./screen-flows.md) -- interactive flows with user screens, custom LWC components, Apex integration
- [Invocable Methods](../apex-fundamentals/invocable-methods.md) -- the @InvocableMethod pattern for Flow-to-Apex bridge
- [Apex Fundamentals](../apex-fundamentals/apex-fundamentals.md) -- sharing, governor limits, async Apex patterns (Queueable, Batch, @future, Schedulable)
- [Solution Architecture Patterns](../solution-architecture/solution-architecture-patterns.md) -- when to use which automation mechanism
