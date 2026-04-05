# Flow + Apex Coexistence Patterns

> **When to read this:** You have both Flows and Apex triggers on the same object (or are deciding which to use), you need to understand how they interleave during the save order of execution, or you are building a screen flow that calls an invocable Apex method and need to handle errors cleanly.

## Rules

- When choosing between a record-triggered flow and an Apex trigger for the same object, always evaluate **automation density** first -- the cumulative count of flows, triggers, workflow rules, and process builders firing on a single DML event. Low density (1-2 automations) favors Flow. Medium density (3-4) favors Flow + Invocable Apex hybrid. High density (5+) favors consolidating into an Apex trigger framework.
- Never duplicate the same business logic in both a flow and an Apex trigger on the same object because maintenance diverges immediately and the two implementations will produce different results when the underlying rule changes.
- When a before-save flow and a before Apex trigger both modify the same field, the before trigger always wins because it executes after the before-save flow (step 5 vs step 4). Never rely on a before-save flow to set a field that a before trigger also sets -- the trigger's value overwrites the flow's value silently.
- When an `@InvocableMethod` is called from a screen flow, always catch exceptions inside the method and return error information in a Result wrapper rather than letting the exception propagate to the flow's fault path, because unhandled exceptions roll back the entire transaction and show the user a generic "An unhandled fault has occurred" message with no useful context.
- When an `@InvocableMethod` is called from a record-triggered flow and you want to block the save with a user-facing error, use the **Custom Error** element in the flow after checking the Result wrapper's error flag -- do not throw an exception from the Apex method, because the Custom Error element gives you control over the error message location (field-level or page-level) and rolls back the transaction cleanly.
- Never add a record-triggered flow to an object that has a complex Apex trigger framework without first mapping the full order of execution for that object, because the flow will interleave with the trigger at specific points and can cause unexpected field overwrites, recursion, or governor limit exhaustion.
- When you have both flows and triggers on the same object, always document which mechanism owns which fields and which business rules, because debugging a production issue across interleaved automation is nearly impossible without a written inventory.
- When building a screen flow that calls Apex via an invocable action, always wire a **fault connector** on the Apex Action element even if you expect the Apex to handle errors internally, because network timeouts, uncatchable `LimitException`, and unexpected platform errors will still propagate as faults.
- Never throw `AuraHandledException` from an `@InvocableMethod` expecting special behavior -- Flow does not recognize `AuraHandledException` as distinct from any other exception. It treats it identically to a generic `Exception`: the flow hits the fault path, and `{!$Flow.FaultMessage}` contains the exception message. `AuraHandledException` is exclusively for `@AuraEnabled` methods called by LWC/Aura components.
- When consolidating automation on a high-density object, always pick **one primary mechanism** per object per trigger event. If the object already has an Apex trigger handler framework, put new automation in the trigger handler. If the object is Flow-first with no trigger, keep it in Flow. Do not mix mechanisms on the same object unless you have a documented architectural reason.

## How It Works

### The Decision Framework: Automation Density

Salesforce's official Record-Triggered Automation Decision Guide (architect.salesforce.com) defines automation density across three dimensions:

1. **Automation Quantity** -- The raw count of metadata entries (flows, trigger handler methods, legacy workflow rules, process builders) that execute during a single DML event on the object.
2. **Record Volume** -- How many records move through the object at a given time. Individual UI edits vs Data Loader batches of 10,000+.
3. **Dependency Sprawl** -- How deeply your automation is intertwined with other objects. An after-save flow that creates child records on three different objects has high dependency sprawl.

| Density Level | Automation Quantity | Record Volume | Recommended Approach |
|---|---|---|---|
| **Low** | 1-2 automations per event | UI-scale (1-10 records) | Record-triggered Flow alone |
| **Medium** | 3-4 automations per event | Mixed (UI + occasional batch) | Flow orchestration + Invocable Apex for complex logic |
| **High** | 5+ automations per event | Batch-scale (200+ records regularly) | Apex trigger framework with handler classes; eliminate flows on this object |

**Concrete examples:**

- **Simple field defaulting** (set Region from BillingState on Account insert) -- Low density, no cross-object logic. Use a before-save flow. An admin can maintain this without developer involvement.
- **Status change notification** (email the owner when Opportunity Stage changes to Closed Won) -- Low density. Use an after-save flow with entry conditions on ISCHANGED. No Apex needed.
- **PDF generation + email + logging** (generate a Purchase Order PDF, email it, stamp digital authorization, create a Task) -- Medium density, complex multi-step logic with callouts. Use a screen flow for the user interaction + an `@InvocableMethod` for the backend logic.
- **High-volume Account automation** (5 different automations fire on Account update: sync to ERP, recalculate health score, update related Contacts, validate compliance fields, publish platform event) -- High density. Consolidate everything into an Apex trigger handler. Flows on this object become a liability.

### Order of Execution: Where Flows and Triggers Interleave

This is the complete save order showing exactly where record-triggered flows and Apex triggers execute relative to each other and to other automation. Every piece of automation in the synchronous phase shares one transaction and one set of governor limits.

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
| 13 | Workflow rules execute. If workflow field updates fire, they modify the record and cause **one additional re-execution** of steps 5, 6, and 10 (before triggers, system validation, after triggers). Custom validation rules, before-save flows, and after-save flows do NOT re-fire during this mini-cycle. | Declarative/Legacy |
| 14 | Escalation rules | Declarative |
| **15** | **After-save record-triggered flows** (Actions and Related Records) | **Flow** |
| 16 | Entitlement rules | Declarative |
| 17 | Roll-up summary field calculations (triggers parent's own save cycle) | System |
| 18 | Cross-object workflow rules | Declarative/Legacy |
| 19 | Criteria-based sharing rules | System |
| 20 | Transaction committed | System |
| 21 | Post-commit: emails sent, `@future` methods run, queueable jobs start, async flow paths execute, platform events fire | Async |

### What the Order Means in Practice

**Before-save flows run BEFORE before triggers (step 4 vs step 5).** If both modify the same field, the trigger's value wins because it runs second. This is the opposite of what many developers expect -- they assume triggers run first because triggers feel more "fundamental." They do not. Before-save flows get first crack at the record, but triggers overwrite them.

**After triggers run BEFORE after-save flows (step 10 vs step 15).** An after trigger that creates child records or updates related objects will have those changes visible to an after-save flow's Get Records elements. This is useful: the trigger does the heavy lifting, and the flow handles lightweight follow-up like notifications.

**Workflow field updates (step 13) cause triggers to re-fire but NOT flows.** When a legacy workflow rule has a field update, Salesforce re-runs before triggers (step 5) and after triggers (step 10) one additional time. Before-save flows (step 4) and after-save flows (step 15) are NOT re-executed during this mini-cycle. This means trigger code needs static recursion guards, but flow code does not need to worry about workflow-induced re-entry.

**After-save flow updates to the triggering record restart the FULL cycle.** If an after-save flow uses an Update Records element on the triggering record, the entire order of execution (steps 1-20) runs again. Salesforce limits after-save flow re-entry to two executions per record per transaction. Before-save flows and Apex triggers WILL re-fire during this restart.

**Field value precedence -- later steps win:**

```
Before-save flow sets Region = "West"         (step 4)
Before trigger sets Region = "Override"        (step 5)  ← trigger wins
Validation rules check Region is not null      (step 7)  ← passes, value is "Override"
Record saved to database                       (step 9)  ← "Override" committed
Workflow field update sets Region = "Legacy"   (step 13) ← workflow wins
Trigger re-fires, could overwrite again        (step 5a) ← depends on trigger logic
After-save flow reads Region                   (step 15) ← sees whatever survived
```

### Flow Error Handling with Apex Exceptions

There are three distinct error-handling scenarios depending on the flow type and the Apex method's behavior.

#### Scenario 1: Screen Flow + Invocable Apex (Recommended: Return Errors in Result Wrapper)

The Apex method catches all exceptions internally and returns success/failure in the Result inner class. The flow checks the `isSuccess` field and branches to a success screen or an error screen. No transaction rollback occurs because the Apex never threw an exception -- it returned a clean result.

```
Screen Flow:
  [Collect Input] → [Call Apex Action] → Decision: {!result.isSuccess}?
                                           ├── Yes → [Success Screen]
                                           └── No  → [Error Screen: {!result.errorMessage}]
```

This is the recommended pattern for all screen flows. The user sees a friendly error message. The flow remains in control. No fault connector needed (but wire one anyway as a safety net).

#### Scenario 2: Screen Flow + Invocable Apex (Fault Connector Path)

The Apex method throws an unhandled exception (or you intentionally let it propagate). The flow's Apex Action element must have a **fault connector** wired to an error-handling path. The `{!$Flow.FaultMessage}` system variable contains the exception's `getMessage()` text. The entire transaction rolls back -- all DML performed by the Apex method and all prior flow DML in the same transaction are undone.

```
Screen Flow:
  [Collect Input] → [Call Apex Action] ──fault──→ [Error Screen: {!$Flow.FaultMessage}]
                          │
                          └── success ──→ [Success Screen]
```

Use this pattern only when you want all-or-nothing behavior and the error message from the exception is meaningful to the user. If the exception message is a raw Apex stack trace, the user sees gibberish.

**How `{!$Flow.FaultMessage}` works:** When any element in a flow throws an unhandled fault, Salesforce populates `{!$Flow.FaultMessage}` with the exception message. If the Apex method throws `new PurchaseOrderException('Contact has no email address')`, the flow variable contains exactly that string: "Contact has no email address". If the method throws a raw `DmlException`, the message will be the DML error text, which is often technical.

#### Scenario 3: Record-Triggered Flow + Custom Error Element

In a record-triggered flow (before-save or after-save), you cannot show a screen to the user. Instead, use the **Custom Error** element (available since Winter '24) to block the save and display a user-facing error message on the record page. The Custom Error element rolls back the entire transaction.

```
Record-Triggered Flow (before-save):
  [Entry Conditions] → [Get Records: check related data] → Decision: valid?
                                                              ├── Yes → (flow ends, save proceeds)
                                                              └── No  → [Custom Error: "Cannot save because..."]
```

The Custom Error element supports two display modes:
- **Page-level error**: Shows at the top of the record page, like a validation rule error.
- **Field-level error**: Shows below a specific field, exactly like a field-level validation rule.

If a record-triggered flow calls an Apex Action and the Apex returns an error in the Result wrapper, the flow can read the error and route to a Custom Error element to block the save with a meaningful message. This is cleaner than letting the Apex throw an exception, which would produce an "An unhandled fault has occurred" message.

#### AuraHandledException in Flow Context

`AuraHandledException` has **no special behavior** when thrown from an `@InvocableMethod` called by a Flow. Flow does not use the Aura/LWC framework for its server communication -- it uses the Flow runtime, which treats all exceptions identically. When a Flow encounters an `AuraHandledException`, it:

1. Hits the fault connector (if wired) or shows the generic unhandled fault error (if not wired)
2. Populates `{!$Flow.FaultMessage}` with the exception's `getMessage()` text
3. Rolls back the transaction

This is identical to what happens with any other exception type (`DmlException`, custom exceptions, `CalloutException`, etc.).

`AuraHandledException` is designed exclusively for the `@AuraEnabled` → LWC/Aura communication channel. It strips the stack trace and sends only the message to the client. In Flow context, this stripping behavior is irrelevant because the flow runtime already extracts just the message for `{!$Flow.FaultMessage}`.

**Rule:** Never throw `AuraHandledException` from an `@InvocableMethod`. If you want to signal an error to a flow, return it in the Result wrapper. If you want to signal an error to an LWC, throw `AuraHandledException` from an `@AuraEnabled` method. These are two different communication patterns for two different callers.

### Screen Flow Design Patterns for Apex Integration

#### The Multi-Screen Pattern: Collect → Confirm → Execute → Result

The standard pattern for screen flows that call Apex follows four phases:

1. **Collect Screen** -- Gather user input (record lookups, text fields, picklists). No DML happens here.
2. **Confirm Screen** -- Display a summary of what will happen. The user reviews and clicks "Execute" (the Next button). No DML happens here.
3. **Execute** -- An Apex Action element (or sequence of flow DML elements) performs the work. No screen is displayed during execution.
4. **Result Screen** -- Display success or error information. This is the final screen.

```
[Collect: Select Contact via Lookup]
        ↓
[Get Records: Fetch PO details for confirmation]
        ↓
[Confirm: "Send PO-0001 PDF to Jane Doe (jane@example.com)?"]
        ↓
[Apex Action: SendPurchaseOrderPdfAction]
        ↓
[Decision: {!result.isSuccess}?]
   ├── Yes → [Success Screen: "PDF sent to jane@example.com"]
   └── No  → [Error Screen: {!result.errorMessage}]
```

**Critical consideration:** DML before a screen is committed when the screen renders. If the user clicks the browser's Back button or closes the modal after the Apex Action runs but before the Result Screen, the Apex action has already executed. Place irreversible actions (email sends, external callouts) after the user's final confirmation, not between intermediate screens.

#### Passing Complex Data Between Flow Screens and Apex

The `@InvocableVariable` inner class pattern is the bridge between Flow and Apex. Each `@InvocableVariable` becomes a named input or output in Flow Builder.

For the Contact selection screen in the PO workflow:

- **Lookup component** on the screen: Returns a Contact record (or just the Contact Id). Best for large data volumes -- supports type-ahead server-side search.
- **Record Choice Set with radio buttons**: Loads all matching records into memory. Works for small bounded lists (5-20 contacts). Hits the 50,000 record SOQL limit at scale.
- **Get Records + Choice**: Query contacts with a Get Records element, then bind the results to a Choice component. Gives you explicit control over the filter criteria.

For the PO workflow, use a **Lookup component** filtered to Contacts related to the PO's Account. Pass the selected Contact Id to the Apex Action's Request wrapper alongside the `recordId`.

#### Displaying Apex Results Back to the User

After the Apex Action executes, the flow has access to all `@InvocableVariable` fields on the Result wrapper. Use these in the success/error screen:

- **Success screen**: Display text referencing result fields: "PDF sent successfully to {!result.recipientEmail}. Task {!result.taskSubject} created."
- **Error screen**: Display the error message: "The following error occurred: {!result.errorMessage}. Please try again or contact your administrator."
- **Conditional screens**: Use a Decision element on `{!result.isSuccess}` to route to the appropriate screen. Never show both screens -- always branch.

### Coexistence Anti-Patterns

These are the patterns that cause production incidents when flows and triggers coexist.

**Anti-Pattern 1: Duplicated Logic**

A before-save flow sets `Account.Region__c` based on `BillingState`. Separately, a before trigger also sets `Account.Region__c` based on `BillingState` using a slightly different mapping. When the business rule changes, someone updates the flow but not the trigger (or vice versa). Different records get different Region values depending on which code path was dominant.

**Fix:** One mechanism owns the field. Document the ownership in a wiki or comment: "Region__c is owned by the before-save flow `Account_Before_Insert_Set_Region`. The trigger handler does NOT touch this field."

**Anti-Pattern 2: Before-Save Flow + Before Trigger on the Same Field**

A before-save flow sets `Priority__c = "High"` based on amount thresholds. A before trigger also sets `Priority__c` based on account tier. The trigger runs at step 5, after the flow at step 4. The trigger always wins. An admin changes the flow threshold, tests it, sees the flow setting the correct value in debug, deploys -- and it does not work in production because the trigger overwrites it. The admin has no visibility into the Apex trigger.

**Fix:** Never have both a before-save flow and a before trigger set the same field. If both need input into the value, consolidate the logic in one place. If the field-setting logic is simple, put it in the flow and remove it from the trigger. If it is complex, put it in the trigger and remove the flow.

**Anti-Pattern 3: After-Save Flow + After Trigger Both Creating Child Records**

An after trigger creates a Task when a PO is approved. An after-save flow also creates a different Task when a PO is approved. Both fire on the same save. The user sees two Tasks. Worse, the after trigger's Task creation (step 10) and the after-save flow's Task creation (step 15) happen at different points in the transaction, so debugging which created which is confusing.

**Fix:** Consolidate child record creation into one mechanism. If the trigger handler already creates Tasks, do not add a flow that also creates Tasks. If you want admins to control the Task creation, move it out of the trigger and into the flow.

**Anti-Pattern 4: Adding a Flow to a Trigger-Heavy Object Without Audit**

An object has an Apex trigger framework with 6 handler methods, 3 workflow rules (legacy), and 2 process builders (legacy). A developer adds a new after-save flow because "flows are the modern way." The combined governor limit consumption (SOQL queries across all automation) exceeds 100, and the next Data Loader batch of 200 records fails with `System.LimitException: Too many SOQL queries: 101`.

**Fix:** Before adding any automation to an object, audit ALL existing automation using:
- Setup > Object Manager > [Object] > Triggers (lists Apex triggers)
- Setup > Object Manager > [Object] > Flow Triggers (lists record-triggered flows with execution order)
- Setup > Object Manager > [Object] > Workflow Rules (legacy)
- Flow Trigger Explorer (Setup > Flows > open any flow > Flow Trigger Explorer button)

Profile the total SOQL and DML consumption across all automation. Only add new automation if there is headroom.

**Anti-Pattern 5: Flow Updating the Triggering Record in After-Save When Before-Save Would Work**

An after-save flow updates `Status__c` on the triggering record. This triggers a full re-execution of the order of execution (steps 1-20), including re-firing all before-save flows and all Apex triggers. The CPU time doubles. If the entry conditions are not tight enough, the after-save flow fires a second time during the re-save.

**Fix:** If you only need to update fields on the triggering record, always use a before-save flow. Reserve after-save flows for cross-object work (creating child records, updating parent records, sending notifications).

## Code Examples

### Invocable Method with Clean Error Handling for Screen Flows

This is the complete pattern for an invocable method that returns errors to the flow instead of throwing exceptions. The flow can branch on `isSuccess` and display the error message on a screen.

```apex
public with sharing class ValidateAndProcessOrderAction {

    @InvocableMethod(
        label='Validate and Process Order'
        description='Validates order data, creates line items, and returns success or error details to the flow'
        category='Purchase Orders'
    )
    public static List<Result> execute(List<Request> requests) {
        List<Result> results = new List<Result>();

        // collect all record IDs upfront for bulk query
        Set<Id> orderIds = new Set<Id>();
        for (Request req : requests) {
            orderIds.add(req.orderId);
        }

        // single bulk query outside the loop
        Map<Id, Purchase_Order__c> orderMap = new Map<Id, Purchase_Order__c>([
            SELECT Id, Name, Status__c, Total_Amount__c, Account__c,
                   Account__r.Name, Account__r.Credit_Limit__c
            FROM Purchase_Order__c
            WHERE Id IN :orderIds
        ]);

        // process each request, catching per-record errors
        List<Purchase_Order__c> ordersToUpdate = new List<Purchase_Order__c>();

        for (Request req : requests) {
            Result res = new Result();

            try {
                Purchase_Order__c order = orderMap.get(req.orderId);

                // validate: order exists
                if (order == null) {
                    res.isSuccess = false;
                    res.errorMessage = 'Order not found: ' + req.orderId;
                    results.add(res);
                    continue;
                }

                // validate: order is in correct status
                if (order.Status__c != 'Draft') {
                    res.isSuccess = false;
                    res.errorMessage = 'Order ' + order.Name + ' is in status "'
                        + order.Status__c + '". Only Draft orders can be processed.';
                    results.add(res);
                    continue;
                }

                // validate: amount does not exceed credit limit
                if (order.Account__r.Credit_Limit__c != null
                    && order.Total_Amount__c > order.Account__r.Credit_Limit__c) {
                    res.isSuccess = false;
                    res.errorMessage = 'Order total ($' + order.Total_Amount__c.setScale(2)
                        + ') exceeds ' + order.Account__r.Name
                        + '\'s credit limit ($' + order.Account__r.Credit_Limit__c.setScale(2) + ').';
                    results.add(res);
                    continue;
                }

                // all validations passed -- stage the update
                Purchase_Order__c orderUpdate = new Purchase_Order__c();
                orderUpdate.Id = order.Id;
                orderUpdate.Status__c = 'Processing';
                orderUpdate.Processed_Date__c = Date.today();
                ordersToUpdate.add(orderUpdate);

                res.isSuccess = true;
                res.errorMessage = null;
                res.processedOrderName = order.Name;

            } catch (Exception ex) {
                // catch any unexpected exception per-record
                res.isSuccess = false;
                res.errorMessage = 'Unexpected error processing order: ' + ex.getMessage();
                System.debug('// error processing order ' + req.orderId + ': '
                    + ex.getMessage() + '\n' + ex.getStackTraceString());
            }

            results.add(res);
        }

        // bulk DML outside the loop
        if (!ordersToUpdate.isEmpty()) {
            Database.SaveResult[] saveResults = Database.update(ordersToUpdate, false);
            // note: partial-success DML -- some records may fail while others succeed
            // in a screen flow context this list typically has 1 element, but bulk-safe code is non-negotiable
            for (Integer i = 0; i < saveResults.size(); i++) {
                if (!saveResults[i].isSuccess()) {
                    System.debug('// DML failed for order: ' + saveResults[i].getErrors()[0].getMessage());
                }
            }
        }

        return results;
    }

    public class Request {
        @InvocableVariable(label='Order ID' description='The Purchase Order to validate and process' required=true)
        public Id orderId;
    }

    public class Result {
        @InvocableVariable(label='Success' description='Whether the order was processed successfully')
        public Boolean isSuccess;

        @InvocableVariable(label='Error Message' description='Details about why processing failed (null on success)')
        public String errorMessage;

        @InvocableVariable(label='Order Name' description='The Name of the processed order (null on failure)')
        public String processedOrderName;
    }
}
```

### Test Class Covering Error Return Paths

```apex
@IsTest
private class ValidateAndProcessOrderActionTest {

    @TestSetup
    static void setupTestData() {
        Account acc = new Account(
            Name = 'Test Corp',
            Credit_Limit__c = 50000.00
        );
        insert acc;

        // create a Draft order under the credit limit
        Purchase_Order__c draftOrder = new Purchase_Order__c(
            Name = 'PO-DRAFT',
            Account__c = acc.Id,
            Status__c = 'Draft',
            Total_Amount__c = 25000.00
        );

        // create an already-processed order
        Purchase_Order__c processedOrder = new Purchase_Order__c(
            Name = 'PO-PROCESSED',
            Account__c = acc.Id,
            Status__c = 'Processing',
            Total_Amount__c = 10000.00
        );

        // create an order that exceeds credit limit
        Purchase_Order__c overLimitOrder = new Purchase_Order__c(
            Name = 'PO-OVERLIMIT',
            Account__c = acc.Id,
            Status__c = 'Draft',
            Total_Amount__c = 75000.00
        );

        insert new List<Purchase_Order__c>{ draftOrder, processedOrder, overLimitOrder };
    }

    @IsTest
    static void testExecute_validDraftOrder_succeeds() {
        Purchase_Order__c po = [SELECT Id FROM Purchase_Order__c WHERE Name = 'PO-DRAFT' LIMIT 1];

        ValidateAndProcessOrderAction.Request req = new ValidateAndProcessOrderAction.Request();
        req.orderId = po.Id;

        Test.startTest();
        List<ValidateAndProcessOrderAction.Result> results =
            ValidateAndProcessOrderAction.execute(new List<ValidateAndProcessOrderAction.Request>{ req });
        Test.stopTest();

        System.assertEquals(1, results.size(), 'Should return exactly one result');
        System.assertEquals(true, results[0].isSuccess, 'Valid draft order should succeed');
        System.assertEquals(null, results[0].errorMessage, 'No error message on success');
        System.assertEquals('PO-DRAFT', results[0].processedOrderName, 'Should return order name');

        // verify the status was updated
        Purchase_Order__c updated = [SELECT Status__c FROM Purchase_Order__c WHERE Id = :po.Id];
        System.assertEquals('Processing', updated.Status__c, 'Status should be updated to Processing');
    }

    @IsTest
    static void testExecute_wrongStatus_returnsError() {
        Purchase_Order__c po = [SELECT Id FROM Purchase_Order__c WHERE Name = 'PO-PROCESSED' LIMIT 1];

        ValidateAndProcessOrderAction.Request req = new ValidateAndProcessOrderAction.Request();
        req.orderId = po.Id;

        Test.startTest();
        List<ValidateAndProcessOrderAction.Result> results =
            ValidateAndProcessOrderAction.execute(new List<ValidateAndProcessOrderAction.Request>{ req });
        Test.stopTest();

        System.assertEquals(false, results[0].isSuccess, 'Wrong-status order should fail');
        System.assert(results[0].errorMessage.contains('Processing'),
            'Error should mention current status: ' + results[0].errorMessage);
    }

    @IsTest
    static void testExecute_exceedsCreditLimit_returnsError() {
        Purchase_Order__c po = [SELECT Id FROM Purchase_Order__c WHERE Name = 'PO-OVERLIMIT' LIMIT 1];

        ValidateAndProcessOrderAction.Request req = new ValidateAndProcessOrderAction.Request();
        req.orderId = po.Id;

        Test.startTest();
        List<ValidateAndProcessOrderAction.Result> results =
            ValidateAndProcessOrderAction.execute(new List<ValidateAndProcessOrderAction.Request>{ req });
        Test.stopTest();

        System.assertEquals(false, results[0].isSuccess, 'Over-limit order should fail');
        System.assert(results[0].errorMessage.contains('credit limit'),
            'Error should mention credit limit: ' + results[0].errorMessage);

        // verify the status was NOT updated
        Purchase_Order__c notUpdated = [SELECT Status__c FROM Purchase_Order__c WHERE Id = :po.Id];
        System.assertEquals('Draft', notUpdated.Status__c, 'Status should remain Draft');
    }

    @IsTest
    static void testExecute_invalidOrderId_returnsError() {
        ValidateAndProcessOrderAction.Request req = new ValidateAndProcessOrderAction.Request();
        req.orderId = 'a000000000000AAAA'; // nonexistent

        Test.startTest();
        List<ValidateAndProcessOrderAction.Result> results =
            ValidateAndProcessOrderAction.execute(new List<ValidateAndProcessOrderAction.Request>{ req });
        Test.stopTest();

        System.assertEquals(false, results[0].isSuccess, 'Invalid order should fail');
        System.assert(results[0].errorMessage.contains('not found'),
            'Error should mention not found: ' + results[0].errorMessage);
    }

    @IsTest
    static void testExecute_bulkMixedResults() {
        // test a batch with one valid and one invalid request
        Purchase_Order__c validPo = [SELECT Id FROM Purchase_Order__c WHERE Name = 'PO-DRAFT' LIMIT 1];
        Purchase_Order__c invalidPo = [SELECT Id FROM Purchase_Order__c WHERE Name = 'PO-PROCESSED' LIMIT 1];

        List<ValidateAndProcessOrderAction.Request> requests = new List<ValidateAndProcessOrderAction.Request>();

        ValidateAndProcessOrderAction.Request req1 = new ValidateAndProcessOrderAction.Request();
        req1.orderId = validPo.Id;
        requests.add(req1);

        ValidateAndProcessOrderAction.Request req2 = new ValidateAndProcessOrderAction.Request();
        req2.orderId = invalidPo.Id;
        requests.add(req2);

        Test.startTest();
        List<ValidateAndProcessOrderAction.Result> results =
            ValidateAndProcessOrderAction.execute(requests);
        Test.stopTest();

        System.assertEquals(2, results.size(), 'Should return two results for two requests');
        System.assertEquals(true, results[0].isSuccess, 'First request (Draft) should succeed');
        System.assertEquals(false, results[1].isSuccess, 'Second request (Processing) should fail');
    }
}
```

### Record-Triggered Flow with Custom Error Element (Conceptual Flow XML)

This shows the structure of a before-save record-triggered flow that calls a lightweight Apex validation and uses the Custom Error element to block the save with a user-facing message. This is the pattern for using Apex logic in a record-triggered flow without letting exceptions propagate.

```xml
<!-- Flow: PO_Before_Update_Validate_Credit_Limit -->
<!-- Type: Record-Triggered, Before-Save -->
<!-- Object: Purchase_Order__c -->
<!-- Event: Update -->
<!-- Entry Condition: Status__c changed to "Submitted" -->

<!--
  Flow Structure:
  1. Start (entry condition: ISCHANGED(Status__c) AND Status__c = "Submitted")
  2. Apex Action: CheckCreditLimitAction (invocable method)
     - Input: orderId = {!$Record.Id}
     - Output: result (Result wrapper with isSuccess, errorMessage)
  3. Decision: Is result.isSuccess = true?
     - Yes → End (flow exits, save proceeds normally)
     - No  → Custom Error element
  4. Custom Error:
     - Error Message: {!result.errorMessage}
     - Display Location: Page-level (or field-level on Total_Amount__c)
     - Behavior: Rolls back the save, displays error to user on the record page
-->
```

The supporting Apex class for the above flow:

```apex
public with sharing class CheckCreditLimitAction {

    @InvocableMethod(
        label='Check Credit Limit'
        description='Validates that the order total does not exceed the account credit limit'
        category='Purchase Orders'
    )
    public static List<Result> execute(List<Request> requests) {
        List<Result> results = new List<Result>();

        // collect account IDs for bulk query
        Set<Id> orderIds = new Set<Id>();
        for (Request req : requests) {
            orderIds.add(req.orderId);
        }

        Map<Id, Purchase_Order__c> orderMap = new Map<Id, Purchase_Order__c>([
            SELECT Id, Name, Total_Amount__c, Account__r.Credit_Limit__c,
                   Account__r.Name
            FROM Purchase_Order__c
            WHERE Id IN :orderIds
        ]);

        for (Request req : requests) {
            Result res = new Result();

            try {
                Purchase_Order__c order = orderMap.get(req.orderId);

                if (order == null) {
                    res.isSuccess = false;
                    res.errorMessage = 'Order not found.';
                    results.add(res);
                    continue;
                }

                // check credit limit
                Decimal creditLimit = order.Account__r.Credit_Limit__c;
                if (creditLimit != null && order.Total_Amount__c > creditLimit) {
                    res.isSuccess = false;
                    res.errorMessage = 'Order total ($'
                        + order.Total_Amount__c.setScale(2)
                        + ') exceeds ' + order.Account__r.Name
                        + '\'s credit limit ($' + creditLimit.setScale(2) + '). '
                        + 'Reduce the order amount or request a credit limit increase.';
                } else {
                    res.isSuccess = true;
                    res.errorMessage = null;
                }

            } catch (Exception ex) {
                res.isSuccess = false;
                res.errorMessage = 'Validation error: ' + ex.getMessage();
            }

            results.add(res);
        }

        return results;
    }

    public class Request {
        @InvocableVariable(label='Order ID' required=true)
        public Id orderId;
    }

    public class Result {
        @InvocableVariable(label='Success')
        public Boolean isSuccess;

        @InvocableVariable(label='Error Message')
        public String errorMessage;
    }
}
```

### Automation Inventory Template

Use this template to document all automation on a high-density object before adding new flows or triggers. Without this inventory, you are guessing at governor limit headroom and field ownership.

```
## Automation Inventory: Purchase_Order__c

### Before-Save Flows
| Flow Name | Trigger Event | Fields Modified | SOQL Used | Notes |
|---|---|---|---|---|
| PO_Before_Insert_Defaults | Insert | Status__c, Region__c | 0 | Sets defaults only |
| PO_Before_Update_Validate | Update | (none -- validation only) | 1 (Get Account) | Routes to Custom Error on failure |

### Apex Triggers (via PurchaseOrderTriggerHandler)
| Handler Method | Event | Fields Modified | SOQL Used | DML Used | Notes |
|---|---|---|---|---|---|
| handleBeforeInsert | Before Insert | PO_Number__c | 1 (query counter) | 0 | Auto-number stamping |
| handleBeforeUpdate | Before Update | (none) | 0 | 0 | Validation only |
| handleAfterInsert | After Insert | (none) | 1 (query Account) | 1 (update Account) | Updates Account.Last_PO_Date__c |
| handleAfterUpdate | After Update | (none) | 2 | 1 | Creates approval Tasks |

### After-Save Flows
| Flow Name | Trigger Event | Records Created/Updated | SOQL Used | DML Used | Notes |
|---|---|---|---|---|---|
| PO_After_Update_Notify | Update (Status = Approved) | 0 | 0 | 0 | Sends email notification |

### Legacy Workflow Rules
| Rule Name | Event | Actions | Notes |
|---|---|---|---|
| PO_Set_Legacy_Flag | Update | Field update: Legacy__c = true | MIGRATE TO FLOW -- causes trigger re-fire |

### Totals Per Insert Transaction
| Resource | Consumed | Limit | Headroom |
|---|---|---|---|
| SOQL Queries | ~4 | 100 | 96 |
| DML Statements | ~2 | 150 | 148 |
| CPU Time (est.) | ~200ms | 10,000ms | 9,800ms |

### Totals Per Update Transaction (worst case)
| Resource | Consumed | Limit | Headroom |
|---|---|---|---|
| SOQL Queries | ~6 (+2 for workflow re-fire) | 100 | 92 |
| DML Statements | ~3 | 150 | 147 |
| CPU Time (est.) | ~500ms | 10,000ms | 9,500ms |
```

## Common Mistakes

1. **Duplicating business logic in both a flow and a trigger on the same object** -- A flow sets `Priority__c` based on amount thresholds. The trigger also sets `Priority__c` based on a different (or same) rule. When the business rule changes, one gets updated and the other does not. Different records get different values depending on timing and execution path. Fix: Assign field ownership to exactly one mechanism. Document which mechanism owns which field. Never let two automations set the same field.

2. **Adding a record-triggered flow to an object without auditing existing automation** -- The developer adds an after-save flow that uses 30 SOQL queries. The existing Apex trigger already uses 65. Combined: 95. On most saves this works. On a Data Loader batch with cross-object triggers cascading, it hits 101 and crashes. Fix: Before adding any automation, audit all triggers, flows, workflow rules, and process builders on the object. Profile total SOQL and DML usage. Only add automation if headroom exists.

3. **Expecting a before-save flow to "win" over a before trigger on the same field** -- The flow runs at step 4, the trigger at step 5. The trigger always overwrites the flow. An admin changes the flow, tests it in isolation (no trigger in the sandbox), deploys to production where the trigger exists, and the flow's value never sticks. Fix: Never have both a before-save flow and a before trigger modify the same field. Remove the redundant setter from one mechanism.

4. **Throwing unhandled exceptions from @InvocableMethod in a screen flow** -- The user sees "An unhandled fault has occurred in flow [FlowName]" with no context. The entire transaction rolls back, including any DML the flow performed before the Apex call. Fix: Catch all exceptions inside the invocable method. Return error information in the Result wrapper. Let the flow branch on `isSuccess` and show a friendly error screen.

5. **Using AuraHandledException in an @InvocableMethod expecting LWC-like behavior** -- The developer assumes `AuraHandledException` will strip the stack trace and send a clean message to the flow. It does not. Flow treats all exceptions identically. The flow hits the fault path and shows a generic error. Fix: Never throw `AuraHandledException` from invocable methods. Return errors in the Result wrapper for flow callers. Reserve `AuraHandledException` for `@AuraEnabled` methods called by LWC/Aura.

6. **Not wiring a fault connector on the Apex Action element in a screen flow** -- The invocable method handles errors internally, so the developer skips the fault connector. Then a `LimitException` (which cannot be caught in Apex) fires, and the flow crashes with an unhandled fault. Fix: Always wire a fault connector on every Apex Action element. Route it to an error screen that displays `{!$Flow.FaultMessage}`. This catches the uncatchable.

7. **After-save flow and after trigger both creating child records for the same event** -- An after trigger creates a Task when Status changes to "Approved." An after-save flow also creates a notification record when Status changes to "Approved." Both fire on the same save. The user sees duplicate downstream effects. Worse, the trigger creates records at step 10 and the flow at step 15, making the debug log confusing. Fix: Consolidate child record creation into one mechanism. If the trigger already handles this event, extend the trigger handler -- do not add a parallel flow.

8. **Letting a workflow field update re-fire triggers without recursion guards** -- A legacy workflow rule has a field update that changes `Legacy_Flag__c`. This re-fires the before and after Apex triggers. The trigger logic runs twice: once for the user's save, once for the workflow field update. If the trigger creates a child record, two child records are created. Fix: Use static recursion guards (a `static Set<Id>` of processed record IDs) in the trigger handler to ensure each record is processed exactly once per context. Alternatively, migrate the workflow rule to a before-save flow and eliminate the re-fire entirely.

9. **Building a record-triggered flow that updates the triggering record in after-save when a before-save flow would work** -- The after-save flow uses an Update Records element to set `Processed_Date__c` on the triggering record. This triggers a full re-execution of the order of execution (steps 1-20), doubling the CPU time and DML consumption. If entry conditions are loose, the flow fires again on the re-save. Fix: Use a before-save flow to set fields on the triggering record. Reserve after-save flows for cross-object DML and notifications.

10. **Not documenting the automation inventory on high-density objects** -- Five developers over three years have added automation to the Account object. Nobody remembers all of it. A new automation causes a CPU timeout in production. Debugging requires reading every trigger, every flow, every workflow rule, and every process builder. Fix: Maintain an automation inventory (see the template above) for every object with 3+ automations. Update it whenever automation is added or modified. Review it during code reviews and before every deployment that touches the object.

## See Also

- [Invocable Methods -- Flow-to-Apex Bridge](../apex-fundamentals/invocable-methods.md)
- [Solution Architecture Patterns](../solution-architecture/solution-architecture-patterns.md)
- [Quick Actions & Screen Flow Integration](../quick-actions/quick-actions-and-screen-flows.md)
- Salesforce Architect Decision Guide: [Record-Triggered Automation](https://architect.salesforce.com/docs/architect/decision-guides/guide/record-triggered)
- Salesforce Help: [Custom Error Element](https://help.salesforce.com/s/articleView?id=platform.flow_ref_elements_custom_error.htm&language=en_US&type=5)
- Salesforce Help: [Triggers and Order of Execution](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_triggers_order_of_execution.htm)
