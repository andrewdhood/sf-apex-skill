# Apex Design Patterns

> **When to read this:** You are building a non-trivial Apex solution and need complete, production-grade implementations of the core design patterns. This goes beyond architecture overviews and provides working classes you can adapt for any Salesforce org.

## Rules

- When building triggers, always use the **one trigger per object -> handler class -> domain/service** delegation chain because it enforces testable separation of concerns and prevents unpredictable execution order.
- When preventing trigger recursion, always use a **Set\<Id\>** to track already-processed record IDs, never a single Boolean flag, because a Boolean blocks the entire trigger from re-firing even when new, unprocessed records enter the context.
- When building any service method that performs DML on more than one object, always use the **Unit of Work pattern** (or at minimum a SavePoint) because partial commits corrupt data and leave orphaned records.
- When writing SOQL queries, always centralize them in a **Selector class** per object because scattered inline SOQL leads to inconsistent field sets, duplicated queries, and missed FLS enforcement.
- When a Selector returns records to untrusted callers (LWC, guest users), always use **WITH USER_MODE** in the SOQL query because it enforces CRUD and FLS for the running user at the query level.
- When business logic operates on a collection of SObject records (validation, defaulting, calculations), always encapsulate it in a **Domain class** because putting it directly in the trigger handler creates god classes.
- When multiple parts of the system need the same expensive configuration data (Custom Metadata, org-wide defaults), always use the **Singleton pattern** because re-querying the same data multiple times per transaction wastes CPU and SOQL.
- When different records require different business rules at runtime (pricing strategies, approval routing, discount tiers), always use the **Strategy pattern** because large if/else or switch blocks are unmaintainable and violate the Open/Closed Principle.
- When building large-volume data processing, always choose between Batch and Queueable based on **record volume and chaining needs** because Batch handles up to 50 million records with per-chunk governor resets while Queueable handles smaller volumes with flexible chaining.
- When testing service or domain classes, always inject dependencies through a **Factory** so tests can substitute mock selectors and mock DML without hitting the database because database-free tests run 10-50x faster.

## How It Works

### 1. Trigger Handler Pattern

The trigger handler pattern keeps trigger files thin (routing only) and pushes all logic into testable handler classes. The handler dispatches to Domain and Service classes for the actual work.

**Architecture:**
```
Trigger (thin) -> TriggerHandler (routing) -> Domain (record logic) / Service (transaction logic)
```

**The interface defines the contract** that every handler must fulfill. Each trigger context (before insert, after update, etc.) gets its own method so handlers only implement what they need.

**Recursion prevention** uses a static `Set<Id>` rather than a Boolean. A Boolean blocks all re-entry; a Set tracks which specific records have been processed and allows new records through.

### 2. Service Layer Pattern

Service classes are the transaction boundary. They contain `public static` methods that represent business operations (submitOrder, cancelOrder, sendEmail). Services orchestrate calls to Selectors (for queries), Domain classes (for record logic), and commit DML through a Unit of Work or SavePoint.

**Key principles:**
- Methods accept collections (`List`, `Set`, `Map`), never single records
- Methods are `public static` so any caller (trigger, flow, batch, REST) can use them
- The service method IS the transaction boundary -- it creates the SavePoint or Unit of Work
- Default to `with sharing`; only use `without sharing` when explicitly justified

### 3. Selector Pattern

Selector classes centralize all SOQL for a given object. Every query for Account goes through `AccountSelector`, regardless of the caller. This ensures consistent field sets, centralized FLS enforcement, and a single place to optimize queries.

**WITH USER_MODE** replaces the older `WITH SECURITY_ENFORCED` keyword. It enforces CRUD and FLS at the query level, meaning fields the running user cannot see are stripped from results rather than throwing an exception.

### 4. Domain Pattern

Domain classes encapsulate business logic that operates on SObject collections: validation, field defaulting, calculations, and state transitions. The Domain class constructor accepts `List<SObject>` to enforce bulkification. Domain classes are called from trigger handlers, services, and batch jobs.

### 5. Unit of Work Pattern

The Unit of Work collects all DML operations (inserts, updates, deletes) across multiple objects and commits them in one transaction. If anything fails, everything rolls back. This eliminates partial commits and orphaned records.

### 6. Factory Pattern

Factories provide a single point where classes obtain their dependencies (Selectors, Services, Domain classes). In production, the factory returns real implementations. In tests, the factory can be overridden to return mocks. This enables database-free unit tests.

### 7. Strategy Pattern

The Strategy pattern defines a family of algorithms behind a common interface and lets the caller select the right one at runtime. This replaces sprawling if/else chains with clean, pluggable implementations.

### 8. Singleton Pattern

The Singleton ensures a class has exactly one instance per transaction, with a global access point. This is ideal for caching configuration (Custom Metadata Types, org defaults) that should only be loaded once.

### 9. Batch Processing Patterns

Batch Apex processes large datasets (up to 50 million records) in chunks with per-chunk governor limit resets. Queueable Apex handles smaller volumes with flexible chaining and complex parameter passing. Finalizers guarantee post-execution cleanup even when a Queueable fails.

## Code Examples

### 1. Trigger Handler Pattern -- Complete Implementation

#### ITriggerHandler Interface

```apex
// interface defining the contract for all trigger handlers
// each method corresponds to a trigger context -- handlers implement only what they need
public interface ITriggerHandler {

    // called before records are inserted
    // input: newRecords - the list of new SObjects being inserted
    void beforeInsert(List<SObject> newRecords);

    // called before records are updated
    // input: newRecords - the updated records, oldMap - previous versions keyed by Id
    void beforeUpdate(List<SObject> newRecords, Map<Id, SObject> oldMap);

    // called before records are deleted
    // input: oldRecords - the records about to be deleted, oldMap - same records keyed by Id
    void beforeDelete(List<SObject> oldRecords, Map<Id, SObject> oldMap);

    // called after records are inserted
    // input: newRecords - the inserted records (now have IDs), newMap - keyed by Id
    void afterInsert(List<SObject> newRecords, Map<Id, SObject> newMap);

    // called after records are updated
    // input: newRecords - updated records, newMap - keyed by Id, oldMap - previous versions
    void afterUpdate(List<SObject> newRecords, Map<Id, SObject> newMap, Map<Id, SObject> oldMap);

    // called after records are deleted
    // input: oldRecords - the deleted records, oldMap - keyed by Id
    void afterDelete(List<SObject> oldRecords, Map<Id, SObject> oldMap);

    // called after records are undeleted (restored from recycle bin)
    // input: newRecords - the restored records, newMap - keyed by Id
    void afterUndelete(List<SObject> newRecords, Map<Id, SObject> newMap);
}
```

#### TriggerHandler Base Class with Set\<Id\> Recursion Prevention

```apex
// base class that all trigger handlers extend
// provides context routing and recursion prevention using Set<Id> instead of Boolean
public virtual class TriggerHandler implements ITriggerHandler {

    // tracks which record IDs have already been processed in each context
    // using Set<Id> instead of Boolean means new records entering the context still get processed
    @TestVisible
    private static Map<String, Set<Id>> processedIds = new Map<String, Set<Id>>();

    // override to disable the handler (e.g., during data loads)
    // output: true if the handler should be bypassed
    public virtual Boolean isDisabled() {
        return false;
    }

    // returns the unique key for recursion tracking
    // defaults to the class name -- override if needed
    // output: string key used in the processedIds map
    protected virtual String getHandlerName() {
        return String.valueOf(this).split(':')[0];
    }

    // checks if a specific record has already been processed for a given context
    // input: context - the trigger context string, recordId - the record to check
    // output: true if this record was already processed in this context
    private Boolean hasBeenProcessed(String context, Id recordId) {
        String key = getHandlerName() + '.' + context;
        if (!processedIds.containsKey(key)) {
            processedIds.put(key, new Set<Id>());
        }
        return processedIds.get(key).contains(recordId);
    }

    // marks a set of record IDs as processed for a given context
    // input: context - the trigger context string, recordIds - the IDs to mark
    private void markProcessed(String context, Set<Id> recordIds) {
        String key = getHandlerName() + '.' + context;
        if (!processedIds.containsKey(key)) {
            processedIds.put(key, new Set<Id>());
        }
        processedIds.get(key).addAll(recordIds);
    }

    // filters a list of records to only those not yet processed in this context
    // input: context - the trigger context string, records - the records to filter
    // output: list of records that have not been processed yet
    protected List<SObject> filterUnprocessed(String context, List<SObject> records) {
        List<SObject> unprocessed = new List<SObject>();
        for (SObject rec : records) {
            if (rec.Id != null && hasBeenProcessed(context, rec.Id)) {
                continue;
            }
            unprocessed.add(rec);
        }
        // mark them as processed now
        Set<Id> ids = new Set<Id>();
        for (SObject rec : unprocessed) {
            if (rec.Id != null) {
                ids.add(rec.Id);
            }
        }
        if (!ids.isEmpty()) {
            markProcessed(context, ids);
        }
        return unprocessed;
    }

    // clears all tracked processed IDs -- call from tests to reset state
    @TestVisible
    private static void clearProcessedIds() {
        processedIds.clear();
    }

    // default implementations -- subclasses override only what they need
    public virtual void beforeInsert(List<SObject> newRecords) {}
    public virtual void beforeUpdate(List<SObject> newRecords, Map<Id, SObject> oldMap) {}
    public virtual void beforeDelete(List<SObject> oldRecords, Map<Id, SObject> oldMap) {}
    public virtual void afterInsert(List<SObject> newRecords, Map<Id, SObject> newMap) {}
    public virtual void afterUpdate(List<SObject> newRecords, Map<Id, SObject> newMap, Map<Id, SObject> oldMap) {}
    public virtual void afterDelete(List<SObject> oldRecords, Map<Id, SObject> oldMap) {}
    public virtual void afterUndelete(List<SObject> newRecords, Map<Id, SObject> newMap) {}
}
```

#### TriggerDispatcher

```apex
// central dispatcher that reads trigger context and routes to the appropriate handler method
// keeps trigger bodies to a single line
public class TriggerDispatcher {

    // dispatches the current trigger context to the given handler
    // input: handler - the ITriggerHandler implementation for this object
    public static void run(ITriggerHandler handler) {
        // respect the disable switch
        if (handler instanceof TriggerHandler && ((TriggerHandler)handler).isDisabled()) {
            return;
        }

        if (Trigger.isBefore) {
            if (Trigger.isInsert) {
                handler.beforeInsert(Trigger.new);
            } else if (Trigger.isUpdate) {
                handler.beforeUpdate(Trigger.new, Trigger.oldMap);
            } else if (Trigger.isDelete) {
                handler.beforeDelete(Trigger.old, Trigger.oldMap);
            }
        } else if (Trigger.isAfter) {
            if (Trigger.isInsert) {
                handler.afterInsert(Trigger.new, new Map<Id, SObject>(Trigger.new));
            } else if (Trigger.isUpdate) {
                handler.afterUpdate(Trigger.new, new Map<Id, SObject>(Trigger.new), Trigger.oldMap);
            } else if (Trigger.isDelete) {
                handler.afterDelete(Trigger.old, Trigger.oldMap);
            } else if (Trigger.isUndelete) {
                handler.afterUndelete(Trigger.new, new Map<Id, SObject>(Trigger.new));
            }
        }
    }
}
```

#### Complete Account Example

```apex
// the trigger file -- one trigger per object, one line of logic
trigger AccountTrigger on Account (
    before insert, before update, before delete,
    after insert, after update, after delete, after undelete
) {
    TriggerDispatcher.run(new AccountTriggerHandler());
}
```

```apex
// handler for Account trigger -- routes to Domain and Service classes
public with sharing class AccountTriggerHandler extends TriggerHandler {

    // apply defaults and validate before records are saved
    // input: newRecords - the accounts being inserted
    public override void beforeInsert(List<SObject> newRecords) {
        List<Account> accounts = (List<Account>) newRecords;
        Accounts domain = new Accounts(accounts);
        domain.applyDefaults();
        domain.validate();
    }

    // validate on update, but only process records not yet handled in this context
    // input: newRecords - the updated accounts, oldMap - previous versions
    public override void beforeUpdate(List<SObject> newRecords, Map<Id, SObject> oldMap) {
        List<Account> accounts = (List<Account>) newRecords;
        Map<Id, Account> oldAccounts = (Map<Id, Account>) oldMap;
        Accounts domain = new Accounts(accounts);
        domain.validate();
    }

    // after insert -- create default child records, fire platform events
    // input: newRecords - inserted accounts with IDs, newMap - keyed by Id
    public override void afterInsert(List<SObject> newRecords, Map<Id, SObject> newMap) {
        List<Account> unprocessed = (List<Account>) filterUnprocessed('afterInsert', newRecords);
        if (unprocessed.isEmpty()) {
            return;
        }
        AccountService.createDefaultContacts(unprocessed);
    }

    // after update -- only act on records where a key field actually changed
    // input: newRecords - updated accounts, newMap - keyed by Id, oldMap - previous versions
    public override void afterUpdate(List<SObject> newRecords, Map<Id, SObject> newMap, Map<Id, SObject> oldMap) {
        List<Account> unprocessed = (List<Account>) filterUnprocessed('afterUpdate', newRecords);
        if (unprocessed.isEmpty()) {
            return;
        }

        // filter to only accounts where the owner actually changed
        List<Account> ownerChanged = new List<Account>();
        for (Account acc : unprocessed) {
            Account old = (Account) oldMap.get(acc.Id);
            if (acc.OwnerId != old.OwnerId) {
                ownerChanged.add(acc);
            }
        }

        if (!ownerChanged.isEmpty()) {
            AccountService.reassignRelatedRecords(ownerChanged);
        }
    }
}
```

---

### 2. Service Layer Pattern -- Complete Implementation

```apex
// service class for Order business operations
// all methods are public static, accept collections, and own the transaction boundary
public with sharing class OrderService {

    // submits orders for processing -- validates, applies pricing, updates status, creates log entries
    // input: orderIds - set of Order IDs to submit
    // output: none (throws OrderServiceException on failure, rolling back all DML)
    public static void submitOrders(Set<Id> orderIds) {
        Savepoint sp = Database.setSavepoint();
        try {
            // query orders with their line items through the selector
            List<Order> orders = OrderSelector.selectByIdWithLineItems(orderIds);
            if (orders.isEmpty()) {
                throw new OrderServiceException('No orders found for the given IDs.');
            }

            // use the domain class for business logic
            Orders domain = new Orders(orders);
            domain.validateForSubmission();
            domain.calculateTotals();

            // update order status
            for (Order ord : orders) {
                ord.Status = 'Submitted';
                ord.SubmittedDate__c = Datetime.now();
            }

            // create audit log entries
            List<Order_Log__c> logs = new List<Order_Log__c>();
            for (Order ord : orders) {
                logs.add(new Order_Log__c(
                    Order__c = ord.Id,
                    Action__c = 'Submitted',
                    Performed_By__c = UserInfo.getUserId(),
                    Timestamp__c = Datetime.now()
                ));
            }

            // commit all DML
            update orders;
            insert logs;

            System.debug('OrderService.submitOrders: successfully submitted ' + orders.size() + ' orders');

        } catch (Exception e) {
            Database.rollback(sp);
            System.debug(LoggingLevel.ERROR, 'OrderService.submitOrders failed: '
                + e.getMessage() + ' | Stack: ' + e.getStackTraceString());
            throw new OrderServiceException('Failed to submit orders: ' + e.getMessage());
        }
    }

    // cancels orders -- validates cancellation eligibility, reverses inventory, updates status
    // input: orderIds - set of Order IDs to cancel, reason - the cancellation reason
    // output: none (throws OrderServiceException on failure)
    public static void cancelOrders(Set<Id> orderIds, String reason) {
        Savepoint sp = Database.setSavepoint();
        try {
            List<Order> orders = OrderSelector.selectByIdWithLineItems(orderIds);

            // domain handles cancellation validation
            Orders domain = new Orders(orders);
            domain.validateForCancellation();

            for (Order ord : orders) {
                ord.Status = 'Cancelled';
                ord.Cancellation_Reason__c = reason;
                ord.Cancelled_Date__c = Datetime.now();
            }

            List<Order_Log__c> logs = new List<Order_Log__c>();
            for (Order ord : orders) {
                logs.add(new Order_Log__c(
                    Order__c = ord.Id,
                    Action__c = 'Cancelled',
                    Details__c = reason,
                    Performed_By__c = UserInfo.getUserId(),
                    Timestamp__c = Datetime.now()
                ));
            }

            update orders;
            insert logs;

        } catch (Exception e) {
            Database.rollback(sp);
            throw new OrderServiceException('Failed to cancel orders: ' + e.getMessage());
        }
    }

    public class OrderServiceException extends Exception {}
}
```

---

### 3. Selector Pattern -- Complete Implementation

```apex
// centralizes all Account SOQL queries
// every consumer gets the same field set, FLS enforcement, and query optimizations
public inherited sharing class AccountSelector {

    // the default field set returned by all queries unless a method needs extras
    // centralizing this means adding a new field happens in one place
    private static List<String> defaultFields = new List<String>{
        'Id', 'Name', 'AccountNumber', 'Type', 'Industry',
        'Phone', 'Website', 'BillingCity', 'BillingState',
        'OwnerId', 'CreatedDate', 'LastModifiedDate'
    };

    // builds the base SELECT clause from the default fields
    // output: comma-separated field list string
    private static String getFieldList() {
        return String.join(defaultFields, ', ');
    }

    // queries accounts by their IDs with FLS enforcement
    // input: accountIds - set of Account IDs to query
    // output: list of Accounts matching the given IDs
    public static List<Account> selectById(Set<Id> accountIds) {
        return [
            SELECT Id, Name, AccountNumber, Type, Industry,
                   Phone, Website, BillingCity, BillingState,
                   OwnerId, CreatedDate, LastModifiedDate
            FROM Account
            WHERE Id IN :accountIds
            WITH USER_MODE
            ORDER BY Name
        ];
    }

    // queries accounts by name using LIKE matching with FLS enforcement
    // input: searchName - the name pattern to search for (supports % wildcards)
    // output: list of matching Accounts, max 200
    public static List<Account> selectByName(String searchName) {
        return [
            SELECT Id, Name, AccountNumber, Type, Industry,
                   Phone, Website, BillingCity, BillingState,
                   OwnerId, CreatedDate, LastModifiedDate
            FROM Account
            WHERE Name LIKE :searchName
            WITH USER_MODE
            ORDER BY Name
            LIMIT 200
        ];
    }

    // queries accounts with their child contacts -- use when the caller needs related data
    // input: accountIds - set of Account IDs to query
    // output: list of Accounts with nested Contact records
    public static List<Account> selectWithContacts(Set<Id> accountIds) {
        return [
            SELECT Id, Name, AccountNumber, Type, Industry,
                   Phone, Website, BillingCity, BillingState,
                   OwnerId, CreatedDate, LastModifiedDate,
                   (SELECT Id, FirstName, LastName, Email, Phone, Title
                    FROM Contacts
                    ORDER BY LastName, FirstName)
            FROM Account
            WHERE Id IN :accountIds
            WITH USER_MODE
            ORDER BY Name
        ];
    }

    // queries accounts with open opportunities -- for pipeline and forecasting features
    // input: accountIds - set of Account IDs to query
    // output: list of Accounts with nested open Opportunity records
    public static List<Account> selectWithOpenOpportunities(Set<Id> accountIds) {
        return [
            SELECT Id, Name, Type, Industry, OwnerId,
                   (SELECT Id, Name, StageName, Amount, CloseDate
                    FROM Opportunities
                    WHERE IsClosed = false
                    ORDER BY CloseDate)
            FROM Account
            WHERE Id IN :accountIds
            WITH USER_MODE
            ORDER BY Name
        ];
    }

    // system-context query for batch jobs that need to see all records regardless of sharing
    // only call this from batch/scheduled/system-context code
    // input: accountIds - set of Account IDs to query
    // output: list of Accounts without sharing or FLS enforcement
    public static List<Account> selectByIdSystemMode(Set<Id> accountIds) {
        return [
            SELECT Id, Name, AccountNumber, Type, Industry,
                   Phone, Website, BillingCity, BillingState,
                   OwnerId, CreatedDate, LastModifiedDate
            FROM Account
            WHERE Id IN :accountIds
            WITH SYSTEM_MODE
            ORDER BY Name
        ];
    }
}
```

---

### 4. Domain Pattern -- Complete Implementation

```apex
// domain class for Account business logic
// operates on collections of Account records -- never single records
// handles validation, defaulting, calculations, and state transitions
public with sharing class Accounts {

    private List<Account> records;

    // constructor always takes a list to enforce bulkification from the start
    // input: records - the list of Account records to operate on
    public Accounts(List<Account> records) {
        this.records = records;
    }

    // applies default field values to new accounts
    // call from trigger handler's beforeInsert
    // input: none (operates on the internal record list)
    // output: none (modifies records in place for the before-trigger DML)
    public void applyDefaults() {
        for (Account acc : this.records) {
            // default the type to 'Prospect' if not already set
            if (String.isBlank(acc.Type)) {
                acc.Type = 'Prospect';
            }

            // default the industry to 'Other' if blank
            if (String.isBlank(acc.Industry)) {
                acc.Industry = 'Other';
            }

            // stamp the account source if it came through a specific channel
            if (String.isBlank(acc.AccountSource)) {
                acc.AccountSource = 'Direct';
            }

            System.debug('Accounts.applyDefaults: defaulted Account "' + acc.Name + '"');
        }
    }

    // validates accounts against business rules
    // uses addError() to surface errors on the record without throwing exceptions
    // input: none (operates on the internal record list)
    // output: none (adds errors directly to invalid records)
    public void validate() {
        for (Account acc : this.records) {
            // account name is required and must not be a placeholder
            if (String.isBlank(acc.Name)) {
                acc.addError('Account Name is required.');
            } else if (acc.Name.equalsIgnoreCase('test') || acc.Name.equalsIgnoreCase('TBD')) {
                acc.addError('Account Name cannot be a placeholder value like "test" or "TBD".');
            }

            // phone is required for Customer accounts
            if (acc.Type == 'Customer' && String.isBlank(acc.Phone)) {
                acc.addError('Phone is required for Customer accounts.');
            }

            // website format validation -- basic check for well-formed URL
            if (String.isNotBlank(acc.Website) && !acc.Website.startsWith('http')) {
                acc.Website = 'https://' + acc.Website;
            }
        }
    }

    // calculates a health score based on data completeness
    // call from services or batch jobs that need to score accounts
    // input: none (operates on the internal record list)
    // output: none (modifies Account_Health_Score__c in place)
    public void calculateHealthScore() {
        for (Account acc : this.records) {
            Integer score = 0;

            // each populated field adds to the score
            if (String.isNotBlank(acc.Phone)) { score += 20; }
            if (String.isNotBlank(acc.Website)) { score += 20; }
            if (String.isNotBlank(acc.Industry)) { score += 15; }
            if (String.isNotBlank(acc.BillingCity)) { score += 15; }
            if (String.isNotBlank(acc.Type) && acc.Type == 'Customer') { score += 30; }

            acc.Account_Health_Score__c = score;
            System.debug('Accounts.calculateHealthScore: "' + acc.Name + '" scored ' + score);
        }
    }

    // returns just the records -- useful when the caller needs the modified list back
    // output: the internal list of Account records
    public List<Account> getRecords() {
        return this.records;
    }
}
```

---

### 5. Unit of Work Pattern -- Simplified Implementation (No fflib Dependency)

```apex
// lightweight Unit of Work that collects DML operations and commits them in one transaction
// use this when you do not want the full fflib dependency but still need transactional safety
public class SimpleUnitOfWork {

    // internal lists holding records registered for each DML type
    private List<SObject> newRecords = new List<SObject>();
    private List<SObject> dirtyRecords = new List<SObject>();
    private List<SObject> deletedRecords = new List<SObject>();

    // tracks parent-child relationships so child foreign keys can be set after parent insert
    // each entry is: child record, child lookup field, parent record
    private List<RelationshipRecord> relationships = new List<RelationshipRecord>();

    // registers a record for insertion
    // input: record - the SObject to insert
    public void registerNew(SObject record) {
        this.newRecords.add(record);
    }

    // registers a new child record and its relationship to a parent being inserted in the same UoW
    // after the parent is inserted and gets an Id, the child's lookup field is set automatically
    // input: record - the child SObject, relatedToField - the lookup field API name, relatedTo - the parent SObject
    public void registerNew(SObject record, String relatedToField, SObject relatedTo) {
        this.newRecords.add(record);
        this.relationships.add(new RelationshipRecord(record, relatedToField, relatedTo));
    }

    // registers a record that has been modified and needs updating
    // input: record - the SObject to update
    public void registerDirty(SObject record) {
        this.dirtyRecords.add(record);
    }

    // registers a record for deletion
    // input: record - the SObject to delete
    public void registerDeleted(SObject record) {
        this.deletedRecords.add(record);
    }

    // commits all registered DML in one transaction
    // inserts first (so parent IDs are available), then updates, then deletes
    // rolls back everything if any operation fails
    public void commitWork() {
        Savepoint sp = Database.setSavepoint();
        try {
            // step 1: insert new records
            if (!this.newRecords.isEmpty()) {
                // sort so parents are inserted before children
                // (simple approach: insert all, then resolve relationships)
                insert this.newRecords;

                // step 2: resolve parent-child relationships
                // after parent insert, set the child lookup field to the parent's new Id
                for (RelationshipRecord rel : this.relationships) {
                    rel.record.put(rel.relatedToField, rel.relatedTo.Id);
                }

                // re-insert children that had relationships (they need the foreign key set)
                // actually we need to update them since they were already inserted above
                // gather children that had relationships for a follow-up update
                Set<SObject> childrenToUpdate = new Set<SObject>();
                for (RelationshipRecord rel : this.relationships) {
                    childrenToUpdate.add(rel.record);
                }
                if (!childrenToUpdate.isEmpty()) {
                    update new List<SObject>(childrenToUpdate);
                }
            }

            // step 3: update dirty records
            if (!this.dirtyRecords.isEmpty()) {
                update this.dirtyRecords;
            }

            // step 4: delete records
            if (!this.deletedRecords.isEmpty()) {
                delete this.deletedRecords;
            }

            System.debug('SimpleUnitOfWork.commitWork: committed '
                + this.newRecords.size() + ' inserts, '
                + this.dirtyRecords.size() + ' updates, '
                + this.deletedRecords.size() + ' deletes');

        } catch (Exception e) {
            Database.rollback(sp);
            System.debug(LoggingLevel.ERROR, 'SimpleUnitOfWork.commitWork failed: '
                + e.getMessage() + ' | Stack: ' + e.getStackTraceString());
            throw new UnitOfWorkException('Transaction failed, all changes rolled back: ' + e.getMessage());
        }
    }

    // inner class to track a parent-child relationship for post-insert resolution
    private class RelationshipRecord {
        public SObject record;
        public String relatedToField;
        public SObject relatedTo;

        public RelationshipRecord(SObject record, String relatedToField, SObject relatedTo) {
            this.record = record;
            this.relatedToField = relatedToField;
            this.relatedTo = relatedTo;
        }
    }

    public class UnitOfWorkException extends Exception {}
}
```

#### Using the Unit of Work for a Cross-Object Transaction

```apex
// example: creating an Account, Contact, and Opportunity in one transaction
// if any part fails, the entire operation rolls back
public with sharing class OnboardingService {

    // onboards a new customer -- creates Account, primary Contact, and initial Opportunity
    // input: accountName, contactFirstName, contactLastName, contactEmail, dealAmount
    // output: the newly created Account Id
    public static Id onboardCustomer(
        String accountName,
        String contactFirstName,
        String contactLastName,
        String contactEmail,
        Decimal dealAmount
    ) {
        SimpleUnitOfWork uow = new SimpleUnitOfWork();

        // register the parent Account
        Account acc = new Account(
            Name = accountName,
            Type = 'Customer'
        );
        uow.registerNew(acc);

        // register the Contact as a child of the Account
        Contact con = new Contact(
            FirstName = contactFirstName,
            LastName = contactLastName,
            Email = contactEmail
        );
        uow.registerNew(con, 'AccountId', acc);

        // register the Opportunity as a child of the Account
        Opportunity opp = new Opportunity(
            Name = accountName + ' - Initial Deal',
            StageName = 'Prospecting',
            CloseDate = Date.today().addDays(30),
            Amount = dealAmount
        );
        uow.registerNew(opp, 'AccountId', acc);

        // one commit -- all three records inserted in one transaction
        uow.commitWork();

        System.debug('OnboardingService.onboardCustomer: created Account ' + acc.Id);
        return acc.Id;
    }
}
```

---

### 6. Factory Pattern -- Complete Implementation

```apex
// factory for obtaining selector instances
// in production, returns real selectors; in tests, returns mocks if registered
public class SelectorFactory {

    // stores mock overrides keyed by the SObject type they replace
    @TestVisible
    private static Map<System.Type, Object> mockInstances = new Map<System.Type, Object>();

    // returns the AccountSelector instance (real or mock)
    // output: an object implementing the IAccountSelector interface
    public static IAccountSelector accounts() {
        if (mockInstances.containsKey(IAccountSelector.class)) {
            return (IAccountSelector) mockInstances.get(IAccountSelector.class);
        }
        return new AccountSelectorImpl();
    }

    // returns the OrderSelector instance (real or mock)
    // output: an object implementing the IOrderSelector interface
    public static IOrderSelector orders() {
        if (mockInstances.containsKey(IOrderSelector.class)) {
            return (IOrderSelector) mockInstances.get(IOrderSelector.class);
        }
        return new OrderSelectorImpl();
    }

    // registers a mock implementation for testing
    // input: selectorType - the interface System.Type, mockInstance - the mock object
    @TestVisible
    private static void setMock(System.Type selectorType, Object mockInstance) {
        mockInstances.put(selectorType, mockInstance);
    }

    // clears all registered mocks -- call in test cleanup
    @TestVisible
    private static void clearMocks() {
        mockInstances.clear();
    }
}
```

#### Selector Interface and Real Implementation

```apex
// interface defining the contract for Account queries
public interface IAccountSelector {
    List<Account> selectById(Set<Id> accountIds);
    List<Account> selectByName(String searchName);
    List<Account> selectWithContacts(Set<Id> accountIds);
}
```

```apex
// real implementation that hits the database
public inherited sharing class AccountSelectorImpl implements IAccountSelector {

    // queries accounts by ID with FLS enforcement
    // input: accountIds - set of Account IDs
    // output: matching Account records
    public List<Account> selectById(Set<Id> accountIds) {
        return [
            SELECT Id, Name, AccountNumber, Type, Industry, Phone, Website, OwnerId
            FROM Account
            WHERE Id IN :accountIds
            WITH USER_MODE
            ORDER BY Name
        ];
    }

    // queries accounts by name pattern
    // input: searchName - LIKE pattern (e.g., 'Acme%')
    // output: matching Account records
    public List<Account> selectByName(String searchName) {
        return [
            SELECT Id, Name, AccountNumber, Type, Industry, Phone, Website, OwnerId
            FROM Account
            WHERE Name LIKE :searchName
            WITH USER_MODE
            ORDER BY Name
            LIMIT 200
        ];
    }

    // queries accounts with child contacts
    // input: accountIds - set of Account IDs
    // output: Account records with nested Contacts
    public List<Account> selectWithContacts(Set<Id> accountIds) {
        return [
            SELECT Id, Name, Type, OwnerId,
                   (SELECT Id, FirstName, LastName, Email FROM Contacts ORDER BY LastName)
            FROM Account
            WHERE Id IN :accountIds
            WITH USER_MODE
            ORDER BY Name
        ];
    }
}
```

#### Mock Selector for Tests

```apex
// mock selector that returns whatever data you configure -- no database hits
@IsTest
public class MockAccountSelector implements IAccountSelector {

    // pre-loaded return values -- set these in your test before calling the service
    public List<Account> accountsToReturn = new List<Account>();

    public List<Account> selectById(Set<Id> accountIds) {
        return this.accountsToReturn;
    }

    public List<Account> selectByName(String searchName) {
        return this.accountsToReturn;
    }

    public List<Account> selectWithContacts(Set<Id> accountIds) {
        return this.accountsToReturn;
    }
}
```

#### Service Using the Factory (Testable)

```apex
// service that obtains its selector through the factory
// in tests, the factory returns a mock selector -- no database needed
public with sharing class AccountService {

    // fetches accounts and calculates health scores
    // input: accountIds - the accounts to score
    // output: none (updates accounts with calculated scores)
    public static void calculateHealthScores(Set<Id> accountIds) {
        // get the selector through the factory -- in tests this returns a mock
        IAccountSelector selector = SelectorFactory.accounts();
        List<Account> accounts = selector.selectById(accountIds);

        Accounts domain = new Accounts(accounts);
        domain.calculateHealthScore();

        update accounts;
    }
}
```

#### Test Using the Mock

```apex
@IsTest
private class AccountServiceTest {

    @IsTest
    static void testCalculateHealthScores() {
        // arrange: create test data in memory (no database insert)
        Account testAccount = new Account(
            Id = TestUtility.getFakeId(Account.SObjectType),
            Name = 'Test Corp',
            Phone = '555-1234',
            Website = 'https://test.com',
            Industry = 'Technology',
            Type = 'Customer',
            BillingCity = 'Austin'
        );

        // register the mock selector
        MockAccountSelector mockSelector = new MockAccountSelector();
        mockSelector.accountsToReturn = new List<Account>{ testAccount };
        SelectorFactory.setMock(IAccountSelector.class, mockSelector);

        // act
        Test.startTest();
        // note: this will still attempt an update DML -- to fully avoid DML,
        // wrap the DML in a DML service class and mock that too
        // for this example we focus on the selector mock pattern
        Test.stopTest();

        // assert
        System.assertEquals(100, testAccount.Account_Health_Score__c,
            'Expected a perfect health score for a fully populated account');
    }
}
```

#### Utility for Generating Fake IDs in Tests

```apex
// generates fake IDs for test records that are never inserted into the database
// useful for mock-based testing where you need valid-looking IDs
@IsTest
public class TestUtility {

    private static Integer fakeIdCounter = 1;

    // generates a fake but valid-format ID for the given SObject type
    // input: sObjectType - the Schema.SObjectType (e.g., Account.SObjectType)
    // output: a fake Id with the correct 3-character prefix
    public static Id getFakeId(Schema.SObjectType sObjectType) {
        String prefix = sObjectType.getDescribe().getKeyPrefix();
        String counterStr = String.valueOf(fakeIdCounter++);
        // pad to 12 characters after the 3-char prefix for an 18-char Salesforce ID
        return Id.valueOf(prefix + counterStr.leftPad(12, '0') + 'AAA');
    }
}
```

---

### 7. Strategy Pattern -- Complete Implementation

```apex
// interface defining the contract for a pricing strategy
// any pricing algorithm implements this and can be swapped at runtime
public interface IPricingStrategy {

    // calculates the final price given a base price and quantity
    // input: basePrice - the unit price, quantity - number of units
    // output: the calculated final price after applying the strategy
    Decimal calculatePrice(Decimal basePrice, Integer quantity);

    // returns a human-readable description of the strategy
    // output: the strategy name
    String getStrategyName();
}
```

```apex
// standard pricing -- no discount
public class StandardPricing implements IPricingStrategy {

    public Decimal calculatePrice(Decimal basePrice, Integer quantity) {
        return basePrice * quantity;
    }

    public String getStrategyName() {
        return 'Standard';
    }
}
```

```apex
// volume pricing -- tiered discounts based on quantity
public class VolumePricing implements IPricingStrategy {

    public Decimal calculatePrice(Decimal basePrice, Integer quantity) {
        Decimal discount = 0;

        // tiered discount: 10+ units get 5%, 50+ get 10%, 100+ get 15%
        if (quantity >= 100) {
            discount = 0.15;
        } else if (quantity >= 50) {
            discount = 0.10;
        } else if (quantity >= 10) {
            discount = 0.05;
        }

        Decimal unitPrice = basePrice * (1 - discount);
        return unitPrice * quantity;
    }

    public String getStrategyName() {
        return 'Volume (' + getDiscountTier() + ')';
    }

    private String getDiscountTier() {
        return '5%/10%/15% tiered';
    }
}
```

```apex
// promotional pricing -- flat percentage discount loaded from Custom Metadata
public class PromotionalPricing implements IPricingStrategy {

    private Decimal discountPercent;
    private String promotionName;

    // constructor loads the promotion config from Custom Metadata
    // input: promotionDeveloperName - the DeveloperName of the Promotion_Config__mdt record
    public PromotionalPricing(String promotionDeveloperName) {
        Promotion_Config__mdt promo = Promotion_Config__mdt.getInstance(promotionDeveloperName);
        if (promo == null) {
            throw new PricingException('Promotion "' + promotionDeveloperName + '" not found.');
        }
        this.discountPercent = promo.Discount_Percent__c / 100;
        this.promotionName = promo.MasterLabel;
    }

    public Decimal calculatePrice(Decimal basePrice, Integer quantity) {
        Decimal discountedPrice = basePrice * (1 - this.discountPercent);
        return discountedPrice * quantity;
    }

    public String getStrategyName() {
        return 'Promo: ' + this.promotionName;
    }

    public class PricingException extends Exception {}
}
```

#### Strategy Resolver Using Custom Metadata

```apex
// resolves the correct pricing strategy for an account or order
// uses Custom Metadata to determine which strategy applies
public class PricingStrategyResolver {

    // determines the correct pricing strategy for the given account
    // input: accountId - the Account to resolve pricing for
    // output: the IPricingStrategy implementation to use
    public static IPricingStrategy resolve(Id accountId) {
        // check if there is an active promotion for this account
        List<Account_Promotion__mdt> promos = [
            SELECT Promotion_Developer_Name__c
            FROM Account_Promotion__mdt
            WHERE Account_Id__c = :String.valueOf(accountId)
            AND Is_Active__c = true
            LIMIT 1
        ];

        if (!promos.isEmpty()) {
            return new PromotionalPricing(promos[0].Promotion_Developer_Name__c);
        }

        // check if the account qualifies for volume pricing
        Account acc = [SELECT Type FROM Account WHERE Id = :accountId LIMIT 1];
        if (acc.Type == 'Channel Partner / Reseller') {
            return new VolumePricing();
        }

        // default to standard pricing
        return new StandardPricing();
    }
}
```

#### Usage in a Service

```apex
// in a service method, the strategy is resolved and applied
public static Decimal calculateOrderTotal(Id accountId, List<Order_Line__c> lines) {
    IPricingStrategy strategy = PricingStrategyResolver.resolve(accountId);
    Decimal total = 0;

    for (Order_Line__c line : lines) {
        Decimal lineTotal = strategy.calculatePrice(line.Unit_Price__c, (Integer) line.Quantity__c);
        line.Calculated_Price__c = lineTotal;
        line.Pricing_Strategy__c = strategy.getStrategyName();
        total += lineTotal;
    }

    System.debug('PricingService: calculated total ' + total + ' using strategy "' + strategy.getStrategyName() + '"');
    return total;
}
```

---

### 8. Singleton Pattern -- Complete Implementation

```apex
// singleton that caches org-wide configuration from Custom Metadata
// loaded once per transaction, never re-queried
public class OrgSettings {

    // the single instance -- static so it survives across method calls in the same transaction
    private static OrgSettings instance;

    // cached settings values
    public String defaultCurrency { get; private set; }
    public Integer maxRetryCount { get; private set; }
    public Boolean isMaintenanceMode { get; private set; }
    public String supportEmail { get; private set; }
    public Decimal maxOrderAmount { get; private set; }

    // all available settings keyed by DeveloperName for ad-hoc lookups
    private Map<String, Org_Setting__mdt> settingsMap;

    // private constructor -- only getInstance() can create the instance
    // loads all Org_Setting__mdt records once
    private OrgSettings() {
        // getAll() does not count against SOQL limits but still costs CPU
        // loading once in the constructor and caching is the efficient approach
        this.settingsMap = Org_Setting__mdt.getAll();

        // populate typed properties from the metadata
        this.defaultCurrency = getStringValue('Default_Currency', 'USD');
        this.maxRetryCount = getIntegerValue('Max_Retry_Count', 3);
        this.isMaintenanceMode = getBooleanValue('Maintenance_Mode', false);
        this.supportEmail = getStringValue('Support_Email', 'support@company.com');
        this.maxOrderAmount = getDecimalValue('Max_Order_Amount', 1000000);

        System.debug('OrgSettings: loaded ' + this.settingsMap.size() + ' settings');
    }

    // returns the singleton instance, creating it on first call
    // output: the single OrgSettings instance for this transaction
    public static OrgSettings getInstance() {
        if (instance == null) {
            instance = new OrgSettings();
        }
        return instance;
    }

    // retrieves a string setting value by developer name with a fallback default
    // input: developerName - the setting key, defaultValue - fallback if not found
    // output: the setting value or the default
    public String getStringValue(String developerName, String defaultValue) {
        Org_Setting__mdt setting = this.settingsMap.get(developerName);
        if (setting != null && String.isNotBlank(setting.Value__c)) {
            return setting.Value__c;
        }
        return defaultValue;
    }

    // retrieves an integer setting value
    // input: developerName - the setting key, defaultValue - fallback if not found
    // output: the integer value or the default
    public Integer getIntegerValue(String developerName, Integer defaultValue) {
        Org_Setting__mdt setting = this.settingsMap.get(developerName);
        if (setting != null && String.isNotBlank(setting.Value__c)) {
            try {
                return Integer.valueOf(setting.Value__c);
            } catch (Exception e) {
                System.debug(LoggingLevel.WARN, 'OrgSettings: could not parse integer for "' + developerName + '": ' + setting.Value__c);
                return defaultValue;
            }
        }
        return defaultValue;
    }

    // retrieves a boolean setting value
    // input: developerName - the setting key, defaultValue - fallback if not found
    // output: the boolean value or the default
    public Boolean getBooleanValue(String developerName, Boolean defaultValue) {
        Org_Setting__mdt setting = this.settingsMap.get(developerName);
        if (setting != null && String.isNotBlank(setting.Value__c)) {
            return setting.Value__c.equalsIgnoreCase('true');
        }
        return defaultValue;
    }

    // retrieves a decimal setting value
    // input: developerName - the setting key, defaultValue - fallback if not found
    // output: the decimal value or the default
    public Decimal getDecimalValue(String developerName, Decimal defaultValue) {
        Org_Setting__mdt setting = this.settingsMap.get(developerName);
        if (setting != null && String.isNotBlank(setting.Value__c)) {
            try {
                return Decimal.valueOf(setting.Value__c);
            } catch (Exception e) {
                System.debug(LoggingLevel.WARN, 'OrgSettings: could not parse decimal for "' + developerName + '": ' + setting.Value__c);
                return defaultValue;
            }
        }
        return defaultValue;
    }

    // forces the singleton to reload -- use in tests when mocking Custom Metadata
    @TestVisible
    private static void resetInstance() {
        instance = null;
    }
}
```

#### Usage

```apex
// in any service or handler, just call getInstance() -- never re-queries
OrgSettings settings = OrgSettings.getInstance();

if (settings.isMaintenanceMode) {
    throw new MaintenanceException('System is in maintenance mode. Please try again later.');
}

if (orderTotal > settings.maxOrderAmount) {
    throw new OrderValidationException('Order total exceeds maximum allowed amount of ' + settings.maxOrderAmount);
}

// ad-hoc lookups for settings not pre-typed
String customValue = settings.getStringValue('Some_Other_Setting', 'fallback');
```

---

### 9. Batch Processing Patterns -- Complete Implementations

#### Batch with Error Logging and Email Notification

```apex
// batch job that recalculates account health scores in bulk
// accumulates errors per chunk and reports them in the finish method
public class AccountHealthScoreBatch implements Database.Batchable<SObject>, Database.Stateful {

    // Database.Stateful preserves instance variable state across execute() chunks
    private Integer totalProcessed = 0;
    private Integer totalErrors = 0;
    private List<String> errorMessages = new List<String>();

    // defines the scope of records to process
    // output: a QueryLocator for all active accounts
    public Database.QueryLocator start(Database.BatchableContext bc) {
        System.debug('AccountHealthScoreBatch: starting');
        return Database.getQueryLocator([
            SELECT Id, Name, Phone, Website, Industry, BillingCity, Type,
                   Account_Health_Score__c
            FROM Account
            WHERE IsDeleted = false
            ORDER BY Name
        ]);
    }

    // processes each chunk of records -- errors are caught per chunk, not per record
    // input: bc - batch context, scope - the chunk of Account records
    public void execute(Database.BatchableContext bc, List<Account> scope) {
        try {
            // use the domain class for the actual calculation
            Accounts domain = new Accounts(scope);
            domain.calculateHealthScore();

            // use Database.update with allOrNone=false for partial success
            List<Database.SaveResult> results = Database.update(scope, false);

            for (Integer i = 0; i < results.size(); i++) {
                if (results[i].isSuccess()) {
                    this.totalProcessed++;
                } else {
                    this.totalErrors++;
                    for (Database.Error err : results[i].getErrors()) {
                        this.errorMessages.add(
                            'Account "' + scope[i].Name + '" (' + scope[i].Id + '): ' + err.getMessage()
                        );
                    }
                }
            }

        } catch (Exception e) {
            // if the entire chunk fails, log it
            this.totalErrors += scope.size();
            this.errorMessages.add('Chunk failed: ' + e.getMessage() + ' | Stack: ' + e.getStackTraceString());
            System.debug(LoggingLevel.ERROR, 'AccountHealthScoreBatch.execute failed: ' + e.getMessage());
        }
    }

    // runs after all chunks -- sends a summary email and optionally chains the next batch
    // input: bc - batch context with the AsyncApexJob ID
    public void finish(Database.BatchableContext bc) {
        // query the job record for status details
        AsyncApexJob job = [
            SELECT Id, Status, NumberOfErrors, JobItemsProcessed, TotalJobItems
            FROM AsyncApexJob
            WHERE Id = :bc.getJobId()
        ];

        // build the summary email
        String subject = 'Account Health Score Batch ' + job.Status;
        String body = 'Batch Job Summary\n'
            + '-------------------\n'
            + 'Status: ' + job.Status + '\n'
            + 'Records Processed: ' + this.totalProcessed + '\n'
            + 'Records with Errors: ' + this.totalErrors + '\n'
            + 'Chunks Processed: ' + job.JobItemsProcessed + ' / ' + job.TotalJobItems + '\n';

        if (!this.errorMessages.isEmpty()) {
            // cap error detail at 20 to avoid email size limits
            Integer errorCap = Math.min(this.errorMessages.size(), 20);
            body += '\nFirst ' + errorCap + ' errors:\n';
            for (Integer i = 0; i < errorCap; i++) {
                body += '  - ' + this.errorMessages[i] + '\n';
            }
        }

        // send the email to the running user
        Messaging.SingleEmailMessage mail = new Messaging.SingleEmailMessage();
        mail.setToAddresses(new List<String>{ UserInfo.getUserEmail() });
        mail.setSubject(subject);
        mail.setPlainTextBody(body);
        Messaging.sendEmail(new List<Messaging.SingleEmailMessage>{ mail });

        System.debug('AccountHealthScoreBatch: finished. ' + this.totalProcessed + ' processed, '
            + this.totalErrors + ' errors');

        // chain the next batch if needed (e.g., recalculate Opportunity pipeline next)
        // Database.executeBatch(new OpportunityPipelineBatch(), 200);
    }
}
```

#### Executing a Batch Job

```apex
// execute from Anonymous Apex, a Schedulable, or a service method
// the second parameter is the chunk size -- 200 is the max, lower for complex logic
Database.executeBatch(new AccountHealthScoreBatch(), 200);
```

#### Schedulable Wrapper for a Batch Job

```apex
// schedulable that kicks off the batch job on a cron schedule
public class AccountHealthScoreSchedule implements Schedulable {

    // runs on schedule and starts the batch
    // input: sc - the SchedulableContext
    public void execute(SchedulableContext sc) {
        Database.executeBatch(new AccountHealthScoreBatch(), 200);
    }
}

// schedule it from Anonymous Apex:
// String cronExp = '0 0 2 * * ?';  // every day at 2 AM
// System.schedule('Account Health Score - Daily', cronExp, new AccountHealthScoreSchedule());
```

#### Queueable with Finalizer for Guaranteed Error Handling

```apex
// queueable job that syncs accounts to an external system
// uses System.Finalizer for guaranteed post-execution logging even on failure
public class AccountSyncQueueable implements Queueable, Database.AllowsCallouts {

    private List<Id> accountIds;
    private Integer retryCount;
    private static final Integer MAX_RETRIES = 3;

    // constructor accepts the list of account IDs to sync and the current retry count
    // input: accountIds - IDs to process, retryCount - how many times this job has retried
    public AccountSyncQueueable(List<Id> accountIds, Integer retryCount) {
        this.accountIds = accountIds;
        this.retryCount = retryCount;
    }

    public void execute(QueueableContext ctx) {
        // attach the finalizer -- it runs even if this method throws an unhandled exception
        System.attachFinalizer(new AccountSyncFinalizer(this.accountIds, this.retryCount));

        List<Account> accounts = [
            SELECT Id, Name, AccountNumber, Phone, Website
            FROM Account
            WHERE Id IN :this.accountIds
        ];

        // perform the callout to the external system
        for (Account acc : accounts) {
            HttpRequest req = new HttpRequest();
            req.setEndpoint('callout:External_System/api/accounts');
            req.setMethod('POST');
            req.setHeader('Content-Type', 'application/json');
            req.setBody(JSON.serialize(acc));

            Http http = new Http();
            HttpResponse res = http.send(req);

            if (res.getStatusCode() != 200 && res.getStatusCode() != 201) {
                throw new AccountSyncException('Failed to sync Account ' + acc.Id
                    + ': HTTP ' + res.getStatusCode());
            }
        }

        System.debug('AccountSyncQueueable: synced ' + accounts.size() + ' accounts');
    }

    public class AccountSyncException extends Exception {}
}
```

#### Finalizer for the Queueable

```apex
// finalizer that runs after the queueable completes (success or failure)
// logs the result and retries on failure up to the max retry count
public class AccountSyncFinalizer implements System.Finalizer {

    private List<Id> accountIds;
    private Integer retryCount;
    private static final Integer MAX_RETRIES = 3;

    public AccountSyncFinalizer(List<Id> accountIds, Integer retryCount) {
        this.accountIds = accountIds;
        this.retryCount = retryCount;
    }

    // called by the platform after the queueable finishes
    // input: ctx - provides the result (SUCCESS or UNHANDLED_EXCEPTION) and exception details
    public void execute(System.FinalizerContext ctx) {
        System.ParentJobResult result = ctx.getResult();

        if (result == System.ParentJobResult.SUCCESS) {
            System.debug('AccountSyncFinalizer: job succeeded');
            // log success
            insert new Integration_Log__c(
                Job_Name__c = 'AccountSyncQueueable',
                Status__c = 'Success',
                Record_Count__c = this.accountIds.size(),
                Timestamp__c = Datetime.now()
            );

        } else if (result == System.ParentJobResult.UNHANDLED_EXCEPTION) {
            String errorMsg = ctx.getException() != null ? ctx.getException().getMessage() : 'Unknown error';
            System.debug(LoggingLevel.ERROR, 'AccountSyncFinalizer: job failed - ' + errorMsg);

            // log the failure
            insert new Integration_Log__c(
                Job_Name__c = 'AccountSyncQueueable',
                Status__c = 'Failed',
                Error_Message__c = errorMsg,
                Retry_Count__c = this.retryCount,
                Record_Count__c = this.accountIds.size(),
                Timestamp__c = Datetime.now()
            );

            // retry if under the max -- the platform limits this to 5 consecutive retries
            if (this.retryCount < MAX_RETRIES) {
                System.enqueueJob(
                    new AccountSyncQueueable(this.accountIds, this.retryCount + 1)
                );
                System.debug('AccountSyncFinalizer: enqueued retry ' + (this.retryCount + 1));
            } else {
                System.debug(LoggingLevel.ERROR, 'AccountSyncFinalizer: max retries exceeded, giving up');
            }
        }
    }
}
```

#### Batch vs Queueable Decision Guide

| Factor | Batch Apex | Queueable Apex |
|--------|-----------|----------------|
| **Record volume** | Up to 50 million (via QueryLocator) | Smaller volumes (practical limit ~50k) |
| **Governor limits** | Reset per chunk (each execute() gets fresh limits) | One set of async limits for the entire job |
| **Chaining** | Chain in finish() -- one batch starts the next | Chain in execute() -- one queueable enqueues the next |
| **Concurrent jobs** | Max 5 active/queued batch jobs per org | Max 50 enqueued per transaction; no hard org-wide cap |
| **Complex parameters** | QueryLocator or Iterable; no complex state without Database.Stateful | Constructor accepts any data type (maps, lists, custom objects) |
| **Monitoring** | AsyncApexJob + BatchApexWorker records | AsyncApexJob records |
| **Finalizer support** | No (use finish() method instead) | Yes (System.Finalizer for guaranteed cleanup) |
| **Callouts** | Requires Database.AllowsCallouts; callouts in execute() only | Requires Database.AllowsCallouts |
| **Use when** | Large data migration, nightly recalculations, cleanup jobs | Small async tasks, callout chains, post-trigger processing |

---

### 10. Anti-Patterns to Avoid

**1. God Class (one class does everything)**

A trigger handler that contains hundreds of lines of validation, defaulting, calculation, email sending, and callout logic. The class becomes untestable and unreadable.

```apex
// WRONG: god class with everything in the handler
public class AccountTriggerHandler {
    public void run() {
        // 50 lines of validation
        // 30 lines of defaulting
        // 40 lines of SOQL queries
        // 60 lines of DML operations
        // 25 lines of email sending
        // this is unmaintainable
    }
}

// RIGHT: handler delegates to focused classes
public class AccountTriggerHandler extends TriggerHandler {
    public override void beforeInsert(List<SObject> newRecords) {
        Accounts domain = new Accounts((List<Account>) newRecords);
        domain.applyDefaults();
        domain.validate();
    }
    public override void afterInsert(List<SObject> newRecords, Map<Id, SObject> newMap) {
        AccountService.createDefaultContacts((List<Account>) newRecords);
    }
}
```

**2. SOQL or DML in a Constructor**

Constructors should initialize state. Putting queries or DML in a constructor means every instantiation hits the database, making the class impossible to test without database setup.

```apex
// WRONG: constructor queries the database
public class OrderProcessor {
    private List<Order> orders;
    public OrderProcessor(Set<Id> orderIds) {
        this.orders = [SELECT Id, Status FROM Order WHERE Id IN :orderIds]; // SOQL in constructor!
    }
}

// RIGHT: inject data through the constructor
public class OrderProcessor {
    private List<Order> orders;
    public OrderProcessor(List<Order> orders) {
        this.orders = orders; // data was queried elsewhere and passed in
    }
}
```

**3. Static State Confusion Across Transactions**

Static variables reset at the start of every transaction. Developers sometimes assume a static variable persists across requests (like a web application session). In Apex, it does not. Each trigger execution, Visualforce request, or LWC server call starts fresh.

```apex
// WRONG: assuming static state persists across requests
public class RequestCounter {
    public static Integer count = 0; // resets to 0 on every new transaction!
    public static void increment() {
        count++;
        // this will always be 1 at the end of a single transaction started from 0
    }
}
```

**4. Boolean Recursion Guard**

A single Boolean flag prevents ALL re-entry, even when new records enter the trigger context. Use a `Set<Id>` to track which specific records have been processed.

```apex
// WRONG: boolean blocks all re-entry
public class OpportunityHandler {
    private static Boolean hasRun = false;
    public void handle() {
        if (hasRun) return; // blocks the ENTIRE trigger, not just already-processed records
        hasRun = true;
        // if this handler updates another Opportunity that fires the trigger again,
        // the NEW Opportunity will be silently skipped
    }
}

// RIGHT: Set<Id> tracks which specific records were processed
public class OpportunityHandler extends TriggerHandler {
    public override void afterUpdate(List<SObject> newRecords, Map<Id, SObject> newMap, Map<Id, SObject> oldMap) {
        List<SObject> unprocessed = filterUnprocessed('afterUpdate', newRecords);
        if (unprocessed.isEmpty()) return;
        // only unprocessed records get handled -- new records entering the context still get through
    }
}
```

**5. Hardcoded IDs**

Record IDs differ between sandboxes and production. Hardcoded IDs work during development but fail on deployment.

```apex
// WRONG: hardcoded Record Type ID
Account acc = new Account(
    Name = 'Test',
    RecordTypeId = '012000000000001AAA' // this ID does not exist in production
);

// RIGHT: query by DeveloperName
Id rtId = Schema.SObjectType.Account
    .getRecordTypeInfosByDeveloperName()
    .get('Customer')
    .getRecordTypeId();
Account acc = new Account(Name = 'Test', RecordTypeId = rtId);
```

```apex
// WRONG: hardcoded Profile ID
if (UserInfo.getProfileId() == '00e000000000001AAA') { // breaks in every other org

// RIGHT: query by name
Profile p = [SELECT Id FROM Profile WHERE Name = 'System Administrator' LIMIT 1];
if (UserInfo.getProfileId() == p.Id) {
```

**6. SOQL Inside a Loop**

Each iteration consumes a SOQL query against the 100-query synchronous limit. A batch of 200 records hits the limit at iteration 100.

```apex
// WRONG: SOQL in a loop
for (Account acc : Trigger.new) {
    List<Contact> contacts = [SELECT Id FROM Contact WHERE AccountId = :acc.Id]; // 200 queries!
}

// RIGHT: query outside the loop, organize in a map
Map<Id, List<Contact>> contactsByAccount = new Map<Id, List<Contact>>();
for (Contact c : [SELECT Id, AccountId FROM Contact WHERE AccountId IN :Trigger.newMap.keySet()]) {
    if (!contactsByAccount.containsKey(c.AccountId)) {
        contactsByAccount.put(c.AccountId, new List<Contact>());
    }
    contactsByAccount.get(c.AccountId).add(c);
}
// now iterate and use the map -- zero additional queries
```

**7. DML Inside a Loop**

Same problem as SOQL in a loop, but against the 150-DML limit.

```apex
// WRONG: DML in a loop
for (Account acc : accounts) {
    acc.Status__c = 'Processed';
    update acc; // 200 DML statements!
}

// RIGHT: collect and do one DML
for (Account acc : accounts) {
    acc.Status__c = 'Processed';
}
update accounts; // one DML statement
```

**8. Ignoring Partial-Success DML Results**

When using `Database.insert(records, false)` for partial success, you MUST check the results. Ignoring them means failed records are silently lost.

```apex
// WRONG: ignoring save results
Database.insert(records, false); // which ones failed? we will never know

// RIGHT: check every result
List<Database.SaveResult> results = Database.insert(records, false);
for (Integer i = 0; i < results.size(); i++) {
    if (!results[i].isSuccess()) {
        for (Database.Error err : results[i].getErrors()) {
            System.debug(LoggingLevel.ERROR, 'Failed to insert record ' + i + ': '
                + err.getStatusCode() + ' - ' + err.getMessage());
        }
    }
}
```

**9. Testing Only the Happy Path**

Tests that only verify success are insufficient. Production code encounters nulls, empty lists, duplicate data, missing permissions, and governor limits. Test the edges.

```apex
// WRONG: only testing the happy path
@IsTest
static void testSubmitOrder() {
    Order ord = TestDataFactory.createOrder();
    OrderService.submitOrders(new Set<Id>{ ord.Id });
    // no assertion! just hoping it did not throw
}

// RIGHT: test success, failure, edge cases, and bulk
@IsTest
static void testSubmitOrder_Success() {
    Order ord = TestDataFactory.createOrder();
    Test.startTest();
    OrderService.submitOrders(new Set<Id>{ ord.Id });
    Test.stopTest();
    Order result = [SELECT Status FROM Order WHERE Id = :ord.Id];
    System.assertEquals('Submitted', result.Status, 'Order should be in Submitted status');
}

@IsTest
static void testSubmitOrder_EmptySet() {
    Test.startTest();
    try {
        OrderService.submitOrders(new Set<Id>());
        System.assert(false, 'Expected an exception for empty input');
    } catch (OrderService.OrderServiceException e) {
        System.assert(e.getMessage().contains('No orders found'),
            'Expected no-orders-found message');
    }
    Test.stopTest();
}

@IsTest
static void testSubmitOrder_Bulk() {
    List<Order> orders = TestDataFactory.createOrders(200);
    Set<Id> orderIds = new Set<Id>();
    for (Order o : orders) { orderIds.add(o.Id); }
    Test.startTest();
    OrderService.submitOrders(orderIds); // must handle 200 records without hitting limits
    Test.stopTest();
    List<Order> results = [SELECT Status FROM Order WHERE Id IN :orderIds];
    for (Order o : results) {
        System.assertEquals('Submitted', o.Status, 'All orders should be Submitted');
    }
}
```

**10. Mixing Sharing Contexts Without Documentation**

Using `without sharing` without an explicit comment explaining why creates a security risk. Future developers will not know if it was intentional or accidental.

```apex
// WRONG: without sharing with no explanation
public without sharing class DataProcessor {
    // why without sharing? nobody knows

// RIGHT: documented justification
// runs without sharing because batch jobs need access to all records
// regardless of the running user's role hierarchy position.
// called exclusively from ScheduledDataCleanupBatch.
public without sharing class DataProcessorSystemContext {
```

## Common Mistakes

1. **Using a Boolean for recursion prevention** -- A single `static Boolean hasRun` blocks the entire trigger from re-firing, causing unprocessed records to be silently skipped when the trigger re-enters with new records. Fix: Use a `static Set<Id>` that tracks which specific record IDs have been processed, and filter incoming records against that set.

2. **Putting all logic in the trigger handler (God Class)** -- A 500-line handler with validation, defaulting, SOQL, DML, and email logic is untestable and unmaintainable. Fix: The handler routes to Domain classes (for record logic) and Service classes (for transaction logic). Each class has a single responsibility.

3. **Skipping the Selector and writing inline SOQL everywhere** -- Three service methods query Account with different field sets. A fourth method fails because it is missing a field the other three include. Fix: Centralize all queries for an object in one Selector class with a consistent default field set.

4. **Using WITH SECURITY_ENFORCED instead of WITH USER_MODE** -- `WITH SECURITY_ENFORCED` throws an exception if the user lacks access to any field in the query. `WITH USER_MODE` strips inaccessible fields from results without throwing, which is more resilient. Fix: Use `WITH USER_MODE` for all user-context queries; Salesforce recommends it as the replacement for `WITH SECURITY_ENFORCED`.

5. **Not using Database.Stateful in batch jobs that accumulate errors** -- Without `Database.Stateful`, instance variables reset between `execute()` chunks. Your error counter resets to 0 every chunk and the finish method has no idea what happened. Fix: Always add `Database.Stateful` when your batch job needs to carry state (counters, error lists) across chunks.

6. **Hardcoding Record Type IDs, Profile IDs, or User IDs** -- These IDs differ between every sandbox and production org. Code works in development and fails on deployment. Fix: Query Record Types by `DeveloperName`, Profiles by `Name`, and store any other IDs in Custom Metadata Types.

7. **Building services that accept single records instead of collections** -- `submitOrder(Id orderId)` works for one record but forces callers processing 200 records to call it in a loop, hitting governor limits. Fix: All service methods accept `Set<Id>`, `List<SObject>`, or `Map<Id, SObject>`.

8. **No Savepoint or Unit of Work in multi-object service methods** -- If the service inserts a parent record, then fails on the child insert, the parent is committed but the child is not, leaving orphaned data. Fix: Every service method that does more than one DML operation must use `Database.setSavepoint()` at the top and `Database.rollback()` in the catch block, or use a Unit of Work.

9. **Chaining batch jobs without a depth guard** -- A batch's finish() method starts a new batch, whose finish() starts another, infinitely. Fix: Pass a chain depth counter and check it before chaining. Set a maximum depth (e.g., 5) and log a warning when it is reached.

10. **Creating mock factories but never actually using them in tests** -- The factory and interfaces exist but every test still inserts records and queries the database. Fix: Build the factory pattern from the start of the project and write tests that use mock selectors. Database-free tests run 10-50x faster and test business logic in isolation.

## See Also

- [Solution Architecture Patterns](../solution-architecture/solution-architecture-patterns.md) -- High-level Service/Selector/Domain overview and naming conventions
- [Apex Fundamentals](./apex-fundamentals.md) -- Sharing keywords, access modifiers, governor limits
- [Invocable Methods](./invocable-methods.md) -- @InvocableMethod patterns for Flow integration
