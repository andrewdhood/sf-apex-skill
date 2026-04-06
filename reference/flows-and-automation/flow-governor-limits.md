# Flow Governor Limits for Apex Developers

> **When to read this:** When building or reviewing a flow that performs DML or queries, when a record-triggered flow coexists with Apex triggers on the same object, when designing for bulk data loads, or when troubleshooting `System.LimitException` errors in transactions that involve both flows and Apex.

## Rules

- Never place a Get Records, Create Records, Update Records, or Delete Records element inside a loop -- each iteration consumes a separate SOQL query or DML statement, and a 200-record batch will blow past the 100 SOQL / 150 DML limits immediately.
- Always remember that flows and Apex triggers on the same object share the same transaction and the same governor limits pool -- a record-triggered flow and an Apex trigger that both fire on Account insert share the same 100 SOQL queries and 150 DML statements. If your trigger uses 60 SOQL queries, your flow has 40 left.
- When updating only the triggering record's fields, always use a before-save flow because it modifies the record in memory without consuming a DML statement and runs approximately 10x faster than an equivalent after-save flow.
- Always use the Transform element (GA Spring '24) instead of a Loop + Assignment + collection variable pattern when you need to reshape one collection into another -- it executes as a single operation with no per-element overhead, replaces dozens of flow elements with one, and is easier to maintain.
- Never mix DML on setup objects (User, Group, GroupMember, QueueSobject, UserRole, PermissionSet) with DML on non-setup objects (Account, Contact, Opportunity, custom objects) in the same transaction -- this throws `MIXED_DML_OPERATION` whether the DML originates in Apex, a flow, or a combination of both.
- Always add fault paths to every DML element and every Apex Action element in a flow because without a fault path, an error rolls back the entire transaction and the user sees a generic system error.
- When a flow approaches governor limits, use a Screen element (screen flows) or an Asynchronous Path (record-triggered flows) to break the transaction boundary and reset limits.
- Always specify "Only the first record" on Get Records elements when you need exactly one record -- retrieving excess records wastes the 50,000 record retrieval limit that is shared across the entire transaction (including your Apex code).
- Never assume a flow that works with 1 record will work in bulk -- record-triggered flows process up to 200 records per batch, and a single DML element inside a loop becomes 200 DML operations during a data load.
- When profiling governor limit consumption on a shared object, always account for ALL automation in the transaction -- before-save flows (step 4), before triggers (step 5), after triggers (step 10), workflow rules (step 13), and after-save flows (step 15) all draw from the same pool.

## How It Works

### Shared Transaction: The Critical Concept for Apex Developers

This is the single most important thing Apex developers need to understand about flow governor limits: **flows do not get their own governor limits**. Every SOQL query, DML statement, callout, and CPU cycle consumed by a flow counts against the exact same per-transaction limits that your Apex triggers and classes consume.

When a user updates an Account record and the org has a before-save flow, a before trigger, an after trigger, and an after-save flow on Account, all four pieces of automation execute in one transaction. If the before-save flow uses 20 SOQL queries, the before trigger uses 30, the after trigger uses 25, and the after-save flow uses 20, the transaction total is 95 out of 100. One more query from anywhere in the chain and the transaction fails with `System.LimitException: Too many SOQL queries: 101`.

This shared-transaction reality means that Apex developers cannot design triggers in isolation. You must know what flows exist on the same object and factor their consumption into your headroom calculations. Conversely, admins building flows need to know how much headroom the Apex triggers have left. The automation inventory pattern (documented in [flow-apex-coexistence.md](./flow-apex-coexistence.md)) is essential for high-density objects.

### How Flow Elements Consume Limits

Each **Get Records** element executes one SOQL query (counts against the 100 synchronous / 200 asynchronous SOQL limit). Each **Create Records**, **Update Records**, and **Delete Records** element executes one DML statement (counts against the 150 DML limit). When these elements are inside a loop, they execute once per iteration -- a loop over 50 records with a Get Records inside it uses 50 of your 100 SOQL queries.

Salesforce provides automatic bulkification for record-triggered flows: when 200 records fire the same flow, Salesforce attempts to batch the DML and SOQL operations into single statements. However, this automatic bulkification breaks down when DML or SOQL elements are placed inside loops, inside decision branches that diverge per record, or in subflows called from loops. Once bulkification breaks, each record is processed individually and limits are consumed per-record.

### The Transform Element: Modern Replacement for Loop + DML

The Transform element (GA in Spring '24, API v60.0+) is the declarative equivalent of a SOQL `SELECT ... FROM ChildObject WHERE ParentId IN :parentIds` followed by a Map-based reshape in Apex. It takes a source collection and produces a target collection using field mappings, without requiring a Loop element, Assignment elements, or intermediate collection variables.

**When to use Transform instead of Loop + Assignment:**
- Mapping fields from one collection of records to another (e.g., creating Contacts from a collection of Leads)
- Reshaping data between objects without per-record conditional logic
- Any scenario where you previously used Loop → Assignment → Add to Collection → DML After Loop

**When Transform does NOT replace Loop + Assignment:**
- When you need per-record conditional logic (Decision elements inside the loop)
- When you need to call Apex Actions per record
- When you need to perform lookups or queries per record (though this is itself an anti-pattern)

Transform executes as a single operation regardless of collection size. It does not consume SOQL or DML by itself -- it only reshapes data in memory. You still need a Create/Update/Delete Records element after the Transform to persist the changes, but that single DML element processes the entire collection.

```
Before (Loop Pattern — 3+ elements, fragile at scale):
  Get Records → Loop → Assignment (set fields) → Assignment (add to collection) → Update Records

After (Transform Pattern — 2 elements, clean):
  Get Records → Transform (source → target with field mappings) → Update Records
```

### Transaction Boundaries

A transaction is a set of operations that execute as a single unit -- either all succeed or all roll back. Understanding where transaction boundaries fall is critical because governor limits reset at each boundary. These are the boundary points in flows:

- **Screen elements** -- each screen commits the current transaction and starts a new one
- **Pause elements** -- similar to screens, they end the transaction
- **Asynchronous Paths** (record-triggered flows) -- the async path runs in a separate transaction with its own governor limits
- **Scheduled Paths** (record-triggered flows) -- scheduled paths run in their own transaction
- **Platform Event publish** -- consuming a platform event starts a new transaction

Subflows do NOT create transaction boundaries. Calling a subflow from a parent flow executes all subflow elements in the parent's transaction, sharing the parent's governor limits. This is a common misconception.

### Before-Save vs After-Save: Impact on Shared Limits

Before-save flows modify the triggering record in memory before it is written to the database:
- No DML statement consumed to update the triggering record
- Runs approximately 10x faster than an equivalent after-save flow
- Cannot create, update, or delete other records
- Runs at step 4 in the order of execution -- before Apex triggers (step 5)

After-save flows run after the record is committed to the database (step 15):
- Every DML element counts against the shared 150 DML limit
- Every Get Records element counts against the shared 100 SOQL limit
- If an after-save flow updates the triggering record, the ENTIRE order of execution restarts (steps 1-20), consuming limits again

**The Apex developer's rule of thumb:** If an admin asks you to review a new flow on a trigger-heavy object, the first thing to check is whether it is before-save or after-save. A before-save flow that only sets fields on the triggering record adds zero DML and zero SOQL to the transaction. An after-save flow that queries and updates related records can easily add 5-10 SOQL and 3-5 DML to every transaction.

### How Apex Invocable Methods Interact with Flow Limits

When a flow calls an `@InvocableMethod`, the Apex code's SOQL and DML consumption counts against the same transaction limits. There is no isolation. If the flow has already used 80 SOQL queries and the invocable method executes 25 more, the transaction fails.

This is particularly dangerous with screen flows that call heavyweight Apex actions. The flow may have performed several Get Records operations across multiple screens (each screen resets limits, but only for subsequent elements). If the Apex action runs in the same transaction as the final screen, it must operate within whatever headroom remains.

Use `Limits.getQueries()` and `Limits.getDMLStatements()` in your invocable methods to log consumption during development:

```apex
@InvocableMethod(label='Process Records')
public static List<Result> execute(List<Request> requests) {
    System.debug('// SOQL used before invocable: ' + Limits.getQueries() + '/' + Limits.getLimitQueries());
    System.debug('// DML used before invocable: ' + Limits.getDmlStatements() + '/' + Limits.getLimitDmlStatements());

    // ... method logic ...

    System.debug('// SOQL used after invocable: ' + Limits.getQueries() + '/' + Limits.getLimitQueries());
    System.debug('// DML used after invocable: ' + Limits.getDmlStatements() + '/' + Limits.getLimitDmlStatements());

    return results;
}
```

This tells you exactly how much headroom the flow left you and how much your method consumed.

## Per-Transaction Governor Limits Reference

These limits are shared across ALL automation in the transaction -- flows, Apex triggers, workflow rules, process builders, and invocable Apex.

| Limit | Synchronous | Asynchronous | Notes |
|---|---|---|---|
| SOQL queries | 100 | 200 | Each Get Records = 1 query. Each Apex SOQL statement = 1 query. All share this pool. |
| DML statements | 150 | 150 | Each Create/Update/Delete element = 1 DML. Each Apex DML statement = 1 DML. Shared. |
| Records retrieved (SOQL) | 50,000 | 50,000 | Total across all Get Records + all Apex SOQL in the transaction. |
| Records processed (DML) | 10,000 | 10,000 | Total across all Create/Update/Delete + all Apex DML in the transaction. |
| CPU time | 10,000 ms | 60,000 ms | Complex formulas, loops, Decision elements, and Apex code all consume CPU. Shared. |
| Heap size | 6 MB | 12 MB | Large collection variables in flows and large Apex collections share this. |
| Callouts (HTTP) | 100 | 100 | Each external service Action element + each Apex HttpRequest. Shared. |
| Email invocations | 10 | 10 | Per `Messaging.sendEmail()` call. 5,000 external recipients per org per day. |
| Flow interviews per 24 hours | 250,000 or (user licenses x 200), whichever is greater | | Across all flow types in the org. |

## Flow-Specific Structural Limits

These are NOT per-transaction governor limits. They are org-wide or per-flow structural limits that constrain how many flows you can build and run.

| Limit | Number | Notes |
|---|---|---|
| Active flows per type per org | 2,000 | Per flow type (RecordTriggeredFlow, ScreenFlow, AutoLaunchedFlow, etc.). Hitting this means you cannot activate new flows of that type. |
| Paused flow interviews per org | 50,000 | Paused interviews consume storage. You cannot delete a flow version that has paused interviews. |
| Scheduled flow interviews per 24 hours | 250,000 | Or (number of user licenses x 200), whichever is greater. Applies to schedule-triggered flows AND scheduled paths on record-triggered flows. |
| Scheduled paths per record-triggered flow | 10 | Each scheduled path runs in its own transaction -- plan your async work across these 10 slots. |
| Flow Tests per flow | 200 | Declarative tests only. For complex flows, supplement with Apex test classes. |
| Flow interview max serialized size | 1 MB | Relevant for Pause elements. Large collection variables can exceed this and cause "Serialized flow interview exceeds maximum size" errors. |
| Executed elements per interview | No limit (API v57.0+) | Removed in Spring '23. Flows on API 56.0 and earlier still enforce 2,000 elements. Upgrade your flow's API version to 66.0+. |

## Configuration Examples

### Anti-Pattern: DML Inside a Loop (The Cardinal Sin)

This is the single most common governor limit violation in flows. A flow that works perfectly with 1 record fails catastrophically during data loads.

```
[Record-Triggered Flow: After Save on Opportunity]

Start
  |
Loop (over triggering records)
  |
  Get Records -> Account where Id = {!Loop_Variable.AccountId}     <- 1 SOQL PER ITERATION
  |
  Assignment -> set Account.Description = "Updated"
  |
  Update Records -> {!Account_Record}                               <- 1 DML PER ITERATION
  |
  (back to Loop)
```

With 200 records in a data load: 200 SOQL queries (exceeds 100 limit at record 101) + 200 DML statements (exceeds 150 limit at record 151). And that is just the flow -- if an Apex trigger on the same object uses even 1 SOQL query, the effective limit for the flow drops to 99.

### Correct Pattern: Collection Variables with DML Outside the Loop

```
[Record-Triggered Flow: After Save on Opportunity]

Start
  |
Get Records -> All Accounts related to triggering Opportunities    <- 1 SOQL TOTAL
  (Filter: Id IN {!collection of AccountIds from triggering records})
  |
Loop (over Account collection)
  |
  Assignment -> set {!Loop_Variable.Description} = "Updated"
  |
  Assignment -> Add {!Loop_Variable} to {!Accounts_To_Update}
  |
  (back to Loop)
  |
Update Records -> {!Accounts_To_Update}                             <- 1 DML TOTAL
```

This uses 1 SOQL and 1 DML regardless of batch size. Combined with Apex triggers on the same object, this leaves maximum headroom.

### Transform Element Pattern (Spring '24+)

When creating records from a source collection, the Transform element replaces the Loop + Assignment pattern entirely.

**Scenario:** After inserting Opportunities, create corresponding Project records.

```
[Record-Triggered Flow: After Save on Opportunity]

Start (Insert only)
  |
Transform Element:
  Source Collection: {!$Record} (triggering Opportunity records)
  Target Object: Project__c
  Field Mappings:
    Project__c.Name          <- Opportunity.Name + " - Project"
    Project__c.Account__c    <- Opportunity.AccountId
    Project__c.Start_Date__c <- Opportunity.CloseDate
    Project__c.Status__c     <- "Not Started" (constant)
  Output: {!Projects_To_Create}
  |
Create Records -> {!Projects_To_Create}                             <- 1 DML TOTAL
```

No loop. No assignment elements. No intermediate collection variable. The Transform maps fields declaratively and produces a ready-to-insert collection.

### Mixed DML Error: Cause and Solutions

The `MIXED_DML_OPERATION` error occurs when setup objects (User, PermissionSet, Group) and non-setup objects (Account, Contact, custom objects) are modified in the same transaction. This applies whether the DML originates in a flow, in Apex, or split across both.

**Common cross-automation scenario:** An after-save flow creates a Contact (non-setup DML). An Apex trigger on the same object updates a PermissionSetAssignment (setup DML). Both run in the same transaction. The transaction fails with `MIXED_DML_OPERATION` even though neither automation mixes DML on its own.

**Solution 1 -- Scheduled Path (0-minute delay):**
Move the setup object DML to a Scheduled Path with a 0-minute delay. This runs in a separate transaction.

```
[Record-Triggered Flow: After Save on Contact]

Immediate Path:
  Create Records -> Create an Account (non-setup DML)

Scheduled Path (0 minutes after record is created):
  Update Records -> Update User record (setup DML)     <- separate transaction
```

**Solution 2 -- Platform Event:**
Publish a platform event after the non-setup DML. A Platform Event-Triggered Flow performs the setup DML in its own transaction.

**Solution 3 -- @future from Invocable Apex:**
The flow calls an `@InvocableMethod` that delegates setup DML to an `@future` method:

```apex
public with sharing class AssignPermissionSetAction {

    @InvocableMethod(label='Assign Permission Set Async')
    public static void execute(List<Request> requests) {
        // collect user IDs for the async method
        Set<Id> userIds = new Set<Id>();
        Set<Id> permSetIds = new Set<Id>();
        for (Request req : requests) {
            userIds.add(req.userId);
            permSetIds.add(req.permissionSetId);
        }
        // delegate to @future -- runs in separate transaction
        assignPermSetFuture(userIds, permSetIds);
    }

    @future
    private static void assignPermSetFuture(Set<Id> userIds, Set<Id> permSetIds) {
        List<PermissionSetAssignment> assignments = new List<PermissionSetAssignment>();
        for (Id userId : userIds) {
            for (Id psId : permSetIds) {
                assignments.add(new PermissionSetAssignment(
                    AssigneeId = userId,
                    PermissionSetId = psId
                ));
            }
        }
        // setup DML in its own transaction -- no mixed DML risk
        insert assignments;
    }

    public class Request {
        @InvocableVariable(label='User ID' required=true)
        public Id userId;
        @InvocableVariable(label='Permission Set ID' required=true)
        public Id permissionSetId;
    }
}
```

### Error Handling with Fault Paths

**Without a fault path:** A DML failure rolls back the entire transaction. For record-triggered flows, the triggering record's save fails. The user sees a generic "An unhandled fault has occurred" message.

**With a fault path:** The transaction continues (no rollback). The failed element is skipped and execution continues down the fault connector. The triggering record's save succeeds. Partial commits can occur.

```
[Record-Triggered Flow: After Save on Opportunity]

Start
  |
Create Records -> Create a Task              --- fault path ---> Assignment
  | (success)                                                      |
  End                                                Set {!Error_Message} = {!$Flow.FaultMessage}
                                                                   |
                                                     Create Records -> Create Error_Log__c
                                                                   |
                                                                  End
```

Key error-handling variables:
- `{!$Flow.FaultMessage}` -- the full error message text from the failed element
- `{!$Flow.CurrentDateTime}` -- timestamp for error logging

**Custom Error Element (record-triggered flows only):** Blocks the save and displays a user-facing error. Rolls back the entire transaction. Use this when validation fails and the record should NOT be saved.

**Rollback Element (screen flows only):** Undoes all DML since the last screen (transaction boundary). Use this for all-or-nothing behavior with a user-friendly error message.

### Profiling Shared Limit Consumption from Apex

When you need to understand how much headroom a flow leaves for your Apex trigger, add limit profiling to your trigger handler:

```apex
public class AccountTriggerHandler {

    public static void handleAfterInsert(List<Account> newAccounts) {
        // log limits BEFORE trigger logic -- shows what the before-save flow
        // and before trigger already consumed
        System.debug('// === AFTER INSERT HANDLER START ===');
        System.debug('// SOQL queries consumed so far: '
            + Limits.getQueries() + '/' + Limits.getLimitQueries());
        System.debug('// DML statements consumed so far: '
            + Limits.getDmlStatements() + '/' + Limits.getLimitDmlStatements());
        System.debug('// CPU time consumed so far: '
            + Limits.getCpuTime() + 'ms/' + Limits.getLimitCpuTime() + 'ms');

        // ... trigger logic ...

        System.debug('// === AFTER INSERT HANDLER END ===');
        System.debug('// SOQL queries after handler: '
            + Limits.getQueries() + '/' + Limits.getLimitQueries());
        System.debug('// DML statements after handler: '
            + Limits.getDmlStatements() + '/' + Limits.getLimitDmlStatements());
    }
}
```

Run a Data Loader insert of 200 records and check the debug log. The "consumed so far" values at handler start show what the before-save flow (step 4) consumed. The delta between start and end shows the trigger's own consumption. The after-save flow (step 15) will consume whatever is left after the trigger finishes.

### Apex Test: Verify Combined Flow + Trigger Limit Consumption

Write an Apex test that inserts 200 records and asserts that the combined flow + trigger consumption stays within acceptable bounds. This catches limit regressions when someone adds a new Get Records element to a flow.

```apex
@IsTest
private class AccountAutomationLimitsTest {

    @IsTest
    static void testBulkInsert_limitsStayWithinBudget() {
        // build 200 Accounts to simulate a Data Loader batch
        List<Account> accounts = new List<Account>();
        for (Integer i = 0; i < 200; i++) {
            accounts.add(new Account(
                Name = 'Limit Test ' + i,
                Industry = 'Technology',
                BillingState = 'CA'
            ));
        }

        Test.startTest();
        insert accounts;
        // after Test.stopTest(), we can check how many limits were consumed
        // in the test transaction (which includes flows + triggers)
        Integer soqlUsed = Limits.getQueries();
        Integer dmlUsed = Limits.getDmlStatements();
        Integer cpuUsed = Limits.getCpuTime();
        Test.stopTest();

        // assert that combined consumption stays below 70% of limits
        // adjust these thresholds based on your automation inventory
        System.assert(soqlUsed < 70,
            'Combined flow + trigger SOQL should stay under 70. Actual: ' + soqlUsed);
        System.assert(dmlUsed < 105,
            'Combined flow + trigger DML should stay under 105. Actual: ' + dmlUsed);

        // verify the flow actually ran (not just that limits were low because nothing fired)
        List<Account> results = [SELECT Region__c FROM Account WHERE Id IN :accounts];
        for (Account a : results) {
            System.assertNotEquals(null, a.Region__c,
                'Before-save flow should have set Region__c');
        }
    }
}
```

This test serves as a regression guard. If an admin adds a Get Records element inside a loop to the Account flow, this test will fail when the SOQL count jumps from (e.g.) 5 to 205. It catches the problem before it hits production during a data load.

### Breaking Transaction Boundaries to Avoid Limits

**Screen Flows -- use screen elements to batch work:**
```
Screen 1 -> Get Records (batch 1) -> Loop -> Update Records
  | (transaction commits, limits reset)
Screen 2 -> Get Records (batch 2) -> Loop -> Update Records
  | (transaction commits, limits reset)
Screen 3 -> Confirmation
```

Each screen commits the current transaction and starts a new one with fresh governor limits. Use this to process large datasets across multiple screens.

**Record-Triggered Flows -- use Asynchronous Paths:**
```
[Record-Triggered Flow: After Save on Opportunity]

Immediate Path:
  Field Update on related Account (lightweight, within limits)

Asynchronous Path (Run Asynchronously):
  Heavy processing -- multiple queries, external callouts, large DML operations
  (Runs in its own transaction with its own governor limits)
```

The asynchronous path gets a full 200 SOQL queries and 60,000ms CPU time (async limits). Use this for work that does not need to complete before the user sees the save confirmation.

## Common Mistakes

1. **DML inside a loop ("the cardinal sin of Flow")** -- This is the single most common governor limit violation. A flow that works perfectly with 1 record fails catastrophically during data loads, imports, or batch updates. Fix: Use the collection pattern (query before the loop, assign inside the loop, DML after the loop) or use the Transform element.

2. **Get Records inside a loop** -- Same principle as DML in a loop but hits the 100 SOQL limit. When combined with Apex triggers that also query, the effective limit inside the flow is even lower. Fix: Perform a single Get Records before the loop that retrieves all needed records, then filter within the loop using Decision elements.

3. **Not accounting for Apex trigger consumption when designing flows** -- Your flow uses 80 SOQL queries, which works in isolation. But the Apex trigger on the same object uses 25 queries. Transaction total: 105 -- over the limit. This only manifests when both automations fire in the same transaction, so it may not appear in testing if the trigger does not exist in the sandbox. Fix: Always audit the full automation inventory on the object. Design flows to use no more than 30-40% of any limit, leaving headroom for triggers and other automation.

4. **Assuming subflows reset governor limits** -- Calling a subflow does NOT create a new transaction. All elements in the subflow count against the same transaction limits as the parent flow. Fix: Use Asynchronous Paths, Platform Events, or Screen elements to create actual transaction boundaries.

5. **Ignoring the 50,000 record retrieval limit** -- A Get Records with no filter (or a very broad filter) can retrieve up to 50,000 records. If your Apex trigger also queries large datasets, you can hit the ceiling from either side. Fix: Always filter Get Records as narrowly as possible. Use "Only the first record" when you need a single record.

6. **Using After-Save flow to update the triggering record when Before-Save would work** -- An after-save Update Records on the triggering record restarts the entire order of execution (steps 1-20), re-firing all before-save flows AND all Apex triggers. This doubles CPU time and DML. Fix: Use a before-save flow to set fields on the triggering record. Reserve after-save flows for cross-object operations.

7. **Forgetting that fault paths prevent rollback** -- Adding a fault connector means the transaction continues after a DML failure. If the flow created records before the failed element, those records persist. In a shared transaction with Apex, this can leave the database in an inconsistent state. Fix: If you need atomicity, omit the fault path and let the error roll back. If you need graceful error handling, add explicit Rollback (screen flows) or Custom Error (record-triggered flows) elements in the fault path.

8. **Not upgrading the flow API version** -- Flows on API version 56.0 and earlier enforce a 2,000 executed elements limit per interview. This limit was removed in API 57.0 (Spring '23). Fix: Open the flow in Flow Builder, check the API version in flow properties, update to 66.0+.

9. **Mixed DML across automation boundaries** -- A flow creates a Contact (non-setup). An Apex trigger on Contact updates a User record (setup). Neither automation mixes DML on its own, but they share a transaction. Fix: Move one DML operation to an async context (Scheduled Path, @future, Platform Event). Audit the full transaction for mixed DML, not just individual automations.

10. **CPU timeout on complex loops** -- Flows with large loops containing multiple Decision elements, Assignment elements, and formula evaluations can exceed the 10,000ms CPU limit. When combined with Apex trigger CPU consumption, the effective time available to the flow shrinks. Fix: Simplify loop logic, use the Transform element where possible, or break work across Asynchronous Paths. Profile CPU consumption from your Apex handler to know how much time remains for the flow.

## Flow Performance Optimization Checklist

- [ ] All DML elements are outside loops
- [ ] All Get Records elements are outside loops (or use the pre-fetched collection pattern)
- [ ] Before-save flow is used for triggering record field updates (zero DML cost)
- [ ] Transform element is used instead of Loop + Assignment where applicable (API v60.0+)
- [ ] Get Records elements have tight filters and use "Only the first record" where applicable
- [ ] Flow API version is 66.0+ (removes the 2,000 element limit)
- [ ] Fault paths are added to all DML and Action elements
- [ ] No mixed DML (setup + non-setup) in the same transaction, including Apex triggers
- [ ] Asynchronous Paths handle heavy/independent processing
- [ ] Collection variables are used to batch DML operations
- [ ] Entry conditions prevent unnecessary flow interviews
- [ ] Automation inventory is documented for all objects with flows + triggers
- [ ] Limit profiling confirms combined flow + Apex consumption stays below 70% of each limit

## See Also

- [Flow + Apex Coexistence Patterns](./flow-apex-coexistence.md) (order of execution, invocable method patterns, automation density framework)
- [Flow Security and Testing](./flow-security-and-testing.md) (run contexts, testing flows from Apex, null handling)
- [Invocable Methods](../apex-fundamentals/invocable-methods.md) (building the flow-to-Apex bridge)
- [Apex Fundamentals](../apex-fundamentals/apex-fundamentals.md) (governor limits from the Apex side)
- Salesforce Help: [Triggers and Order of Execution](https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_triggers_order_of_execution.htm)
- Salesforce Help: [Transform Element](https://help.salesforce.com/s/articleView?id=sf.flow_ref_elements_transform.htm&language=en_US&type=5)
- Salesforce Architect: [Record-Triggered Automation Decision Guide](https://architect.salesforce.com/docs/architect/decision-guides/guide/record-triggered)
