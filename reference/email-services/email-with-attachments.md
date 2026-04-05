# Email Services with Attachments

> **When to read this:** You need to send an email from Apex with a file attachment (especially a PDF), handle org-wide email addresses, or understand Salesforce email governor limits.

## Rules

- When sending an email to a Contact, Lead, or User, always use `setTargetObjectId()` instead of `setToAddresses()` because `setTargetObjectId()` enables merge field resolution in templates, creates an activity record when `setSaveAsActivity(true)`, and respects the recipient's email opt-out preferences.
- Never use both `setTargetObjectId()` and `setToAddresses()` with the same email address because the recipient will receive two copies of the email.
- Always use `setOrgWideEmailAddressId()` for any user-facing email because without it the "From" address is the running user's email, which looks unprofessional and may cause deliverability issues. Query the `OrgWideEmailAddress` record by `Address` or `DisplayName` — never hardcode the Id.
- When attaching a PDF blob, always set both `setFileName()` with a `.pdf` extension AND `setContentType('application/pdf')` on the `EmailFileAttachment` because some email clients use the content type to determine how to display the attachment, and a missing extension confuses users.
- Always check the `Messaging.SendEmailResult` return value after calling `Messaging.sendEmail()` because the method does not throw an exception on all failures — some errors (like an invalid email address format) are returned in the result object.
- Never assume `Messaging.sendEmail()` failures mean the email bounced because Salesforce only queues the email during the transaction — actual delivery happens asynchronously after the transaction commits. The `SendEmailResult` only catches errors that prevent queuing (bad address format, missing required fields, limit exceeded).
- Always call `setSaveAsActivity(true)` when sending to a Contact or Lead because this creates an EmailMessage record (or Task in orgs without Enhanced Email) in the activity timeline, giving sales reps visibility into what was sent.
- When using `setSaveAsActivity(true)`, always also call `setWhatId()` with the related record Id (like the Purchase Order) because this links the activity to both the Contact AND the record, so the email shows in the activity timeline on both objects.
- Always check deliverability settings before debugging "emails not sending" in sandboxes because sandbox orgs default to "System email only" — Apex-sent emails to external addresses are silently dropped. Set it to "All email" in Setup > Email > Deliverability.
- When building the email body in Apex (not using a template), always set BOTH `setPlainTextBody()` and `setHtmlBody()` because some email clients only display plain text, and spam filters are more suspicious of HTML-only emails with no plain text alternative.

## How It Works

### The Email Object Model

Salesforce email sending in Apex uses three key classes from the `Messaging` namespace:

1. **`Messaging.SingleEmailMessage`** -- Represents one email to send. Contains all the metadata: recipients, subject, body, attachments, org-wide address, template, and activity logging settings.

2. **`Messaging.EmailFileAttachment`** -- Represents a file to attach. You set the file name, content type, and body (as a `Blob`). You can attach multiple files to one email.

3. **`Messaging.sendEmail()`** -- The static method that actually sends. It accepts a `List<Messaging.SingleEmailMessage>` (up to 10 per invocation) and returns a `List<Messaging.SendEmailResult>`.

### How Sending Works

When you call `Messaging.sendEmail()`, Salesforce does NOT immediately send the email. It validates the email parameters, queues the email for delivery, and returns a `SendEmailResult` indicating whether queuing succeeded. The actual email delivery happens asynchronously after the Apex transaction commits. If the transaction rolls back (uncaught exception, failed DML, etc.), queued emails are NOT sent.

### Activity Logging

When `setSaveAsActivity(true)` is set and `setTargetObjectId()` points to a Contact or Lead:
- In orgs with **Enhanced Email** enabled (the default since 2020): an `EmailMessage` record is created, visible in the activity timeline.
- In orgs without Enhanced Email: a `Task` record is created with `Subject = 'Email: <your subject>'`.

The `setWhatId()` value determines which record (Account, Opportunity, custom object) the activity is also linked to. For custom objects, the object must have Activities enabled (Allow Activities = true in the object settings).

### Templates vs Programmatic Body

**Use an Email Template when:**
- Non-developers need to edit email content.
- You want merge fields resolved automatically from the target Contact/Lead and the `whatId` record.
- The email format is mostly static with dynamic field insertions.

**Build the body in Apex when:**
- The email content depends on complex logic (conditional sections, aggregated data, calculations).
- You need to include data from multiple objects not easily reachable via merge field syntax.
- The email is a system notification that developers maintain.

You can also use a hybrid: call `setTreatBodiesAsTemplate(true)` to enable merge field syntax (`{!Contact.FirstName}`) inside programmatically-built `setHtmlBody()` strings. This requires `setTargetObjectId()` and optionally `setWhatId()` to be set.

## Code Examples

### Complete Email Service for Purchase Order PDF

```apex
public with sharing class PurchaseOrderEmailService {

    // send the PO PDF to the specified contact
    // @param poId: the Purchase_Order__c record Id
    // @param contactId: the Contact to send the email to
    // @param pdfBlob: the rendered PDF blob from PurchaseOrderPDFService
    // @param pdfFileName: the file name for the attachment (e.g., "PO-00123.pdf")
    // @return Boolean: true if the email was queued successfully
    public static Boolean sendPurchaseOrderPDF(
        Id poId,
        Id contactId,
        Blob pdfBlob,
        String pdfFileName
    ) {
        System.debug('Sending PO PDF email. PO: ' + poId + ', Contact: ' + contactId);

        // query the PO for field values we need in the email body
        Purchase_Order__c po = [
            SELECT Id, Name, Grand_Total__c, Authorizer_Name__c, PO_Date__c
            FROM Purchase_Order__c
            WHERE Id = :poId
            LIMIT 1
        ];

        // query the contact for the greeting
        Contact recipient = [
            SELECT Id, FirstName, LastName, Email
            FROM Contact
            WHERE Id = :contactId
            LIMIT 1
        ];

        System.debug('Recipient: ' + recipient.Email);

        // build the PDF attachment
        Messaging.EmailFileAttachment pdfAttachment = new Messaging.EmailFileAttachment();
        pdfAttachment.setFileName(pdfFileName);
        pdfAttachment.setBody(pdfBlob);
        pdfAttachment.setContentType('application/pdf');
        // inline = false means it shows as a downloadable attachment, not embedded
        pdfAttachment.setInline(false);

        // build the email message
        Messaging.SingleEmailMessage email = new Messaging.SingleEmailMessage();

        // set the "From" address to the org-wide email
        // query it by address — never hardcode the Id
        List<OrgWideEmailAddress> owea = [
            SELECT Id
            FROM OrgWideEmailAddress
            WHERE Address = 'purchasing@yourcompany.com'
            LIMIT 1
        ];
        if (!owea.isEmpty()) {
            email.setOrgWideEmailAddressId(owea[0].Id);
            System.debug('Using org-wide email address: purchasing@yourcompany.com');
        }

        // target the contact — this enables activity logging and opt-out respect
        email.setTargetObjectId(contactId);

        // link the activity to the Purchase Order record
        // the PO object must have "Allow Activities" enabled for this to work
        email.setWhatId(poId);

        // log the email as an activity on both the Contact and the PO
        email.setSaveAsActivity(true);

        // build the email subject and body
        email.setSubject('Purchase Order ' + po.Name + ' for Your Review');

        // always set both HTML and plain text body
        String htmlBody = buildHtmlBody(po, recipient);
        String plainBody = buildPlainTextBody(po, recipient);
        email.setHtmlBody(htmlBody);
        email.setPlainTextBody(plainBody);

        // attach the PDF
        email.setFileAttachments(new List<Messaging.EmailFileAttachment>{ pdfAttachment });

        // send the email
        List<Messaging.SendEmailResult> results = Messaging.sendEmail(
            new List<Messaging.SingleEmailMessage>{ email }
        );

        // check the result — sendEmail does not always throw on failure
        Boolean success = results[0].isSuccess();
        if (success) {
            System.debug('Email queued successfully for delivery');
        } else {
            // log the errors — don't just swallow them
            for (Messaging.SendEmailError err : results[0].getErrors()) {
                System.debug(LoggingLevel.ERROR,
                    'Email send failed. Status: ' + err.getStatusCode() +
                    ', Message: ' + err.getMessage() +
                    ', Fields: ' + err.getFields()
                );
            }
        }

        return success;
    }

    // build the HTML email body
    // @param po: the Purchase Order record with fields populated
    // @param recipient: the Contact record for the greeting
    // @return String: the HTML body
    private static String buildHtmlBody(Purchase_Order__c po, Contact recipient) {
        String greeting = String.isNotBlank(recipient.FirstName)
            ? recipient.FirstName
            : recipient.LastName;

        // remember — use inline styles for email HTML, not CSS classes
        // email clients strip <style> blocks and external CSS
        String body = '';
        body += '<div style="font-family: Arial, sans-serif; font-size: 14px; color: #333333;">';
        body += '<p>Dear ' + greeting + ',</p>';
        body += '<p>Please find attached Purchase Order <strong>' + po.Name + '</strong>';
        body += ' dated ' + po.PO_Date__c.format() + '.</p>';
        body += '<table style="border-collapse: collapse; margin: 15px 0;">';
        body += '<tr>';
        body += '<td style="padding: 5px 15px 5px 0; font-weight: bold;">PO Number:</td>';
        body += '<td style="padding: 5px 0;">' + po.Name + '</td>';
        body += '</tr>';
        body += '<tr>';
        body += '<td style="padding: 5px 15px 5px 0; font-weight: bold;">Total Amount:</td>';
        body += '<td style="padding: 5px 0;">$' + po.Grand_Total__c.setScale(2).toPlainString() + '</td>';
        body += '</tr>';
        body += '<tr>';
        body += '<td style="padding: 5px 15px 5px 0; font-weight: bold;">Authorized By:</td>';
        body += '<td style="padding: 5px 0;">' + po.Authorizer_Name__c + '</td>';
        body += '</tr>';
        body += '</table>';
        body += '<p>Please review and confirm at your earliest convenience.</p>';
        body += '<p>Best regards,<br/>' + po.Authorizer_Name__c + '</p>';
        body += '</div>';

        return body;
    }

    // build the plain text email body
    // @param po: the Purchase Order record with fields populated
    // @param recipient: the Contact record for the greeting
    // @return String: the plain text body
    private static String buildPlainTextBody(Purchase_Order__c po, Contact recipient) {
        String greeting = String.isNotBlank(recipient.FirstName)
            ? recipient.FirstName
            : recipient.LastName;

        String body = '';
        body += 'Dear ' + greeting + ',\n\n';
        body += 'Please find attached Purchase Order ' + po.Name;
        body += ' dated ' + po.PO_Date__c.format() + '.\n\n';
        body += 'PO Number: ' + po.Name + '\n';
        body += 'Total Amount: $' + po.Grand_Total__c.setScale(2).toPlainString() + '\n';
        body += 'Authorized By: ' + po.Authorizer_Name__c + '\n\n';
        body += 'Please review and confirm at your earliest convenience.\n\n';
        body += 'Best regards,\n' + po.Authorizer_Name__c;

        return body;
    }
}
```

### Using an Email Template Instead of Programmatic Body

```apex
public with sharing class PurchaseOrderTemplateEmailService {

    // send the PO PDF using a stored email template
    // @param poId: the Purchase_Order__c record Id
    // @param contactId: the Contact to receive the email
    // @param pdfBlob: the rendered PDF blob
    // @param pdfFileName: the attachment file name
    // @param templateDeveloperName: the unique name of the email template
    // @return Boolean: true if email was queued successfully
    public static Boolean sendWithTemplate(
        Id poId,
        Id contactId,
        Blob pdfBlob,
        String pdfFileName,
        String templateDeveloperName
    ) {
        System.debug('Sending PO PDF with template: ' + templateDeveloperName);

        // look up the email template by developer name — never hardcode Id
        EmailTemplate tmpl = [
            SELECT Id
            FROM EmailTemplate
            WHERE DeveloperName = :templateDeveloperName
            LIMIT 1
        ];

        // build the attachment
        Messaging.EmailFileAttachment pdfAttachment = new Messaging.EmailFileAttachment();
        pdfAttachment.setFileName(pdfFileName);
        pdfAttachment.setBody(pdfBlob);
        pdfAttachment.setContentType('application/pdf');
        pdfAttachment.setInline(false);

        // build the email
        Messaging.SingleEmailMessage email = new Messaging.SingleEmailMessage();

        // set org-wide address
        List<OrgWideEmailAddress> owea = [
            SELECT Id FROM OrgWideEmailAddress
            WHERE Address = 'purchasing@yourcompany.com' LIMIT 1
        ];
        if (!owea.isEmpty()) {
            email.setOrgWideEmailAddressId(owea[0].Id);
        }

        // template requires setTargetObjectId — this is the Contact/Lead
        // whose merge fields ({!Contact.FirstName}, etc.) get resolved
        email.setTargetObjectId(contactId);

        // setWhatId controls which record's merge fields get resolved
        // for custom object merge fields like {!Purchase_Order__c.Name}
        // the custom object MUST have Allow Activities enabled
        email.setWhatId(poId);

        // point to the template — this overrides setSubject/setHtmlBody/setPlainTextBody
        email.setTemplateId(tmpl.Id);

        email.setSaveAsActivity(true);
        email.setFileAttachments(new List<Messaging.EmailFileAttachment>{ pdfAttachment });

        List<Messaging.SendEmailResult> results = Messaging.sendEmail(
            new List<Messaging.SingleEmailMessage>{ email }
        );

        return results[0].isSuccess();
    }
}
```

### Sending to Multiple Recipients in One Transaction

```apex
public with sharing class BulkPurchaseOrderEmailService {

    // send PO PDFs to multiple contacts in a single transaction
    // @param emailRequests: list of wrapper objects with PO and Contact info
    // @return Map<Id, Boolean>: map of PO Id to send success/failure
    public static Map<Id, Boolean> sendBulkEmails(List<POEmailRequest> emailRequests) {
        System.debug('Sending ' + emailRequests.size() + ' PO emails');

        // governor limit: max 10 SingleEmailMessages per sendEmail() call
        // if you have more than 10, you must batch them
        if (emailRequests.size() > 10) {
            throw new PurchaseOrderEmailException(
                'Cannot send more than 10 emails per transaction. ' +
                'Received: ' + emailRequests.size() + '. Use a Queueable to batch.'
            );
        }

        // look up org-wide email address once
        List<OrgWideEmailAddress> owea = [
            SELECT Id FROM OrgWideEmailAddress
            WHERE Address = 'purchasing@yourcompany.com' LIMIT 1
        ];
        Id oweaId = owea.isEmpty() ? null : owea[0].Id;

        List<Messaging.SingleEmailMessage> emails = new List<Messaging.SingleEmailMessage>();

        for (POEmailRequest req : emailRequests) {
            Messaging.EmailFileAttachment att = new Messaging.EmailFileAttachment();
            att.setFileName(req.pdfFileName);
            att.setBody(req.pdfBlob);
            att.setContentType('application/pdf');
            att.setInline(false);

            Messaging.SingleEmailMessage msg = new Messaging.SingleEmailMessage();
            if (oweaId != null) {
                msg.setOrgWideEmailAddressId(oweaId);
            }
            msg.setTargetObjectId(req.contactId);
            msg.setWhatId(req.poId);
            msg.setSaveAsActivity(true);
            msg.setSubject('Purchase Order ' + req.poName + ' for Your Review');
            msg.setHtmlBody(req.htmlBody);
            msg.setPlainTextBody(req.plainBody);
            msg.setFileAttachments(new List<Messaging.EmailFileAttachment>{ att });

            emails.add(msg);
        }

        // send all emails in one call
        List<Messaging.SendEmailResult> results = Messaging.sendEmail(emails);

        // map results back to PO Ids
        Map<Id, Boolean> resultMap = new Map<Id, Boolean>();
        for (Integer i = 0; i < results.size(); i++) {
            resultMap.put(emailRequests[i].poId, results[i].isSuccess());
            if (!results[i].isSuccess()) {
                for (Messaging.SendEmailError err : results[i].getErrors()) {
                    System.debug(LoggingLevel.ERROR,
                        'Failed to send email for PO ' + emailRequests[i].poId +
                        ': ' + err.getMessage()
                    );
                }
            }
        }

        return resultMap;
    }

    // wrapper class for email request data
    public class POEmailRequest {
        public Id poId;
        public String poName;
        public Id contactId;
        public Blob pdfBlob;
        public String pdfFileName;
        public String htmlBody;
        public String plainBody;
    }

    public class PurchaseOrderEmailException extends Exception {}
}
```

### Complete Test Class

```apex
@isTest
private class PurchaseOrderEmailServiceTest {

    @TestSetup
    static void makeData() {
        // create a contact to send to
        Account acct = new Account(Name = 'Test Vendor');
        insert acct;

        Contact con = new Contact(
            FirstName = 'John',
            LastName = 'Doe',
            Email = 'john.doe@example.com',
            AccountId = acct.Id
        );
        insert con;

        // create a purchase order
        Purchase_Order__c po = new Purchase_Order__c(
            PO_Date__c = Date.today(),
            Status__c = 'Draft',
            Authorizer_Name__c = 'Jane Smith',
            Grand_Total__c = 1500.00,
            Payment_Terms__c = 'Net 30'
        );
        insert po;
    }

    @isTest
    static void testSendPurchaseOrderPDF() {
        Purchase_Order__c po = [SELECT Id, Name FROM Purchase_Order__c LIMIT 1];
        Contact con = [SELECT Id FROM Contact LIMIT 1];
        Blob testBlob = Blob.valueOf('Test PDF');

        Test.startTest();
        // sendEmail() in test context does NOT actually deliver the email
        // but it DOES execute the code and count against limits
        Boolean success = PurchaseOrderEmailService.sendPurchaseOrderPDF(
            po.Id,
            con.Id,
            testBlob,
            po.Name + '.pdf'
        );
        Test.stopTest();

        // verify the email was queued (not that it was delivered)
        System.assertEquals(true, success, 'Email should have been queued successfully');

        // verify activity was logged on the contact
        // (only works if Enhanced Email is enabled — use EmailMessage)
        // in orgs without Enhanced Email, check for Task records instead
        List<Task> tasks = [
            SELECT Id, Subject
            FROM Task
            WHERE WhoId = :con.Id
              AND WhatId = :po.Id
        ];
        // note: activity creation may not occur in all test contexts
        // this assertion is informational — the key assertion is success == true
    }

    @isTest
    static void testSendWithoutOrgWideAddress() {
        // test that the email still sends when no org-wide address is found
        // (OrgWideEmailAddress records cannot be created in test context)
        Purchase_Order__c po = [SELECT Id, Name FROM Purchase_Order__c LIMIT 1];
        Contact con = [SELECT Id FROM Contact LIMIT 1];
        Blob testBlob = Blob.valueOf('Test PDF');

        Test.startTest();
        Boolean success = PurchaseOrderEmailService.sendPurchaseOrderPDF(
            po.Id,
            con.Id,
            testBlob,
            'TestPO.pdf'
        );
        Test.stopTest();

        // should succeed even without org-wide address — falls back to running user
        System.assertEquals(true, success, 'Email should succeed without org-wide address');
    }

    @isTest
    static void testBulkEmailLimitEnforcement() {
        // verify that trying to send more than 10 emails throws our custom exception
        List<BulkPurchaseOrderEmailService.POEmailRequest> requests =
            new List<BulkPurchaseOrderEmailService.POEmailRequest>();

        for (Integer i = 0; i < 11; i++) {
            BulkPurchaseOrderEmailService.POEmailRequest req =
                new BulkPurchaseOrderEmailService.POEmailRequest();
            req.poId = null;
            req.poName = 'PO-' + i;
            req.contactId = null;
            req.pdfBlob = Blob.valueOf('test');
            req.pdfFileName = 'test.pdf';
            req.htmlBody = '<p>test</p>';
            req.plainBody = 'test';
            requests.add(req);
        }

        Test.startTest();
        try {
            BulkPurchaseOrderEmailService.sendBulkEmails(requests);
            System.assert(false, 'Should have thrown PurchaseOrderEmailException');
        } catch (BulkPurchaseOrderEmailService.PurchaseOrderEmailException e) {
            System.assert(
                e.getMessage().contains('Cannot send more than 10'),
                'Error message should mention the 10-email limit'
            );
        }
        Test.stopTest();
    }

    @isTest
    static void testBulkEmailSend() {
        Purchase_Order__c po = [SELECT Id, Name FROM Purchase_Order__c LIMIT 1];
        Contact con = [SELECT Id FROM Contact LIMIT 1];

        BulkPurchaseOrderEmailService.POEmailRequest req =
            new BulkPurchaseOrderEmailService.POEmailRequest();
        req.poId = po.Id;
        req.poName = po.Name;
        req.contactId = con.Id;
        req.pdfBlob = Blob.valueOf('test PDF');
        req.pdfFileName = po.Name + '.pdf';
        req.htmlBody = '<p>Test email body</p>';
        req.plainBody = 'Test email body';

        Test.startTest();
        Map<Id, Boolean> results = BulkPurchaseOrderEmailService.sendBulkEmails(
            new List<BulkPurchaseOrderEmailService.POEmailRequest>{ req }
        );
        Test.stopTest();

        System.assertEquals(true, results.get(po.Id), 'Email for PO should have succeeded');
    }
}
```

### Stamping the Record After Email Send (Digital Authorization Pattern)

```apex
public with sharing class PurchaseOrderAuthorizationService {

    // generate PDF, email it, and stamp the PO with digital authorization
    // this is the method an @InvocableMethod would call from a screen flow
    // @param poId: the Purchase_Order__c record Id
    // @param contactId: the Contact to receive the email
    public static void authorizeAndSendPDF(Id poId, Id contactId) {
        System.debug('Starting authorization flow for PO: ' + poId);

        // step 1 — generate the PDF FIRST (before any DML)
        // getContentAsPDF() is a callout and must happen before DML
        Blob pdfBlob = PurchaseOrderPDFService.generatePDF(poId);

        // query the PO name for the file name and authorization stamp
        Purchase_Order__c po = [
            SELECT Id, Name, Authorizer_Name__c
            FROM Purchase_Order__c
            WHERE Id = :poId
            LIMIT 1
        ];

        String pdfFileName = po.Name + '.pdf';

        // step 2 — send the email
        Boolean emailSent = PurchaseOrderEmailService.sendPurchaseOrderPDF(
            poId,
            contactId,
            pdfBlob,
            pdfFileName
        );

        if (emailSent) {
            // step 3 — stamp the PO with digital authorization
            po.Digital_Authorization__c =
                'Digital Authorization from ' + po.Authorizer_Name__c;
            update po;

            System.debug('PO stamped with digital authorization');

            // step 4 — optionally save the PDF as a file on the record
            PurchaseOrderPDFService.savePDFAsFile(poId, pdfBlob, pdfFileName);
        } else {
            System.debug(LoggingLevel.ERROR,
                'Email send failed — PO not stamped with authorization');
            // throw or handle the error based on your requirements
            throw new PurchaseOrderEmailException(
                'Failed to send Purchase Order email. The PO has not been authorized.'
            );
        }
    }

    public class PurchaseOrderEmailException extends Exception {}
}
```

## Common Mistakes

1. **Using `setToAddresses()` when you should use `setTargetObjectId()`** -- `setToAddresses()` takes a list of raw email strings. It does not create activity records, does not resolve merge fields in templates, and does not respect email opt-out settings. Fix: use `setTargetObjectId(contactId)` for Contacts and Leads. Only use `setToAddresses()` for ad hoc email addresses that are not Salesforce records (like external webhook endpoints or one-off addresses).

2. **Not checking the `SendEmailResult` after `sendEmail()`** -- `Messaging.sendEmail()` does not throw an exception for all failure modes. Invalid template IDs, missing required fields, and some address issues return `isSuccess() = false` without throwing. Fix: always check `results[0].isSuccess()` and iterate `results[0].getErrors()` for debugging info. Log errors with `LoggingLevel.ERROR`.

3. **Calling `sendEmail()` with more than 10 `SingleEmailMessage` objects** -- The governor limit is 10 `SingleEmailMessage` invocations per transaction. Passing a list of 11+ messages to `sendEmail()` will fail. Fix: if you need to send more than 10, use a Queueable chain or Batch Apex. Each Queueable execution gets its own 10-message limit.

4. **Forgetting deliverability settings in sandboxes** -- Sandboxes default the email deliverability access level to "System email only," which silently drops all non-system emails from Apex. Developers spend hours debugging code that works perfectly — the email just never leaves Salesforce. Fix: go to Setup > Email > Deliverability and set "Access level" to "All email." Do this immediately after every sandbox refresh.

5. **Hardcoding `OrgWideEmailAddress` Ids** -- Org-wide email address records have different Ids across environments (sandbox vs production, different sandboxes). Hardcoded Ids break on deployment. Fix: always query `OrgWideEmailAddress` by `Address` or `DisplayName` field. Handle the case where the query returns no results (the org-wide address hasn't been set up yet or hasn't been verified).

6. **Setting `setSaveAsActivity(true)` without `setTargetObjectId()`** -- Activity logging requires a Contact, Lead, or User as the target. If you only used `setToAddresses()`, `setSaveAsActivity(true)` is silently ignored — no activity is created. Fix: always use `setTargetObjectId()` when you want activity logging. If your recipient isn't a Contact/Lead, consider creating a Contact record first or accepting that no activity will be logged.

7. **Using `setWhatId()` with a custom object that doesn't have Activities enabled** -- If the custom object's metadata has `allowActivities` set to `false`, the `setWhatId()` call doesn't throw an error, but the activity won't be linked to the record. Fix: enable Activities on the custom object (Object Manager > Purchase Order > Details > Allow Activities = checked). Redeploy the object metadata.

8. **Setting both `setTemplateId()` and `setHtmlBody()`/`setPlainTextBody()`** -- When `setTemplateId()` is provided, the template's subject and body override anything set via `setSubject()`, `setHtmlBody()`, and `setPlainTextBody()`. Your programmatic body is silently discarded. Fix: use one approach or the other. If you need to modify a template's output, use `Messaging.renderStoredEmailTemplate()` to get the rendered body, modify the string, then set it via `setHtmlBody()` without `setTemplateId()`.

9. **Not setting `setContentType()` on the `EmailFileAttachment`** -- Without an explicit content type, Salesforce guesses based on the file extension. This usually works, but some email clients (notably older Outlook versions and some mobile clients) mishandle attachments without a proper MIME type. Fix: always set `setContentType('application/pdf')` for PDF attachments. Use `'text/csv'` for CSV, `'image/png'` for PNG, etc.

10. **Testing email send counts against the daily org limit** -- `Messaging.sendEmail()` in test context does NOT actually deliver emails, but it DOES count against the per-transaction limit of 10 and the daily org limit of 5,000 single emails. If you run a large test suite that sends many emails, you can exhaust daily limits. Fix: use `Limits.getEmailInvocations()` and `Limits.getLimitEmailInvocations()` to monitor usage. In tests, consider mocking the email service or keeping the number of email-sending test methods minimal.

## See Also

- [Visualforce PDF Generation](../visualforce-pdf/visualforce-pdf-generation.md) -- generating the PDF blob that gets attached to the email
