# Invocable Methods -- Flow-to-Apex Bridge

> **When to read this:** You need to call Apex from a Flow, expose Apex logic to Flow Builder, or understand the `@InvocableMethod` and `@InvocableVariable` annotations, bulkification requirements, and error handling patterns.

## Rules

- When writing an `@InvocableMethod`, always accept `List<Request>` and return `List<Result>` using inner class wrappers -- even if the current use case only processes one record. Flow batches multiple interviews into a single Apex call, and a method that assumes a single input will break in bulk scenarios.
- Never put more than one `@InvocableMethod` per class. Salesforce enforces this at compile time. One class, one invocable method. If you need multiple invocable actions, create multiple classes.
- Always annotate the method with `label`, `description`, and `category` so admins can find it in Flow Builder. A method without a label shows its Apex method name, which is meaningless to admins.
- When the invocable method makes HTTP callouts (including `getContentAsPDF()` in some contexts), always add `callout=true` to the annotation: `@InvocableMethod(callout=true ...)`. Without this, the flow engine may not allocate a callout-capable transaction, and the callout will fail at runtime.
- Never perform SOQL or DML inside a loop within the invocable method. The method receives a list -- iterate once, batch your queries, batch your DML. Governor limits are shared with the calling flow's transaction.
- When returning results from an invocable method, always return exactly the same number of Result items as the number of Request items received. The flow maps outputs back to interviews by list index. A size mismatch causes a runtime error.
- Always use `with sharing` on invocable method classes by default. The method runs in the context of the running user. Only use `without sharing` when you have an explicit business reason (e.g., querying records the user cannot see) and document why.
- When handling errors, always catch exceptions inside the invocable method and return error information in the Result wrapper rather than letting the exception propagate -- unless you intentionally want the flow to hit its fault path. Unhandled exceptions roll back the entire transaction.
- Never assume the invocable method runs in its own transaction. It shares the calling flow's transaction context, including all governor limits already consumed by prior flow elements (Get Records, Create Records, other Apex actions, etc.).
- When testing invocable methods, always call the static method directly with a constructed `List<Request>`. Do not try to test by running a flow -- unit tests should isolate the Apex logic.

## How It Works

### The Annotation

`@InvocableMethod` marks a single static method in a class as callable from Flow Builder, Process Builder, the REST API, and Agentforce. The method must be:

- `public` or `global`
- `static`
- Accept zero or one parameter (if one, it must be a `List<>`)
- Return `void` or a `List<>`

The parameter and return types can be primitives (`List<String>`, `List<Id>`), SObjects (`List<Account>`), or custom inner classes annotated with `@InvocableVariable`.

### The Inner Class Pattern

For any non-trivial invocable method, use the **Request/Result inner class pattern**. This is the standard Salesforce pattern and the only way to pass multiple distinct inputs/outputs between Flow and Apex.

```
Flow Interview 1 ──> Request { poId, contactId }  ─┐
Flow Interview 2 ──> Request { poId, contactId }  ─┤──> List<Request> ──> @InvocableMethod ──> List<Result>
Flow Interview 3 ──> Request { poId, contactId }  ─┘
```

Each `@InvocableVariable` on the inner class becomes a named input or output in Flow Builder. The `label` and `description` on each variable control what the admin sees.

### Bulkification

This is the most commonly misunderstood aspect of invocable methods. When a record-triggered flow fires for 200 records (e.g., a bulk data load), Salesforce batches all 200 flow interviews into a single Apex call. Your method receives a `List<Request>` with 200 elements.

If your method does one SOQL query per request inside a loop, you hit the 100 SOQL query limit at record 101. If it does one DML per request, you hit the 150 DML limit at record 151.

The correct pattern: collect all IDs from the request list, query once, process in memory, DML once.

### Governor Limits Context

The invocable method shares the calling transaction's governor limits. If the flow already consumed 40 SOQL queries before calling your Apex action, you have 60 remaining (out of the synchronous limit of 100). Key limits to watch:

| Limit | Synchronous Cap | What Consumes It |
|-------|----------------|-----------------|
| SOQL queries | 100 | Every `[SELECT ...]` or `Database.query()` |
| DML statements | 150 | Every `insert`, `update`, `delete`, `upsert` |
| Query rows | 50,000 | Total records returned across all SOQL |
| DML rows | 10,000 | Total records inserted/updated/deleted |
| Callouts | 100 | HTTP requests, `getContentAsPDF()`, web service calls |
| Single emails | 10 per invocation | `Messaging.sendEmail()` calls |
| CPU time | 10,000 ms | All Apex execution in the transaction |
| Heap size | 6 MB (sync) / 12 MB (async) | All variables, collections, blobs in memory |

`getContentAsPDF()` counts as a callout in some contexts. `Messaging.sendEmail()` has both a per-transaction limit (10 single emails per `sendEmail()` call) and a daily org-wide limit (5,000 single emails per day, or the number of user licenses x 15, whichever is lower).

### The callout=true Parameter

If your invocable method makes any HTTP callout -- including `PageReference.getContentAsPDF()`, `HttpRequest`, or web service calls -- you must declare it:

```apex
@InvocableMethod(callout=true label='Send PO PDF' description='Generates PDF and emails it')
```

Without `callout=true`, the flow engine assumes the method is callout-free and may place it in a transaction context that prohibits callouts. The method will throw `System.CalloutException: You have uncommitted work pending` at runtime.

**Important**: If `callout=true` is set, the method cannot be called after any DML in the same transaction. This means: in the flow, do not place Create/Update/Delete Records elements before the Apex Action that has `callout=true`. The flow must call the Apex action first, then do any DML-based flow elements after.

### Error Handling Strategies

**Strategy 1: Return errors in the Result wrapper (recommended for most cases)**

The invocable method catches exceptions internally and returns success/failure information in the Result. The flow checks the result and branches accordingly. No transaction rollback occurs -- the flow stays on the happy path or takes an alternate path based on the result.

**Strategy 2: Let exceptions propagate to the flow's fault path**

The invocable method throws an exception (or does not catch one). The flow's Action element must have a **fault connector** wired to an error-handling path. The `{!$Flow.FaultMessage}` variable contains the exception message. The entire transaction rolls back unless the fault path explicitly handles it.

**Strategy 3: Throw a custom exception to intentionally roll back**

Useful when you want to guarantee all-or-nothing behavior. Throw an `InvocableActionException` or any custom exception. The fault path fires, and all DML in the transaction is rolled back.

For the Purchase Order pattern, **Strategy 1** is best: catch the exception, return an error message, and let the flow show a friendly error screen to the user rather than the generic "An unhandled fault has occurred" message.

### When to Use @InvocableMethod vs @AuraEnabled vs REST

| Criterion | @InvocableMethod | @AuraEnabled | REST API (@RestResource) |
|-----------|-----------------|-------------|------------------------|
| Called from | Flows, Process Builder, REST API, Agentforce | LWC, Aura components | External systems, integrations |
| Parameter shape | `List<Request>` (one param, bulkified) | Any number of params, any types | HTTP request body |
| Caching | No | Yes (`cacheable=true`) | No |
| Runs as | Running user (respects sharing) | Running user (respects sharing) | API user / integration user |
| Callout support | Yes (with `callout=true`) | Yes | Yes |
| Admin-configurable | Yes (visible in Flow Builder) | No (requires code changes) | No |
| Bulk-ready | Yes (receives List) | No (one call per interaction) | Depends on implementation |
| Best for | Automation, admin-driven processes | Interactive UI, real-time UX | External system integration |

**Rule of thumb**: If the caller is a flow or admin-built automation, use `@InvocableMethod`. If the caller is an LWC or Aura component, use `@AuraEnabled`. If the caller is an external system, use `@RestResource`. Never use `@AuraEnabled` just because you are more familiar with it -- flows cannot call `@AuraEnabled` methods.

## Code Examples

### Basic Invocable Method: The Skeleton

```apex
public with sharing class CreateTaskAction {

    @InvocableMethod(
        label='Create Follow-Up Task'
        description='Creates a Task linked to the specified record'
        category='Custom Actions'
    )
    public static List<Result> execute(List<Request> requests) {
        List<Result> results = new List<Result>();
        List<Task> tasksToInsert = new List<Task>();

        // build all tasks first -- no DML inside the loop
        for (Request req : requests) {
            Task t = new Task();
            t.Subject = req.subject;
            t.WhatId = req.recordId;
            t.OwnerId = req.ownerId;
            t.Status = 'Not Started';
            t.ActivityDate = Date.today().addDays(req.dueDays != null ? req.dueDays : 7);
            tasksToInsert.add(t);
        }

        // single DML for the entire batch
        List<Database.SaveResult> saveResults = Database.insert(tasksToInsert, false);

        // map results back 1:1 with requests
        for (Integer i = 0; i < saveResults.size(); i++) {
            Result res = new Result();
            if (saveResults[i].isSuccess()) {
                res.isSuccess = true;
                res.taskId = saveResults[i].getId();
                res.errorMessage = null;
            } else {
                res.isSuccess = false;
                res.taskId = null;
                res.errorMessage = saveResults[i].getErrors()[0].getMessage();
            }
            results.add(res);
        }

        return results;
    }

    // -- input wrapper: one instance per flow interview --
    public class Request {
        @InvocableVariable(label='Record ID' description='The record to associate the Task with' required=true)
        public Id recordId;

        @InvocableVariable(label='Task Subject' description='Subject line for the Task' required=true)
        public String subject;

        @InvocableVariable(label='Owner ID' description='User ID to assign the Task to' required=true)
        public Id ownerId;

        @InvocableVariable(label='Due In Days' description='Number of days until the Task is due (default 7)')
        public Integer dueDays;
    }

    // -- output wrapper: one instance per flow interview --
    public class Result {
        @InvocableVariable(label='Success' description='Whether the Task was created successfully')
        public Boolean isSuccess;

        @InvocableVariable(label='Task ID' description='The ID of the created Task (null if failed)')
        public Id taskId;

        @InvocableVariable(label='Error Message' description='Error details if the Task creation failed')
        public String errorMessage;
    }
}
```

### Complete Purchase Order PDF + Email Pattern

This is the full implementation for the Purchase Order workflow: generate a PDF from a Visualforce page, email it to a selected Contact, stamp the PO with a digital authorization, and log the action.

```apex
public with sharing class SendPurchaseOrderPdfAction {

    // -- only one @InvocableMethod per class --
    @InvocableMethod(
        callout=true
        label='Send Purchase Order PDF'
        description='Generates a PDF of the Purchase Order, emails it to the selected Contact, stamps digital authorization, and logs the action.'
        category='Purchase Orders'
    )
    public static List<Result> execute(List<Request> requests) {
        List<Result> results = new List<Result>();

        // collect all PO IDs and Contact IDs upfront for bulk query
        Set<Id> poIds = new Set<Id>();
        Set<Id> contactIds = new Set<Id>();
        for (Request req : requests) {
            poIds.add(req.purchaseOrderId);
            contactIds.add(req.contactId);
        }

        // bulk query: get all POs and Contacts in one shot each
        Map<Id, Purchase_Order__c> poMap = new Map<Id, Purchase_Order__c>([
            SELECT Id, Name, Authorizer_Name__c, Account__c, Account__r.Name,
                   Total_Amount__c, Status__c, Digital_Authorization__c
            FROM Purchase_Order__c
            WHERE Id IN :poIds
        ]);
        Map<Id, Contact> contactMap = new Map<Id, Contact>([
            SELECT Id, FirstName, LastName, Email
            FROM Contact
            WHERE Id IN :contactIds
        ]);

        // process each request
        // NOTE: getContentAsPDF() and sendEmail() have per-transaction limits.
        // For bulk scenarios (200+ records), consider using Queueable or Batch Apex instead.
        List<Purchase_Order__c> posToUpdate = new List<Purchase_Order__c>();
        List<Task> tasksToInsert = new List<Task>();
        List<Messaging.SingleEmailMessage> emailsToSend = new List<Messaging.SingleEmailMessage>();

        for (Request req : requests) {
            Result res = new Result();

            try {
                Purchase_Order__c po = poMap.get(req.purchaseOrderId);
                Contact recipient = contactMap.get(req.contactId);

                // validate inputs
                if (po == null) {
                    res.isSuccess = false;
                    res.errorMessage = 'Purchase Order not found: ' + req.purchaseOrderId;
                    results.add(res);
                    continue;
                }
                if (recipient == null || String.isBlank(recipient.Email)) {
                    res.isSuccess = false;
                    res.errorMessage = 'Contact not found or has no email address: ' + req.contactId;
                    results.add(res);
                    continue;
                }

                // generate the PDF from the Visualforce page
                // the VF page must accept an 'id' parameter and render the PO
                PageReference pdfPage = Page.Purchase_Order_PDF;
                pdfPage.getParameters().put('id', po.Id);
                Blob pdfBlob;
                if (Test.isRunningTest()) {
                    // getContentAsPDF() returns a blank blob in test context --
                    // it does not actually render the VF page
                    pdfBlob = Blob.valueOf('Test PDF Content');
                } else {
                    pdfBlob = pdfPage.getContentAsPDF();
                }
                System.debug('// generated PDF for PO: ' + po.Name + ', size: ' + pdfBlob.size() + ' bytes');

                // build the email attachment
                Messaging.EmailFileAttachment attachment = new Messaging.EmailFileAttachment();
                attachment.setFileName(po.Name + '.pdf');
                attachment.setBody(pdfBlob);
                attachment.setContentType('application/pdf');

                // build the email message
                Messaging.SingleEmailMessage email = new Messaging.SingleEmailMessage();
                email.setToAddresses(new List<String>{ recipient.Email });
                email.setSubject('Purchase Order: ' + po.Name);
                email.setPlainTextBody(
                    'Dear ' + recipient.FirstName + ',\n\n' +
                    'Please find attached the Purchase Order ' + po.Name + '.\n\n' +
                    'Thank you.'
                );
                email.setFileAttachments(new List<Messaging.EmailFileAttachment>{ attachment });

                // use org-wide email address if one is configured
                // look up by display name or hardcode the OrgWideEmailAddress Id
                List<OrgWideEmailAddress> owea = [
                    SELECT Id FROM OrgWideEmailAddress WHERE DisplayName = 'Purchase Orders' LIMIT 1
                ];
                if (!owea.isEmpty()) {
                    email.setOrgWideEmailAddressId(owea[0].Id);
                }

                emailsToSend.add(email);

                // stamp the PO with digital authorization
                Purchase_Order__c poUpdate = new Purchase_Order__c();
                poUpdate.Id = po.Id;
                poUpdate.Digital_Authorization__c = 'Digital Authorization from ' + po.Authorizer_Name__c;
                posToUpdate.add(poUpdate);

                // log the action as a completed Task
                Task logTask = new Task();
                logTask.Subject = 'PO PDF Sent: ' + po.Name;
                logTask.WhatId = po.Id;
                logTask.WhoId = recipient.Id;
                logTask.Status = 'Completed';
                logTask.Description = 'PDF of ' + po.Name + ' emailed to ' +
                    recipient.FirstName + ' ' + recipient.LastName +
                    ' (' + recipient.Email + ')';
                logTask.ActivityDate = Date.today();
                tasksToInsert.add(logTask);

                res.isSuccess = true;
                res.errorMessage = null;

            } catch (Exception ex) {
                // catch per-record errors without killing the entire batch
                res.isSuccess = false;
                res.errorMessage = ex.getMessage();
                System.debug('// error processing PO ' + req.purchaseOrderId + ': ' + ex.getMessage());
            }

            results.add(res);
        }

        // bulk DML and email send -- outside the loop
        // IMPORTANT: callout=true means getContentAsPDF() above counts as a callout.
        // DML must happen AFTER all callouts are complete (no callouts after DML in the same transaction).
        if (!posToUpdate.isEmpty()) {
            update posToUpdate;
            System.debug('// updated ' + posToUpdate.size() + ' Purchase Orders with digital authorization');
        }
        if (!tasksToInsert.isEmpty()) {
            insert tasksToInsert;
            System.debug('// created ' + tasksToInsert.size() + ' activity log Tasks');
        }
        if (!emailsToSend.isEmpty()) {
            List<Messaging.SendEmailResult> emailResults = Messaging.sendEmail(emailsToSend);
            System.debug('// sent ' + emailsToSend.size() + ' emails');

            // check for email send failures
            for (Integer i = 0; i < emailResults.size(); i++) {
                if (!emailResults[i].isSuccess()) {
                    // find the corresponding result and mark it failed
                    // note: emailsToSend index may not match results index if some were skipped
                    System.debug('// email send failed: ' + emailResults[i].getErrors()[0].getMessage());
                }
            }
        }

        return results;
    }

    // -- input wrapper --
    public class Request {
        @InvocableVariable(
            label='Purchase Order ID'
            description='ID of the Purchase Order record to generate a PDF for'
            required=true
        )
        public Id purchaseOrderId;

        @InvocableVariable(
            label='Contact ID'
            description='ID of the Contact to email the PDF to'
            required=true
        )
        public Id contactId;
    }

    // -- output wrapper --
    public class Result {
        @InvocableVariable(label='Success' description='Whether the PDF was generated and emailed successfully')
        public Boolean isSuccess;

        @InvocableVariable(label='Error Message' description='Error details if the action failed')
        public String errorMessage;
    }
}
```

### Test Class for the Purchase Order Invocable Method

```apex
@IsTest
private class SendPurchaseOrderPdfActionTest {

    @TestSetup
    static void setupTestData() {
        // create an Account
        Account acc = new Account(Name = 'Test Corp');
        insert acc;

        // create a Contact with an email
        Contact con = new Contact(
            FirstName = 'Jane',
            LastName = 'Doe',
            Email = 'jane.doe@example.com',
            AccountId = acc.Id
        );
        insert con;

        // create a Purchase Order
        Purchase_Order__c po = new Purchase_Order__c(
            Name = 'PO-0001',
            Account__c = acc.Id,
            Authorizer_Name__c = 'John Smith',
            Status__c = 'Approved',
            Total_Amount__c = 15000.00
        );
        insert po;
    }

    @IsTest
    static void testSendPdf_singleRecord_success() {
        Purchase_Order__c po = [SELECT Id FROM Purchase_Order__c LIMIT 1];
        Contact con = [SELECT Id FROM Contact LIMIT 1];

        // build the request list -- even for one record, it must be a list
        SendPurchaseOrderPdfAction.Request req = new SendPurchaseOrderPdfAction.Request();
        req.purchaseOrderId = po.Id;
        req.contactId = con.Id;

        List<SendPurchaseOrderPdfAction.Request> requests = new List<SendPurchaseOrderPdfAction.Request>();
        requests.add(req);

        Test.startTest();
        List<SendPurchaseOrderPdfAction.Result> results = SendPurchaseOrderPdfAction.execute(requests);
        Test.stopTest();

        // assert 1:1 result mapping
        System.assertEquals(1, results.size(), 'Should return exactly one result for one request');
        System.assertEquals(true, results[0].isSuccess, 'Should succeed for valid PO and Contact');
        System.assertEquals(null, results[0].errorMessage, 'Error message should be null on success');

        // verify the PO was stamped with digital authorization
        Purchase_Order__c updatedPo = [
            SELECT Digital_Authorization__c FROM Purchase_Order__c WHERE Id = :po.Id
        ];
        System.assertEquals(
            'Digital Authorization from John Smith',
            updatedPo.Digital_Authorization__c,
            'PO should be stamped with authorizer name'
        );

        // verify the Task was created
        List<Task> tasks = [SELECT Subject, WhatId, WhoId, Status FROM Task WHERE WhatId = :po.Id];
        System.assertEquals(1, tasks.size(), 'Should create one log Task');
        System.assertEquals('Completed', tasks[0].Status, 'Task should be completed');
        System.assertEquals(con.Id, tasks[0].WhoId, 'Task should be linked to the Contact');
    }

    @IsTest
    static void testSendPdf_bulkScenario() {
        // create 5 POs and 5 Contacts for a bulk test
        Account acc = [SELECT Id FROM Account LIMIT 1];
        List<Contact> contacts = new List<Contact>();
        List<Purchase_Order__c> pos = new List<Purchase_Order__c>();

        for (Integer i = 0; i < 5; i++) {
            contacts.add(new Contact(
                FirstName = 'Bulk',
                LastName = 'Contact' + i,
                Email = 'bulk' + i + '@example.com',
                AccountId = acc.Id
            ));
            pos.add(new Purchase_Order__c(
                Name = 'PO-BULK-' + i,
                Account__c = acc.Id,
                Authorizer_Name__c = 'Bulk Authorizer ' + i,
                Status__c = 'Approved',
                Total_Amount__c = 1000.00 * i
            ));
        }
        insert contacts;
        insert pos;

        // build request list
        List<SendPurchaseOrderPdfAction.Request> requests = new List<SendPurchaseOrderPdfAction.Request>();
        for (Integer i = 0; i < 5; i++) {
            SendPurchaseOrderPdfAction.Request req = new SendPurchaseOrderPdfAction.Request();
            req.purchaseOrderId = pos[i].Id;
            req.contactId = contacts[i].Id;
            requests.add(req);
        }

        Test.startTest();
        List<SendPurchaseOrderPdfAction.Result> results = SendPurchaseOrderPdfAction.execute(requests);
        Test.stopTest();

        // verify 1:1 result count
        System.assertEquals(5, results.size(), 'Should return 5 results for 5 requests');

        // verify all succeeded
        for (SendPurchaseOrderPdfAction.Result res : results) {
            System.assertEquals(true, res.isSuccess, 'Each bulk result should succeed');
        }

        // verify Tasks created in bulk
        List<Task> tasks = [SELECT Id FROM Task WHERE WhatId IN :pos];
        System.assertEquals(5, tasks.size(), 'Should create 5 log Tasks');
    }

    @IsTest
    static void testSendPdf_invalidPurchaseOrder() {
        Contact con = [SELECT Id FROM Contact LIMIT 1];

        // pass a bogus PO ID
        SendPurchaseOrderPdfAction.Request req = new SendPurchaseOrderPdfAction.Request();
        req.purchaseOrderId = 'a000000000000AAAA'; // nonexistent
        req.contactId = con.Id;

        Test.startTest();
        List<SendPurchaseOrderPdfAction.Result> results = SendPurchaseOrderPdfAction.execute(
            new List<SendPurchaseOrderPdfAction.Request>{ req }
        );
        Test.stopTest();

        System.assertEquals(false, results[0].isSuccess, 'Should fail for invalid PO');
        System.assert(results[0].errorMessage.contains('not found'), 'Error should mention PO not found');
    }

    @IsTest
    static void testSendPdf_contactWithNoEmail() {
        Purchase_Order__c po = [SELECT Id FROM Purchase_Order__c LIMIT 1];

        // create a Contact with no email
        Contact noEmailContact = new Contact(FirstName = 'No', LastName = 'Email');
        insert noEmailContact;

        SendPurchaseOrderPdfAction.Request req = new SendPurchaseOrderPdfAction.Request();
        req.purchaseOrderId = po.Id;
        req.contactId = noEmailContact.Id;

        Test.startTest();
        List<SendPurchaseOrderPdfAction.Result> results = SendPurchaseOrderPdfAction.execute(
            new List<SendPurchaseOrderPdfAction.Request>{ req }
        );
        Test.stopTest();

        System.assertEquals(false, results[0].isSuccess, 'Should fail for Contact with no email');
        System.assert(results[0].errorMessage.contains('no email'), 'Error should mention missing email');
    }
}
```

### Invocable Method with Callout (External API Pattern)

```apex
public with sharing class VerifyAddressAction {

    @InvocableMethod(
        callout=true
        label='Verify Mailing Address'
        description='Calls an external address verification API to validate and standardize the address'
        category='Address Verification'
    )
    public static List<Result> execute(List<Request> requests) {
        List<Result> results = new List<Result>();

        for (Request req : requests) {
            Result res = new Result();
            try {
                // build the HTTP request
                HttpRequest httpReq = new HttpRequest();
                httpReq.setEndpoint('callout:Address_Verification_API/verify');
                httpReq.setMethod('POST');
                httpReq.setHeader('Content-Type', 'application/json');
                httpReq.setBody(JSON.serialize(new Map<String, String>{
                    'street' => req.street,
                    'city' => req.city,
                    'state' => req.state,
                    'zip' => req.postalCode
                }));
                httpReq.setTimeout(10000); // 10 second timeout

                Http http = new Http();
                HttpResponse httpRes = http.send(httpReq);

                if (httpRes.getStatusCode() == 200) {
                    Map<String, Object> responseBody = (Map<String, Object>) JSON.deserializeUntyped(httpRes.getBody());
                    res.isValid = (Boolean) responseBody.get('valid');
                    res.standardizedAddress = (String) responseBody.get('standardized_address');
                    res.errorMessage = null;
                } else {
                    res.isValid = false;
                    res.standardizedAddress = null;
                    res.errorMessage = 'API returned status ' + httpRes.getStatusCode();
                }
            } catch (Exception ex) {
                res.isValid = false;
                res.errorMessage = ex.getMessage();
            }
            results.add(res);
        }

        return results;
    }

    public class Request {
        @InvocableVariable(label='Street' required=true)
        public String street;

        @InvocableVariable(label='City' required=true)
        public String city;

        @InvocableVariable(label='State' required=true)
        public String state;

        @InvocableVariable(label='Postal Code' required=true)
        public String postalCode;
    }

    public class Result {
        @InvocableVariable(label='Is Valid')
        public Boolean isValid;

        @InvocableVariable(label='Standardized Address')
        public String standardizedAddress;

        @InvocableVariable(label='Error Message')
        public String errorMessage;
    }
}
```

### Simple Invocable Method: Single Value Return (No Inner Class)

For simple use cases where you pass one value and get one value back, you can skip the inner class pattern and use primitive types:

```apex
public with sharing class GetAccountNameAction {

    @InvocableMethod(
        label='Get Account Name'
        description='Returns the Account Name for a given Account ID'
        category='Utilities'
    )
    public static List<String> execute(List<Id> accountIds) {
        // still receives a list -- still must return a list of the same size
        List<String> names = new List<String>();
        Map<Id, Account> accountMap = new Map<Id, Account>([
            SELECT Id, Name FROM Account WHERE Id IN :accountIds
        ]);

        for (Id accId : accountIds) {
            Account acc = accountMap.get(accId);
            names.add(acc != null ? acc.Name : 'Unknown');
        }

        return names;
    }
}
```

**Note**: This works for simple cases, but the inner class pattern is preferred because it is extensible. Adding a second input later requires refactoring the primitive approach into inner classes -- which is a breaking change for any flows already using the method.

## Common Mistakes

1. **Putting SOQL or DML inside a loop in the invocable method** -- When Flow batches 200 interviews into one call, your method receives 200 requests. A SOQL query per iteration hits the 100-query limit at record 101; DML per iteration hits the 150-statement limit at record 151. Fix: Collect all IDs from the request list, query once with `WHERE Id IN :idSet`, process in a Map, DML once outside the loop.

2. **Returning fewer Result items than Request items received** -- The flow maps results back to interviews by list index. If you receive 5 requests but return 3 results, the 4th and 5th interviews get a runtime error. Fix: Always return exactly one Result per Request, even for error cases -- populate the Result with `isSuccess = false` and an error message.

3. **Forgetting `callout=true` when using `getContentAsPDF()`** -- `PageReference.getContentAsPDF()` behaves like a callout internally. Without the `callout=true` flag on the annotation, the flow engine may place the method in a transaction that disallows callouts, causing a `CalloutException`. Fix: Always include `callout=true` if the method generates PDFs or makes HTTP requests.

4. **Performing DML before a callout in the same invocable method** -- Salesforce prohibits callouts after uncommitted DML in the same transaction. If you `insert` a record, then call `getContentAsPDF()`, you get `System.CalloutException: You have uncommitted work pending`. Fix: Do all callouts first (PDF generation, HTTP requests), then do all DML (updates, inserts) after.

5. **Declaring more than one `@InvocableMethod` in the same class** -- This causes a compile error: `A class can only have one method annotated with @InvocableMethod`. Fix: Create a separate class for each invocable action. Name classes descriptively: `SendPurchaseOrderPdfAction`, `CreateFollowUpTaskAction`, etc.

6. **Using `@InvocableMethod` on a non-static method or a method with multiple parameters** -- The method must be `public static` and accept at most one parameter (which must be a `List<>`). Fix: Make the method static, consolidate multiple inputs into an inner Request class.

7. **Not handling the test context for `getContentAsPDF()`** -- In a test class, `getContentAsPDF()` does not actually render the Visualforce page. It returns a blob, but you cannot assert on its content. If your code assumes the blob is a valid PDF (e.g., checks the byte size), it may fail in tests. Fix: Use `Test.isRunningTest()` to provide a mock blob in test context, or simply do not assert on PDF content.

8. **Exposing invocable variables without labels or descriptions** -- Admins in Flow Builder see raw API names like `purchaseOrderId` instead of friendly labels like "Purchase Order ID". This makes the action unusable for non-developers. Fix: Always set `label` and `description` on every `@InvocableVariable`, and set `label`, `description`, and `category` on the `@InvocableMethod`.

9. **Catching all exceptions silently without returning error information** -- The method catches an exception, logs it with `System.debug()`, but returns a Result with `isSuccess = true` or leaves `errorMessage` null. The flow proceeds as if nothing went wrong. Fix: Always populate the Result with `isSuccess = false` and the exception message when an error occurs.

10. **Not testing the bulk scenario** -- The method works for one record but fails at 200 because of governor limits in loops, or because the result list size does not match. Fix: Always write a test that passes at least 5-10 requests in a single call (200 for trigger-driven scenarios). Verify the result list size matches the request list size.

## See Also

- [Quick Actions & Screen Flow Integration](../quick-actions/quick-actions-and-screen-flows.md)
- [Visualforce PDF Generation](../visualforce-pdf/) (if available)
- [Email Services](../email-services/) (if available)
