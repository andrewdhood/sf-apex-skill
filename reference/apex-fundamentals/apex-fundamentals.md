# Apex Fundamentals for Solution Architects

> **When to read this:** Before writing any Apex class, trigger, or test. This is the comprehensive quick-reference for producing correct, production-grade Apex that respects governor limits and Salesforce security.

## Rules

- When declaring a top-level class, always specify an explicit access modifier (`public` or `global`) because Apex requires it and omitting it causes a compile error.
- When writing any class that touches records, always declare a sharing keyword (`with sharing`, `without sharing`, or `inherited sharing`) because omitting it defaults to `without sharing` in most contexts, which silently ignores record-level security.
- Always default to `with sharing` on every class unless you have a documented reason to bypass sharing rules, because this enforces the running user's record access and is the Salesforce security best practice.
- Never use `without sharing` on a controller serving a Visualforce page or `@AuraEnabled` methods unless you explicitly need system-level access, because the user could see records they shouldn't.
- When writing SOQL, always use bind variables instead of string concatenation because concatenation opens the door to SOQL injection attacks.
- Never put SOQL queries or DML statements inside a `for` loop because each iteration counts against the 100-SOQL / 150-DML governor limit, and a batch of 200 records will blow past both.
- Always bulkify every trigger, invocable method, and batch handler by operating on `List<SObject>` rather than single records because Salesforce processes records in chunks of up to 200.
- When you need partial-success DML, always use `Database.insert(records, false)` and iterate `Database.SaveResult[]` to handle failures, because the plain `insert records` statement is all-or-nothing and rolls back the entire transaction on any single failure.
- Always use `Test.startTest()` / `Test.stopTest()` in test methods to reset governor limits for the code under test, because without them your test setup counts against the same limits as your code.
- Never write a test that only asserts "no exception was thrown" because that proves nothing about correctness. Always assert specific field values, record counts, or error messages.

## How It Works

### Class Structure & Access Modifiers

Apex classes follow a Java-like syntax but run on Salesforce's multi-tenant platform with its own set of rules. Every top-level class needs an access modifier. Inner classes default to `private` if no modifier is specified.

| Modifier | Visibility | When to use |
|----------|-----------|-------------|
| `public` | All Apex in the same namespace | Default choice for most classes |
| `private` | Same class only (or outer class for inner classes) | Inner classes, helper methods |
| `global` | All namespaces, including managed packages and web services | `@RestResource` classes, managed package APIs |
| `virtual` | Can be extended and methods overridden | Base classes in trigger handler frameworks |
| `abstract` | Must be extended; cannot be instantiated directly | Framework contracts where subclasses provide implementation |

Classes can also implement interfaces (`implements`) or extend other classes (`extends`). A class can implement multiple interfaces but extend only one class. Interfaces are implicitly `global`.

```apex
// a base class that other classes can extend
public virtual class BaseService {
    public virtual void execute() {
        // default implementation
    }
}

// a subclass that overrides the base behavior
public class AccountService extends BaseService {
    public override void execute() {
        // account-specific logic
    }
}

// an interface defining a contract
public interface Loggable {
    void log(String message);
}

// a class implementing the interface
public class AccountService extends BaseService implements Loggable {
    public override void execute() { /* ... */ }
    public void log(String message) {
        System.debug(message);
    }
}
```

### Sharing Keywords

Sharing keywords control whether the running user's record-level access (OWD, sharing rules, manual shares, role hierarchy) is enforced.

| Keyword | Behavior | Use when |
|---------|----------|----------|
| `with sharing` | Enforces current user's sharing rules | Default for all controllers, services, and `@AuraEnabled` classes |
| `without sharing` | Ignores sharing rules; sees all records | Batch jobs needing full data access, system integrations, specific utility methods that must see all data |
| `inherited sharing` | Uses the sharing mode of the calling class | Utility/helper classes that should adapt to their caller's context |
| *(omitted)* | Behaves like `without sharing` in most contexts | Never do this. Always be explicit. |

```apex
// the default for almost everything
public with sharing class PurchaseOrderService {
    // sharing rules of the running user are enforced
    // if the user can't see a PO record, SOQL won't return it
}

// use sparingly and document why
public without sharing class PurchaseOrderEscalation {
    // bypasses sharing to escalate POs across ownership boundaries
    // DOCUMENT: needed because escalation must find POs owned by any user
}

// perfect for utility classes
public inherited sharing class QueryHelper {
    // runs with whatever sharing mode the caller has
    // if called from a with-sharing class, sharing is enforced
    // if called from a without-sharing class, sharing is bypassed
}
```

**Critical detail:** Sharing keywords only control record-level access (which records you see). They do NOT control object-level CRUD or field-level security (FLS). You must separately enforce CRUD/FLS using `WITH USER_MODE` in SOQL, `Security.stripInaccessible()`, or `Schema.DescribeSObjectResult` checks.

### Data Types

**Primitives:** `String`, `Integer`, `Long`, `Decimal`, `Double`, `Boolean`, `Date`, `Datetime`, `Time`, `Id`, `Blob`

**Collections:**

```apex
// list: ordered, duplicates allowed, zero-indexed
List<Account> accounts = new List<Account>();
accounts.add(new Account(Name = 'Acme'));
Account first = accounts[0];

// set: unordered, no duplicates, great for deduplication
Set<Id> accountIds = new Set<Id>();
accountIds.add(someAccount.Id);
Boolean exists = accountIds.contains(someId);

// map: key-value pairs, keys are unique
Map<Id, Account> accountMap = new Map<Id, Account>();
accountMap.put(someAccount.Id, someAccount);
Account a = accountMap.get(someId);

// shorthand: build a map directly from a SOQL query
Map<Id, Account> accountMap = new Map<Id, Account>(
    [SELECT Id, Name FROM Account WHERE Id IN :accountIds]
);
```

**SObject and generic typing:**

```apex
// specific SObject type
Account acc = new Account(Name = 'Acme');

// generic SObject for dynamic code
SObject record = Schema.getGlobalDescribe().get('Account').newSObject();
record.put('Name', 'Dynamic Acme');

// casting from generic to specific
Account specificAcc = (Account) record;
```

### SOQL: Inline, Dynamic, and Advanced Patterns

**Inline SOQL (compile-time checked):**

```apex
// basic query with bind variable
String industry = 'Technology';
List<Account> accounts = [
    SELECT Id, Name, Industry
    FROM Account
    WHERE Industry = :industry
    LIMIT 200
];

// relationship query: parent-to-child (subquery)
List<Account> accountsWithContacts = [
    SELECT Id, Name,
        (SELECT Id, FirstName, LastName, Email FROM Contacts)
    FROM Account
    WHERE Id IN :accountIds
];

// relationship query: child-to-parent (dot notation)
List<Contact> contacts = [
    SELECT Id, FirstName, Account.Name, Account.Industry
    FROM Contact
    WHERE Account.Industry = 'Technology'
];

// aggregate query
List<AggregateResult> results = [
    SELECT AccountId, COUNT(Id) contactCount, MAX(CreatedDate) latestContact
    FROM Contact
    GROUP BY AccountId
    HAVING COUNT(Id) > 5
];
// access aggregate fields by alias
for (AggregateResult ar : results) {
    Id accountId = (Id) ar.get('AccountId');
    Integer count = (Integer) ar.get('contactCount');
}
```

**Dynamic SOQL (runtime-constructed):**

```apex
// use Database.query() for simple cases
String objectName = 'Account';
String query = 'SELECT Id, Name FROM ' + objectName + ' WHERE Industry = :industry';
List<SObject> results = Database.query(query);

// use Database.queryWithBinds() for safe bind variable resolution (Spring '23+)
// this prevents SOQL injection and doesn't require variables to be in scope
String query = 'SELECT Id, Name FROM Account WHERE Industry = :industry AND CreatedDate > :startDate';
Map<String, Object> binds = new Map<String, Object>{
    'industry' => 'Technology',
    'startDate' => Date.today().addDays(-30)
};
List<Account> results = Database.queryWithBinds(query, binds, AccessLevel.USER_MODE);
```

**WITH USER_MODE (enforce CRUD + FLS in SOQL):**

```apex
// inline SOQL: enforces field-level security and object permissions
List<Account> accounts = [
    SELECT Id, Name, AnnualRevenue
    FROM Account
    WITH USER_MODE
];
// if the user lacks Read on AnnualRevenue, this throws an exception
// this is the preferred way to enforce FLS in Apex as of API v56+

// dynamic SOQL: pass AccessLevel.USER_MODE
List<Account> accounts = Database.query(
    'SELECT Id, Name FROM Account',
    AccessLevel.USER_MODE
);
```

### DML Operations

**Standard DML statements (all-or-nothing):**

```apex
// insert
Account acc = new Account(Name = 'New Account');
insert acc;
// after insert, acc.Id is populated

// update
acc.Name = 'Updated Account';
update acc;

// upsert: insert or update based on an external ID or record ID
// uses the record's Id field by default
upsert acc;
// or specify an external ID field
upsert acc External_Id__c;

// delete
delete acc;

// undelete: restore from recycle bin
undelete acc;
```

**Database class methods (partial success):**

```apex
// insert with partial success
List<Account> accounts = new List<Account>{
    new Account(Name = 'Good Account'),
    new Account() // missing required Name field
};

// allOrNone = false allows partial success
Database.SaveResult[] results = Database.insert(accounts, false);

// always iterate results to find failures
for (Integer i = 0; i < results.size(); i++) {
    if (results[i].isSuccess()) {
        System.debug('// inserted: ' + results[i].getId());
    } else {
        for (Database.Error err : results[i].getErrors()) {
            System.debug('// failed record ' + i + ': ' + err.getStatusCode() + ' - ' + err.getMessage());
        }
    }
}

// upsert returns Database.UpsertResult[]
Database.UpsertResult[] upsertResults = Database.upsert(records, External_Id__c, false);

// delete returns Database.DeleteResult[]
Database.DeleteResult[] deleteResults = Database.delete(recordIds, false);
```

**Bulkification pattern:**

```apex
// WRONG: DML inside a loop
for (Contact c : contacts) {
    c.MailingCity = 'San Francisco';
    update c; // 150 contacts = 150 DML statements = governor limit hit
}

// RIGHT: collect, then DML once
List<Contact> toUpdate = new List<Contact>();
for (Contact c : contacts) {
    c.MailingCity = 'San Francisco';
    toUpdate.add(c);
}
update toUpdate; // 1 DML statement regardless of list size
```

### Governor Limits Quick Reference

These limits apply per-transaction. Exceeding any of them throws a `System.LimitException` that cannot be caught.

| Resource | Synchronous Limit | Asynchronous Limit |
|----------|-------------------|-------------------|
| SOQL queries | 100 | 200 |
| SOQL rows returned (total) | 50,000 | 50,000 |
| DML statements | 150 | 150 |
| DML rows (total) | 10,000 | 10,000 |
| Heap size | 6 MB | 12 MB |
| CPU time | 10,000 ms (10s) | 60,000 ms (60s) |
| Callouts (HTTP / web service) | 100 | 100 |
| Callout timeout (single) | 120,000 ms | 120,000 ms |
| Future method invocations | 50 | 0 (can't call future from future) |
| Queueable jobs enqueued | 50 | 1 |
| Email invocations (single) | 10 | 10 |
| SOSL queries | 20 | 20 |
| `Limits.getQueries()` | Check at runtime | Check at runtime |

**Checking limits at runtime:**

```apex
System.debug('// SOQL queries used: ' + Limits.getQueries() + ' of ' + Limits.getLimitQueries());
System.debug('// DML statements used: ' + Limits.getDmlStatements() + ' of ' + Limits.getLimitDmlStatements());
System.debug('// Heap size used: ' + Limits.getHeapSize() + ' of ' + Limits.getLimitHeapSize());
System.debug('// CPU time used: ' + Limits.getCpuTime() + 'ms of ' + Limits.getLimitCpuTime() + 'ms');
```

### Trigger Framework

Always use one trigger per object, with the trigger itself containing zero logic. All logic lives in a handler class.

**The trigger (logic-free):**

```apex
trigger AccountTrigger on Account (
    before insert, before update, before delete,
    after insert, after update, after delete, after undelete
) {
    AccountTriggerHandler handler = new AccountTriggerHandler();
    handler.run();
}
```

**Base handler class (virtual, reusable):**

```apex
public virtual class TriggerHandler {

    // recursion guard: track processed record IDs per context
    private static Map<String, Set<Id>> processedIds = new Map<String, Set<Id>>();

    public void run() {
        // route to the correct method based on trigger context
        if (Trigger.isBefore) {
            if (Trigger.isInsert) { beforeInsert(Trigger.new); }
            if (Trigger.isUpdate) { beforeUpdate(Trigger.new, Trigger.oldMap); }
            if (Trigger.isDelete) { beforeDelete(Trigger.old); }
        }
        if (Trigger.isAfter) {
            if (Trigger.isInsert) { afterInsert(Trigger.new); }
            if (Trigger.isUpdate) { afterUpdate(Trigger.new, Trigger.oldMap); }
            if (Trigger.isDelete) { afterDelete(Trigger.old); }
            if (Trigger.isUndelete) { afterUndelete(Trigger.new); }
        }
    }

    // check if a record has already been processed in this transaction
    // prevents infinite recursion when triggers cause re-entry
    protected Boolean hasBeenProcessed(Id recordId, String context) {
        String key = this.toString() + ':' + context;
        if (!processedIds.containsKey(key)) {
            processedIds.put(key, new Set<Id>());
        }
        if (processedIds.get(key).contains(recordId)) {
            return true;
        }
        processedIds.get(key).add(recordId);
        return false;
    }

    // subclasses override only the methods they need
    protected virtual void beforeInsert(List<SObject> newRecords) {}
    protected virtual void beforeUpdate(List<SObject> newRecords, Map<Id, SObject> oldMap) {}
    protected virtual void beforeDelete(List<SObject> oldRecords) {}
    protected virtual void afterInsert(List<SObject> newRecords) {}
    protected virtual void afterUpdate(List<SObject> newRecords, Map<Id, SObject> oldMap) {}
    protected virtual void afterDelete(List<SObject> oldRecords) {}
    protected virtual void afterUndelete(List<SObject> newRecords) {}
}
```

**Concrete handler with recursion guard:**

```apex
public class AccountTriggerHandler extends TriggerHandler {

    // after update: sync account name to related contacts
    protected override void afterUpdate(List<SObject> newRecords, Map<Id, SObject> oldMap) {
        List<Contact> contactsToUpdate = new List<Contact>();

        for (SObject sobj : newRecords) {
            Account acc = (Account) sobj;
            Account oldAcc = (Account) oldMap.get(acc.Id);

            // skip if we already processed this record (recursion guard)
            if (hasBeenProcessed(acc.Id, 'afterUpdate')) {
                continue;
            }

            // only act if the name actually changed
            if (acc.Name != oldAcc.Name) {
                // query related contacts
                for (Contact c : [SELECT Id FROM Contact WHERE AccountId = :acc.Id]) {
                    c.Account_Name_Cached__c = acc.Name;
                    contactsToUpdate.add(c);
                }
            }
        }

        if (!contactsToUpdate.isEmpty()) {
            update contactsToUpdate;
        }
    }
}
```

**Why not use a static Boolean for recursion prevention:** A simple `static Boolean hasRun = false` blocks the trigger for ALL records after the first execution. If a batch of 200 fires the trigger, then a subsequent DML in the same transaction legitimately triggers it again, the Boolean blocks all 200 records. Use ID-based tracking (a `Set<Id>`) so each record is processed exactly once.

### Exception Handling

**Try/catch basics:**

```apex
public with sharing class AccountService {

    // params: accountId - the record to process
    // returns: the updated Account
    public Account processAccount(Id accountId) {
        try {
            Account acc = [SELECT Id, Name FROM Account WHERE Id = :accountId];
            acc.Status__c = 'Processed';
            update acc;
            return acc;
        } catch (QueryException e) {
            System.debug('// query failed: ' + e.getMessage());
            throw new ServiceException('Account not found: ' + accountId, e);
        } catch (DmlException e) {
            System.debug('// DML failed: ' + e.getMessage() + ' at index ' + e.getDmlIndex(0));
            throw new ServiceException('Failed to update account: ' + e.getDmlMessage(0), e);
        }
    }
}
```

**Custom exception class:**

```apex
public class ServiceException extends Exception {}

// usage
throw new ServiceException('Purchase order not found');

// with cause chaining
try {
    update someRecord;
} catch (DmlException e) {
    throw new ServiceException('Update failed: ' + e.getDmlMessage(0), e);
}
```

**AuraHandledException for LWC/Aura:**

```apex
// when an @AuraEnabled method needs to send a user-friendly error to the UI
@AuraEnabled
public static String submitPurchaseOrder(Id poId) {
    try {
        // business logic here
        PurchaseOrderService.submit(poId);
        return 'Success';
    } catch (ServiceException e) {
        // AuraHandledException sends the message to the LWC
        // the LWC catches it in its .catch() or try/catch block
        throw new AuraHandledException(e.getMessage());
    } catch (Exception e) {
        // log the full error server-side for debugging
        System.debug('// unexpected error: ' + e.getMessage() + '\n' + e.getStackTraceString());
        // send a sanitized message to the client — never expose stack traces
        throw new AuraHandledException('An unexpected error occurred. Please contact your administrator.');
    }
}
```

**Critical:** You cannot extend `AuraHandledException`. If you need structured error data, serialize it into the message string and parse it in JavaScript.

### Asynchronous Apex

Use async processing when you need higher governor limits, callouts from triggers, or to defer long-running work.

| Type | When to use | Limits | Can chain? | Accepts SObjects? |
|------|------------|--------|------------|-------------------|
| `@future` | Simple fire-and-forget (callouts from triggers, lightweight post-processing) | 50 per txn, no monitoring, no chaining | No | No (primitives only) |
| `Queueable` | Complex async work, chaining, non-primitive params | 50 enqueued per txn (1 from async), monitorable via `AsyncApexJob` | Yes (chain 1 job) | Yes |
| `Batch` | Processing 1K-50M records in chunks | 5 concurrent, fresh limits per `execute()` chunk | Yes (via `finish()`) | Yes |
| `Schedulable` | Recurring time-based execution | 100 scheduled jobs org-wide | Typically starts a batch | N/A |

**@future method:**

```apex
public class ExternalSystemSync {

    // params: accountIds - IDs to sync to external system
    // note: @future methods can only accept primitives or collections of primitives
    @future(callout=true)
    public static void syncAccounts(Set<Id> accountIds) {
        List<Account> accounts = [SELECT Id, Name, BillingCity FROM Account WHERE Id IN :accountIds];
        // make HTTP callout to external system
        HttpRequest req = new HttpRequest();
        req.setEndpoint('https://api.example.com/sync');
        req.setMethod('POST');
        req.setBody(JSON.serialize(accounts));
        Http http = new Http();
        HttpResponse res = http.send(req);
        System.debug('// sync response: ' + res.getStatusCode());
    }
}
```

**Queueable Apex:**

```apex
public class AccountProcessingJob implements Queueable, Database.AllowsCallouts {

    private List<Account> accounts;

    // params: accounts - the records to process
    public AccountProcessingJob(List<Account> accounts) {
        this.accounts = accounts;
    }

    public void execute(QueueableContext ctx) {
        // process accounts with full governor limits of async context
        for (Account acc : accounts) {
            acc.Processed__c = true;
        }
        update accounts;

        // chain to another job if needed (max 1 from async context)
        if (thereIsMoreWork()) {
            System.enqueueJob(new AccountProcessingJob(nextBatch));
        }
    }
}

// enqueue from synchronous code
System.enqueueJob(new AccountProcessingJob(accountList));
```

**Batch Apex:**

```apex
public class AccountCleanupBatch implements Database.Batchable<SObject>, Database.Stateful {

    // stateful: instance variables persist across execute() calls
    private Integer totalProcessed = 0;

    // start: define the query for records to process
    // returns: a QueryLocator (up to 50 million records)
    public Database.QueryLocator start(Database.BatchableContext bc) {
        return Database.getQueryLocator([
            SELECT Id, Name, LastModifiedDate
            FROM Account
            WHERE LastModifiedDate < :Date.today().addYears(-2)
        ]);
    }

    // execute: runs once per chunk (default 200 records)
    // each execute() gets fresh governor limits
    public void execute(Database.BatchableContext bc, List<Account> scope) {
        for (Account acc : scope) {
            acc.Status__c = 'Archived';
        }
        update scope;
        totalProcessed += scope.size();
    }

    // finish: runs once after all chunks complete
    public void finish(Database.BatchableContext bc) {
        System.debug('// batch complete. total processed: ' + totalProcessed);
        // optionally chain another batch or send a notification
    }
}

// execute with custom batch size (default 200, max 2000)
Database.executeBatch(new AccountCleanupBatch(), 200);
```

**Schedulable Apex:**

```apex
public class NightlyCleanupScheduler implements Schedulable {

    // execute: runs at the scheduled time
    public void execute(SchedulableContext sc) {
        // typically kicks off a batch job
        Database.executeBatch(new AccountCleanupBatch(), 200);
    }
}

// schedule via anonymous Apex or setup menu
// run every night at 2 AM
String cronExp = '0 0 2 * * ?';
System.schedule('Nightly Account Cleanup', cronExp, new NightlyCleanupScheduler());

// cron expression format: seconds minutes hours day_of_month month day_of_week [year]
// '0 0 2 * * ?' = at 2:00 AM every day
// '0 0 12 ? * MON-FRI' = at noon on weekdays
// '0 0 0 1 * ?' = midnight on the 1st of every month
```

### Testing

**Test class structure:**

```apex
@isTest
private class AccountServiceTest {

    // @TestSetup runs once before all test methods
    // each test method gets its own copy of this data (rolled back between tests)
    @TestSetup
    static void setupTestData() {
        List<Account> accounts = new List<Account>();
        for (Integer i = 0; i < 200; i++) {
            accounts.add(new Account(Name = 'Test Account ' + i));
        }
        insert accounts;
    }

    @isTest
    static void testProcessAccount_positive() {
        // arrange: get the test data created in @TestSetup
        Account testAcc = [SELECT Id, Name FROM Account LIMIT 1];

        // act: reset governor limits, then run the code under test
        Test.startTest();
        AccountService service = new AccountService();
        Account result = service.processAccount(testAcc.Id);
        Test.stopTest();

        // assert: verify specific outcomes
        System.assertEquals('Processed', result.Status__c, 'Account status should be Processed');
        // re-query to confirm persistence
        Account refreshed = [SELECT Status__c FROM Account WHERE Id = :testAcc.Id];
        System.assertEquals('Processed', refreshed.Status__c, 'Persisted status should be Processed');
    }

    @isTest
    static void testProcessAccount_invalidId() {
        // arrange: use a fake ID that doesn't exist
        Id fakeId = '001000000000000AAA';

        // act + assert: verify the exception is thrown
        Test.startTest();
        try {
            AccountService service = new AccountService();
            service.processAccount(fakeId);
            System.assert(false, 'Expected ServiceException was not thrown');
        } catch (ServiceException e) {
            System.assert(e.getMessage().contains('not found'),
                'Exception message should mention not found: ' + e.getMessage());
        }
        Test.stopTest();
    }

    @isTest
    static void testProcessAccount_bulk() {
        // arrange: get all 200 test accounts
        List<Account> accounts = [SELECT Id FROM Account];
        System.assertEquals(200, accounts.size(), 'Should have 200 test accounts');

        // act: process all 200
        Test.startTest();
        AccountService service = new AccountService();
        for (Account acc : accounts) {
            service.processAccount(acc.Id);
        }
        Test.stopTest();

        // assert: all should be processed
        List<Account> processed = [SELECT Id FROM Account WHERE Status__c = 'Processed'];
        System.assertEquals(200, processed.size(), 'All 200 accounts should be Processed');
    }
}
```

**Key testing annotations and methods:**

| Annotation / Method | Purpose |
|---------------------|---------|
| `@isTest` | Marks a class or method as test-only. Does not count against org code limits. |
| `@TestSetup` | Creates shared test data once for the entire class. Each test gets its own rollback-isolated copy. |
| `@TestVisible` | Makes a private method or variable accessible to test classes without making it public. |
| `Test.startTest()` | Resets governor limits. All async code enqueued after this runs synchronously at `stopTest()`. |
| `Test.stopTest()` | Executes pending async work (future, queueable, batch). Restores original governor limits. |
| `System.assertEquals(expected, actual, msg)` | Asserts equality. Always include the message parameter. |
| `System.assertNotEquals(expected, actual, msg)` | Asserts inequality. |
| `System.assert(condition, msg)` | Asserts a Boolean condition is true. |
| `Test.isRunningTest()` | Returns true during test execution. Use sparingly to bypass callouts in tests. |

**Test best practices:**
- Aim for 90%+ code coverage (75% is the production deployment minimum, not the quality standard).
- Always test bulk operations (200+ records) to catch non-bulkified code.
- Test negative cases: bad input, missing records, permission errors.
- Never use `@SeeAllData=true` unless testing against org data that can't be created programmatically (like standard price books in some orgs).
- Use `System.runAs(testUser)` to test sharing and profile-specific behavior.

### Common Annotations Reference

| Annotation | Where | Purpose |
|------------|-------|---------|
| `@AuraEnabled` | Method or property | Exposes to Lightning components (LWC/Aura). Add `(cacheable=true)` for read-only methods the client can cache. |
| `@AuraEnabled(cacheable=true)` | Method | Client-side caching for read-only wire methods. Cannot perform DML. |
| `@InvocableMethod` | Static method | Exposes to Flow, Process Builder, REST API. One per class. Accepts `List<Request>` input. |
| `@InvocableVariable` | Instance variable | Marks a variable as input/output for an `@InvocableMethod` wrapper class. |
| `@future` | Static method | Runs async. Add `(callout=true)` for HTTP. Only primitive params. |
| `@HttpGet` / `@HttpPost` / `@HttpPatch` / `@HttpDelete` | Method in `@RestResource` class | Maps HTTP verbs to Apex methods for REST APIs. |
| `@RestResource(urlMapping='/myEndpoint/*')` | Class | Exposes a class as a REST endpoint at `/services/apexrest/myEndpoint/`. |
| `@TestVisible` | Method or variable | Lets test classes access private members without changing actual visibility. |
| `@isTest` | Class or method | Marks test code. Does not count against Apex code size limit. |
| `@TestSetup` | Static method | Runs once to create shared test data for the class. |
| `@ReadOnly` | Method | Raises SOQL row limit to 1,000,000 but blocks all DML. For VF read-only pages. |
| `@SuppressWarnings` | Class or method | Suppresses compiler warnings (rarely needed). |

## Code Examples

### Complete @InvocableMethod with wrapper class

```apex
public with sharing class SendPurchaseOrderEmail {

    // the invocable method: called from Flow
    // params: requests - list of Request wrappers from the flow
    // returns: list of Result wrappers back to the flow
    @InvocableMethod(
        label='Send Purchase Order Email'
        description='Generates a PDF and emails it to the specified contact'
        category='Purchase Orders'
    )
    public static List<Result> execute(List<Request> requests) {
        List<Result> results = new List<Result>();

        for (Request req : requests) {
            Result res = new Result();
            try {
                // generate PDF, send email, log action
                PurchaseOrderService.sendPdf(req.purchaseOrderId, req.contactId);
                res.isSuccess = true;
                res.message = 'Email sent successfully';
            } catch (Exception e) {
                res.isSuccess = false;
                res.message = e.getMessage();
                System.debug('// email send failed: ' + e.getMessage());
            }
            results.add(res);
        }
        return results;
    }

    // input wrapper
    public class Request {
        @InvocableVariable(label='Purchase Order ID' required=true)
        public Id purchaseOrderId;

        @InvocableVariable(label='Contact ID' required=true)
        public Id contactId;
    }

    // output wrapper
    public class Result {
        @InvocableVariable(label='Success')
        public Boolean isSuccess;

        @InvocableVariable(label='Message')
        public String message;
    }
}
```

### Complete @AuraEnabled controller for LWC

```apex
public with sharing class PurchaseOrderController {

    // wire-compatible: read-only, cacheable
    // params: recordId - the purchase order ID from the LWC
    // returns: the purchase order record with related line items
    @AuraEnabled(cacheable=true)
    public static Purchase_Order__c getPurchaseOrder(Id recordId) {
        return [
            SELECT Id, Name, Status__c, Total_Amount__c, Authorizer_Name__c,
                (SELECT Id, Product__c, Quantity__c, Unit_Price__c FROM Line_Items__r)
            FROM Purchase_Order__c
            WHERE Id = :recordId
            WITH USER_MODE
        ];
    }

    // imperative: performs DML, not cacheable
    // params: poId - the purchase order to submit
    // returns: success message string
    @AuraEnabled
    public static String submitPurchaseOrder(Id poId) {
        try {
            Purchase_Order__c po = [
                SELECT Id, Status__c, Authorizer_Name__c
                FROM Purchase_Order__c
                WHERE Id = :poId
                WITH USER_MODE
            ];

            if (po.Status__c == 'Submitted') {
                throw new AuraHandledException('This purchase order has already been submitted.');
            }

            po.Status__c = 'Submitted';
            po.Digital_Authorization__c = 'Digital Authorization from ' + po.Authorizer_Name__c;
            update po;
            return 'Purchase order submitted successfully';
        } catch (AuraHandledException e) {
            throw e; // re-throw as-is for LWC
        } catch (Exception e) {
            System.debug('// submit failed: ' + e.getMessage() + '\n' + e.getStackTraceString());
            throw new AuraHandledException('Failed to submit purchase order. Contact your administrator.');
        }
    }
}
```

## Common Mistakes

1. **Omitting sharing keywords on classes** -- Happens because the compiler doesn't require them. Fix: Always declare `with sharing` on every class. Use a code review checklist or static analysis rule (PMD) to catch omissions.

2. **SOQL inside a for loop** -- Happens when processing records one at a time. Fix: Collect IDs into a `Set<Id>`, query once outside the loop, and build a `Map<Id, SObject>` for lookups.

3. **DML inside a for loop** -- Same root cause as SOQL in loops. Fix: Collect records into a `List<SObject>`, then perform a single DML statement after the loop.

4. **Using a static Boolean for recursion prevention** -- Happens because tutorials often show `static Boolean hasRun`. Fix: Use a `static Set<Id>` that tracks individual record IDs, so each record is processed exactly once per context.

5. **Writing tests that only check for no exceptions** -- Happens when developers write tests purely for coverage. Fix: Assert specific field values, record counts, error messages, and negative scenarios.

6. **Using `@SeeAllData=true` in test classes** -- Happens when test data setup feels complex. Fix: Create all test data in `@TestSetup`. The only valid exception is when referencing data that can't be created in tests (like some standard price book scenarios).

7. **Passing SObject parameters to @future methods** -- The compiler allows it through serialization tricks, but @future only officially supports primitives. Fix: Pass `Set<Id>` and re-query the records inside the @future method. Or better yet, use Queueable which accepts any parameter type.

8. **Not including a message in System.assertEquals** -- When the assertion fails, you get a generic error with no context. Fix: Always use the three-parameter form: `System.assertEquals(expected, actual, 'descriptive message')`.

9. **Forgetting to test bulk operations** -- Works fine with 1 record but fails at 200. Fix: Always include a test that creates and processes 200+ records to surface hidden SOQL/DML-in-loop patterns.

10. **Using Database.query() with string concatenation instead of bind variables** -- Creates SOQL injection vulnerabilities. Fix: Use `Database.queryWithBinds()` with a bind map, or use inline SOQL with `:variable` bind syntax.

## See Also

- [Deployment & CI/CD](../deployment/apex-deployment.md)
- [Visualforce PDF Generation](../visualforce-pdf/)
- [Email Services](../email-services/)
- [Quick Actions & Invocable Methods](../quick-actions/)
- [Flows & Automation](../flows-and-automation/)
