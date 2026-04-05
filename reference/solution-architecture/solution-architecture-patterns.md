# Solution Architecture Patterns

> **When to read this:** You are designing an Apex solution from scratch, choosing between Flow and Apex, structuring classes for a multi-object transaction, or setting up naming conventions for a new project.

## Rules

- When deciding between Flow and Apex, always evaluate **automation density** on the object first because stacking multiple record-triggered flows on a high-density object causes governor limit failures and unpredictable execution order.
- When building any record-triggered automation, always use **one trigger per object** delegating to a handler class because multiple triggers on the same object have no guaranteed execution order.
- When an Apex class does more than one DML operation across related objects, always use the **Unit of Work pattern** (or at minimum a manual SavePoint) because partial commits corrupt data.
- When a Screen Flow needs to do anything beyond simple record CRUD, always call an **@InvocableMethod** in Apex because Apex gives you transaction control, bulkification, and testable logic that Flow cannot.
- Never hardcode IDs, email addresses, URLs, thresholds, or feature flags in Apex because those values differ between sandboxes and production. Use **Custom Metadata Types** instead.
- Never put business logic directly in a trigger body because it cannot be unit-tested in isolation, cannot be called from other contexts, and becomes unmaintainable.
- When multiple subscribers need to react to a single event without blocking the original transaction, always use **Platform Events** because synchronous chaining creates governor limit pressure and tight coupling.
- When querying the same object from multiple service methods, always centralize those queries in a **Selector class** because scattered inline SOQL leads to inconsistent field sets and duplicated queries.
- Never create a custom object when a standard object (Task, Event, Case, etc.) already covers the use case because standard objects integrate with reporting, activity timelines, and platform features for free.
- When building a Quick Action that launches a user-facing process, always use the **Quick Action -> Screen Flow -> Invocable Apex** pattern because it gives admins control over the UI layer while developers own the transaction logic.

## How It Works

### The Decision Framework: Apex vs Flow vs Flow + Apex

The right tool depends on three factors: **complexity**, **automation density**, and **who maintains it**.

**Use Flow alone** when the automation is straightforward record CRUD (create, update, delete), the object has low automation density (fewer than 3 automations firing on the same event), the logic is under ~50 nodes, and an admin should be able to modify it without developer involvement. Examples: auto-populating fields on record creation, sending a notification when a status changes, simple approval routing.

**Use Apex alone** when the logic involves complex data transformations, external callouts, PDF generation, email with attachments, batch/scheduled processing, or any scenario requiring explicit transaction control. Also use Apex when the object already has high automation density and adding another flow risks hitting CPU or DML limits. Examples: generating and emailing PDFs, complex pricing calculations, integration endpoints.

**Use Flow + Apex (the hybrid)** when users need a guided screen experience but the backend logic is too complex for Flow. The Screen Flow owns the UI (screens, choices, navigation), and an `@InvocableMethod` class owns the transaction. This is the most common real-world pattern for user-initiated actions. Examples: the Purchase Order email workflow, guided case escalation, multi-step data entry with validation.

Salesforce's own Record-Triggered Automation Decision Guide now treats **automation density** as the primary indicator. As density increases on an object, the risk of re-entering the save order, exhausting CPU limits, and fragmenting logic grows. High-density objects should consolidate automation into Apex triggers with handler classes.

### The Glue Pattern: Quick Action -> Screen Flow -> Invocable Apex

This is the standard pattern for user-initiated processes that need both a UI and complex backend logic.

**Quick Action** — A button on the record page. Type is "Flow" (not LWC, not Visualforce). It passes `recordId` automatically to the flow.

**Screen Flow** — Presents screens to the user (select a contact, confirm details, show success/error). The flow handles the user experience but delegates all heavy lifting to Apex.

**Invocable Apex** — An `@InvocableMethod` class that receives structured input from the flow, executes the business logic (queries, DML, PDF generation, emails, logging), and returns structured results.

The boundaries are clean: admins own the flow screens, developers own the Apex logic, and the Quick Action is the entry point on the page layout.

### Service Layer Pattern

The Service layer encapsulates business processes into reusable, testable methods. Every major business operation (sending an email, approving a record, generating a PDF) lives in a Service class.

```
AccountService.cls          — mergeAccounts(), calculateHealthScore()
OpportunityService.cls      — closeWon(), applyDiscount()
PurchaseOrderService.cls    — sendPurchaseOrderEmail(), stampDigitalAuth()
```

**Key rules:**
- Service methods accept **collections** (List, Set, Map), never single records, to enforce bulkification from the start.
- Service methods are `public static` so they can be called from triggers, flows, REST endpoints, batch jobs, or other services.
- Service methods own the transaction boundary — they are where you create a SavePoint or Unit of Work instance.
- Service methods use `with sharing` by default. Only switch to `without sharing` when explicitly justified and documented.

### Selector Pattern

The Selector layer centralizes all SOQL for a given object. Every query for `Purchase_Order__c` goes through `PurchaseOrderSelector`, regardless of which service or trigger calls it.

```apex
public with sharing class PurchaseOrderSelector {

    // queries PO with all standard fields needed by the service layer
    // input: poIds - set of Purchase_Order__c IDs to query
    // output: list of Purchase_Order__c records with line items
    public static List<Purchase_Order__c> selectByIdWithLineItems(Set<Id> poIds) {
        return [
            SELECT Id, Name, Status__c, Authorizer_Name__c,
                   Digital_Authorization__c, Total_Amount__c,
                   (SELECT Id, Name, Quantity__c, Unit_Price__c, Description__c
                    FROM Purchase_Order_Line_Items__r
                    ORDER BY Name)
            FROM Purchase_Order__c
            WHERE Id IN :poIds
        ];
    }
}
```

**Why this matters:**
- Every consumer queries the same fields, preventing runtime errors from missing fields.
- Adding a new field to the query happens in one place.
- You can enforce FLS and sharing checks in the selector.
- Unit tests can verify query behavior independently.

### Domain Layer Pattern

The Domain layer holds business logic that operates on a collection of SObject records. Think of it as "what does this record know how to do?" — validation, defaulting, calculation, state transitions.

```apex
public with sharing class PurchaseOrders {

    private List<Purchase_Order__c> records;

    public PurchaseOrders(List<Purchase_Order__c> records) {
        this.records = records;
    }

    // stamps digital authorization on POs that were successfully emailed
    // input: none (operates on the internal record list)
    // output: none (modifies records in place for later DML)
    public void stampDigitalAuthorization() {
        for (Purchase_Order__c po : this.records) {
            if (String.isNotBlank(po.Authorizer_Name__c)) {
                po.Digital_Authorization__c = 'Digital Authorization from ' + po.Authorizer_Name__c;
            }
        }
    }
}
```

**When to use it:** When the same business rule applies across multiple entry points (trigger, service, batch). Instead of duplicating the logic, put it in the Domain class and call it from wherever.

### Unit of Work Pattern

The Unit of Work aggregates all DML operations (inserts, updates, deletes) and commits them in a single transaction with automatic rollback on failure.

In the fflib implementation, you register records as new, dirty (updated), or deleted throughout your service method, then call `commitWork()` once at the end. The Unit of Work handles parent-child ID resolution (inserting a parent, grabbing its ID, and setting it on the child before inserting the child).

**Simplified version without fflib:**

```apex
public with sharing class PurchaseOrderService {

    // sends PO email, logs the action, and stamps digital auth in one transaction
    // input: poId - the Purchase Order record ID, contactId - the recipient
    // output: none (throws exception on failure, rolling back all DML)
    public static void sendPurchaseOrderEmail(Id poId, Id contactId) {
        Savepoint sp = Database.setSavepoint();
        try {
            // all the work happens here (query, generate PDF, send email, log, update PO)
            // ...

            // commit: insert the log record
            insert emailLog;

            // commit: update the PO with digital auth stamp
            update po;

        } catch (Exception e) {
            Database.rollback(sp);
            throw new PurchaseOrderException('Failed to send PO email: ' + e.getMessage());
        }
    }
}
```

**Rule:** One Unit of Work (or SavePoint) per service method call. The service method IS the transaction boundary.

### Record-Triggered Flow + Apex Trigger Coexistence

Salesforce's order of execution determines when triggers and flows fire. The critical sequence is:

1. System validation rules
2. **Before triggers** fire (Apex)
3. Custom validation rules
4. **After triggers** fire (Apex)
5. Assignment rules, auto-response rules
6. Workflow rules (legacy)
7. **Record-triggered flows** fire (before-save and after-save)
8. If workflow/flow field updates occur, before and after triggers re-fire (once)

**Coexistence rules:**
- Before-save flows run AFTER Apex before triggers. After-save flows run AFTER Apex after triggers.
- You can set trigger order (1-2000) on flows to control which flow runs first when multiple flows exist on the same object.
- You CANNOT make a flow run before an Apex trigger on the same event — Apex always goes first.
- If both a trigger and a flow update the same field, the flow wins because it runs later.
- Avoid having both a trigger and a flow doing DML on the same child object in the same save — the re-firing of triggers creates confusion and potential recursion.

**Best practice:** Pick one mechanism per object. If you have an Apex trigger on Account, keep all Account automation in that trigger's handler. Do not add record-triggered flows on Account alongside it unless you have a deliberate reason and document the interaction.

### Custom Metadata Types for Configuration

Custom Metadata Types (`__mdt`) store configuration values that are deployable, cacheable, and editable by admins without code changes.

```apex
// querying custom metadata does NOT count against SOQL governor limits
Email_Settings__mdt settings = Email_Settings__mdt.getInstance('Purchase_Order_Email');
String orgWideAddress = settings.Org_Wide_Email_Address__c;
String replyTo = settings.Reply_To_Address__c;
Boolean enableEmailSend = settings.Is_Active__c;
```

**Use Custom Metadata for:**
- Email addresses, API endpoints, and external URLs
- Feature toggles (enable/disable functionality without deployment)
- Threshold values (approval limits, batch sizes, retry counts)
- Mapping tables (status value X maps to picklist value Y)
- Any value that differs between sandboxes and production

**Never use Custom Settings for new development** — Custom Metadata Types replaced them. Custom Settings have no deployment support and are not metadata-aware.

### Platform Events for Async Decoupling

Platform Events fire-and-forget: the publisher commits the event and moves on. Subscribers process independently in their own transaction.

**When to use Platform Events:**
- Logging that should not block the main transaction (publish a log event; a subscriber writes it to a custom object).
- Notifying external systems without making the user wait for a callout.
- Decoupling complex post-processing from the save transaction (e.g., after an order is placed, a platform event triggers fulfillment processing).

**When NOT to use Platform Events:**
- When the subscriber's work must succeed or fail with the main transaction. Platform events are asynchronous — you cannot roll them back after commit.
- Simple same-transaction DML. Just do it in the trigger/service.

```apex
// publishing a platform event from Apex
PO_Email_Sent__e event = new PO_Email_Sent__e(
    Purchase_Order_Id__c = poId,
    Sent_To_Email__c = contactEmail,
    Sent_Date__c = Datetime.now()
);
EventBus.publish(event);
// the event is published regardless of transaction rollback unless you use
// Database.setSavepoint() before the publish — events bypass savepoints
```

### Error Handling Strategy

Errors need to be caught, surfaced, and logged in the right places.

| Layer | Strategy |
|---|---|
| **Trigger** | Never swallow exceptions. Let them bubble up so the user sees the error and the DML rolls back. |
| **Service** | Catch exceptions at the service boundary. Log them. Re-throw a custom exception with a user-friendly message. |
| **Invocable** | Catch exceptions, return error details in the Response inner class so the Flow can branch on success/failure. Never throw unhandled exceptions from an Invocable — the Flow error screen is unhelpful. |
| **Flow** | Add a Fault Path on every Action element. Use the `{!$Flow.FaultMessage}` variable to display or log the error. |
| **Batch** | Catch exceptions per-chunk in the `execute` method. Accumulate errors. Report them in the `finish` method. |

**Custom exception classes** make error handling readable:

```apex
public class PurchaseOrderException extends Exception {}
```

Throw domain-specific exceptions (`PurchaseOrderException`, `EmailSendException`) rather than generic `Exception` so callers can catch specific failure types.

## Code Examples

### Service Layer + Invocable Method Integration

```apex
public with sharing class SendPurchaseOrderEmail {

    // invocable method called by the Send PO Email screen flow
    // input: requests - list of Request objects containing poId and contactId
    // output: list of Response objects with success flag and message
    @InvocableMethod(
        label='Send Purchase Order Email'
        description='Generates a PDF of the Purchase Order and emails it to the selected Contact'
        category='Purchase Orders'
    )
    public static List<Response> invoke(List<Request> requests) {
        List<Response> responses = new List<Response>();

        for (Request req : requests) {
            Response res = new Response();
            try {
                // delegate to the service layer
                PurchaseOrderService.sendPurchaseOrderEmail(req.poId, req.contactId);
                res.isSuccess = true;
                res.message = 'Purchase Order emailed successfully.';
            } catch (Exception e) {
                res.isSuccess = false;
                res.message = 'Error: ' + e.getMessage();
                System.debug(LoggingLevel.ERROR, 'SendPurchaseOrderEmail failed: ' + e.getMessage()
                    + ' | Stack: ' + e.getStackTraceString());
            }
            responses.add(res);
        }

        return responses;
    }

    public class Request {
        @InvocableVariable(label='Purchase Order ID' required=true)
        public Id poId;

        @InvocableVariable(label='Contact ID' required=true)
        public Id contactId;
    }

    public class Response {
        @InvocableVariable(label='Success')
        public Boolean isSuccess;

        @InvocableVariable(label='Message')
        public String message;
    }
}
```

### Trigger + Handler Pattern

```apex
// one trigger per object — all events delegate to the handler
trigger PurchaseOrderTrigger on Purchase_Order__c (
    before insert, before update, before delete,
    after insert, after update, after delete, after undelete
) {
    PurchaseOrderTriggerHandler handler = new PurchaseOrderTriggerHandler();
    handler.run();
}
```

```apex
public with sharing class PurchaseOrderTriggerHandler {

    // static flag to prevent recursion
    private static Boolean isExecuting = false;

    // routes trigger context to the appropriate method
    // input: none (reads from Trigger context variables)
    // output: none
    public void run() {
        if (isExecuting) {
            return;
        }
        isExecuting = true;

        try {
            if (Trigger.isBefore) {
                if (Trigger.isInsert) {
                    handleBeforeInsert(Trigger.new);
                } else if (Trigger.isUpdate) {
                    handleBeforeUpdate(Trigger.new, Trigger.oldMap);
                }
            } else if (Trigger.isAfter) {
                if (Trigger.isInsert) {
                    handleAfterInsert(Trigger.new);
                } else if (Trigger.isUpdate) {
                    handleAfterUpdate(Trigger.new, Trigger.oldMap);
                }
            }
        } finally {
            isExecuting = false;
        }
    }

    // handles before insert logic — field defaulting, validation
    // input: newRecords - list of Purchase_Order__c being inserted
    // output: none (modifies records in place)
    private void handleBeforeInsert(List<Purchase_Order__c> newRecords) {
        PurchaseOrders domain = new PurchaseOrders(newRecords);
        domain.applyDefaults();
        domain.validate();
    }

    // handles before update logic — validation, conditional field stamps
    // input: newRecords - list of updated records, oldMap - map of previous values
    // output: none (modifies records in place)
    private void handleBeforeUpdate(List<Purchase_Order__c> newRecords,
                                     Map<Id, Purchase_Order__c> oldMap) {
        PurchaseOrders domain = new PurchaseOrders(newRecords);
        domain.validate();
    }

    private void handleAfterInsert(List<Purchase_Order__c> newRecords) {
        // after-insert logic here (e.g., create related records)
    }

    private void handleAfterUpdate(List<Purchase_Order__c> newRecords,
                                    Map<Id, Purchase_Order__c> oldMap) {
        // after-update logic here (e.g., fire platform events)
    }
}
```

### Custom Metadata Configuration Pattern

```apex
public with sharing class AppConfig {

    // lazily loaded config cache — avoids repeated SOQL
    private static Map<String, App_Config__mdt> configCache;

    // retrieves a configuration value by developer name
    // input: developerName - the DeveloperName of the config record
    // output: the App_Config__mdt record, or null if not found
    public static App_Config__mdt get(String developerName) {
        if (configCache == null) {
            configCache = App_Config__mdt.getAll();
        }
        return configCache.get(developerName);
    }

    // retrieves a string config value with a default fallback
    // input: developerName - the config key, defaultValue - fallback if not found
    // output: the config value or the default
    public static String getString(String developerName, String defaultValue) {
        App_Config__mdt config = get(developerName);
        return (config != null && String.isNotBlank(config.Value__c))
            ? config.Value__c
            : defaultValue;
    }
}
```

## Naming Conventions

### Apex Classes

| Type | Convention | Example |
|---|---|---|
| Service class | `{Object}Service` | `PurchaseOrderService` |
| Selector class | `{Object}Selector` | `PurchaseOrderSelector` |
| Domain class | `{Object}s` (plural) | `PurchaseOrders` |
| Trigger handler | `{Object}TriggerHandler` | `PurchaseOrderTriggerHandler` |
| Invocable | `{ActionVerb}{Object}{Noun}` | `SendPurchaseOrderEmail` |
| Test class | `{ClassName}Test` | `SendPurchaseOrderEmailTest` |
| Batch class | `{Description}Batch` | `PurgeOldEmailLogsBatch` |
| Schedulable | `{Description}Schedule` | `DailyPOReportSchedule` |
| Queueable | `{Description}Queueable` | `SyncPurchaseOrderQueueable` |
| Controller (VF) | `{PageName}Controller` | `PurchaseOrderPDFController` |
| Exception | `{Domain}Exception` | `PurchaseOrderException` |
| Custom Metadata wrapper | `{ConfigArea}Config` | `AppConfig`, `EmailConfig` |

### Triggers

| Convention | Example |
|---|---|
| `{Object}Trigger` (one per object) | `PurchaseOrderTrigger` |

### Flows

| Type | Convention | Example |
|---|---|---|
| Screen Flow | `{Object}_{Action_Description}` | `Send_Purchase_Order_Email` |
| Record-Triggered (Before) | `{Object}_Before_{Event}_{Description}` | `PO_Before_Update_Validate_Status` |
| Record-Triggered (After) | `{Object}_After_{Event}_{Description}` | `PO_After_Insert_Create_Line_Items` |
| Scheduled Flow | `{Object}_Scheduled_{Description}` | `PO_Scheduled_Expiration_Check` |
| Autolaunched (subflow) | `{Object}_Sub_{Description}` | `PO_Sub_Calculate_Totals` |

### Quick Actions

| Convention | Example |
|---|---|
| `{Object}.{Verb}_{Noun}` | `Purchase_Order__c.Send_PO_Email` |

### Custom Metadata Types

| Convention | Example |
|---|---|
| `{Feature_Area}_Settings__mdt` | `Email_Settings__mdt` |
| `{Feature_Area}_Config__mdt` | `App_Config__mdt` |

### Methods and Variables

- Methods: `camelCase` verbs — `sendEmail()`, `calculateTotal()`, `validateStatus()`
- Variables: `camelCase` nouns — `purchaseOrder`, `contactEmail`, `lineItems`
- Constants: `UPPER_SNAKE_CASE` — `MAX_RETRY_COUNT`, `DEFAULT_BATCH_SIZE`
- Boolean variables: prefix with `is`, `has`, or `should` — `isSuccess`, `hasLineItems`, `shouldSendEmail`

## When to Create Custom Objects vs Custom Fields vs Standard Objects

**Use a standard object** when Salesforce provides one that fits your use case. Task, Event, Case, Lead, Opportunity, and Contact cover most business scenarios. Standard objects integrate with activity timelines, reporting, Einstein, and platform features automatically. You cannot replicate that integration with custom objects.

**Add custom fields to a standard object** when you need to track additional data points that extend the standard object's purpose. Example: adding `PO_Sent_Date__c` and `PO_Recipient__c` to a Task for email logging — the Task still represents an activity, you are just enriching it.

**Create a custom object** only when no standard object covers the concept AND the data requires its own relationships, page layouts, sharing model, and reporting. Example: `Purchase_Order__c` is a custom object because Salesforce has no standard purchase order object. `Purchase_Order_Line_Item__c` is a custom object because it represents a child record with its own fields (quantity, unit price, description) that would not fit on the parent.

**Decision checklist:**
1. Does a standard object exist for this concept? If yes, use it.
2. Can I add fields to an existing standard or custom object? If yes, do that.
3. Does this data need its own page layout, sharing rules, validation rules, and reports? If yes, create a custom object.
4. Will this custom object have more than 10 fields? If not, reconsider whether a few custom fields on an existing object would suffice.

## Common Mistakes

1. **Putting business logic in the trigger body** — Developers write validation and DML directly in the trigger file. Fix: Keep the trigger body to 1-2 lines that delegate to a handler class. The trigger is a routing mechanism, not a logic container.

2. **Multiple triggers on the same object** — There is no guaranteed execution order for multiple triggers on the same object and event. Fix: One trigger per object, one handler class per trigger, dispatch to domain/service classes from the handler.

3. **Hardcoding IDs and email addresses** — Record IDs differ between sandboxes and production. Hardcoded emails break in sandboxes. Fix: Use Custom Metadata Types for all environment-specific values. Query record types by DeveloperName, never by ID.

4. **Choosing Flow when automation density is already high** — Adding a 5th record-triggered flow to an object that already has 4 flows and a trigger causes CPU timeouts. Fix: Consolidate automation into the Apex trigger handler when density exceeds 3 automations on the same event.

5. **Throwing unhandled exceptions from @InvocableMethod** — The Flow fault screen shows a generic error message that is useless to users. Fix: Wrap all Invocable logic in try-catch. Return error details in a Response inner class so the Flow can display a meaningful message or branch on success/failure.

6. **Creating custom objects for data that belongs on standard objects** — Creating `Email_Log__c` when a Task with a few custom fields would suffice. You lose activity timeline integration, standard reports, and Einstein insights. Fix: Evaluate standard objects first. Only create custom objects when the data truly needs its own entity.

7. **Skipping the service layer and calling DML from controllers/invocables directly** — This leads to duplicated logic when the same operation is needed from a trigger, a batch job, and a REST endpoint. Fix: All business logic goes in service classes. Controllers, invocables, triggers, and batch classes are thin callers of service methods.

8. **Not using SavePoint or Unit of Work for multi-step transactions** — If step 3 of 5 fails, steps 1 and 2 are already committed and you have corrupted data. Fix: Always wrap multi-step operations in a SavePoint (or use fflib Unit of Work). Roll back on any failure.

9. **Using Custom Settings instead of Custom Metadata Types** — Custom Settings cannot be deployed with metadata, are not version-controlled, and lack the deployment capabilities of Custom Metadata. Fix: Use Custom Metadata Types for all new configuration. Migrate existing Custom Settings when practical.

10. **Ignoring Flow + Trigger interaction on the same object** — A flow updates a field, which re-fires the trigger, which updates another field, which re-fires the flow. Fix: Document which mechanism owns which fields. Use static Boolean flags in trigger handlers to prevent recursion. Never have both a flow and a trigger updating the same field.

## See Also

- [Purchase Order PDF Workflow](./purchase-order-pdf-workflow.md)
- [Visualforce PDF](../visualforce-pdf/) — PDF generation patterns
- [Email Services](../email-services/) — Email with attachment patterns
- [Quick Actions](../quick-actions/) — Quick action configuration
- [Flows and Automation](../flows-and-automation/) — Flow + Apex interaction patterns
