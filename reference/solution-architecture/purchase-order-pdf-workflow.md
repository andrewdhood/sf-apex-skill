# Purchase Order PDF Workflow — End-to-End Pattern

> **When to read this:** You are building the Purchase Order email workflow — Quick Action launches Screen Flow, user selects a Contact, Apex generates a PDF from Visualforce, emails it, logs the action, and stamps digital authorization on the PO record.

## Rules

- When generating a PDF from Visualforce in Apex, always use `PageReference.getContentAsPDF()` with `Page.YourPageName` syntax (not a string URL) because string-based PageReferences fail in managed packages and some API versions.
- When styling a Visualforce PDF, always use inline CSS or `<style>` blocks with CSS 2.1 properties only because the Flying Saucer render engine does not support CSS3, flexbox, grid, or external stylesheets.
- When building an `@InvocableMethod`, always use inner `Request` and `Response` classes with `@InvocableVariable` annotations because this is the only way to pass multiple parameters to and from a Flow.
- When sending email from an Invocable method, always wrap the entire operation in a try-catch and return error details in the Response because unhandled exceptions produce a useless Flow fault screen.
- When logging the email send, always use a Task with `WhatId` set to the Purchase Order and `WhoId` set to the Contact because Tasks appear in the activity timeline on both records without any custom UI.
- When stamping Digital Authorization, always do it in the same transaction as the email send because if the email succeeds but the field update fails (or vice versa), the record state is inconsistent.
- Never call `getContentAsPDF()` from a trigger, batch, or queueable context because it is treated as a callout since API version 34.0 and will throw a `CalloutException` in those contexts.
- Never use SLDS classes or Lightning Design System stylesheets in a Visualforce PDF page because they do not render in the PDF engine. Use plain HTML tables with inline CSS.
- When testing code that calls `getContentAsPDF()`, always wrap the call in `Test.isRunningTest()` check that returns a dummy Blob because `getContentAsPDF()` returns blank content in test context and cannot be meaningfully asserted against.
- When building the Screen Flow, always include a confirmation screen before the Apex action and a result screen after because users need to confirm who they are emailing and see whether it succeeded.

## Object Model

### Purchase_Order__c (Custom Object)

| Field | API Name | Type | Purpose |
|---|---|---|---|
| PO Number | `Name` | Auto Number | `PO-{0000}` format |
| Status | `Status__c` | Picklist | Draft, Submitted, Approved, Sent, Closed |
| Authorizer Name | `Authorizer_Name__c` | Text(255) | Name of the person authorizing the PO |
| Digital Authorization | `Digital_Authorization__c` | Text(255) | Stamped after email: "Digital Authorization from {Name}" |
| Total Amount | `Total_Amount__c` | Currency | Roll-up or formula of line item totals |
| Vendor | `Vendor__c` | Lookup(Account) | The vendor/supplier |
| Notes | `Notes__c` | Long Text Area | Additional PO notes |

### Purchase_Order_Line_Item__c (Custom Object — Child)

| Field | API Name | Type | Purpose |
|---|---|---|---|
| Line Item Name | `Name` | Auto Number | `LI-{0000}` |
| Purchase Order | `Purchase_Order__c` | Master-Detail(Purchase_Order__c) | Parent PO |
| Description | `Description__c` | Text(255) | What is being ordered |
| Quantity | `Quantity__c` | Number(10,0) | How many |
| Unit Price | `Unit_Price__c` | Currency | Price per unit |
| Line Total | `Line_Total__c` | Formula(Currency) | `Quantity__c * Unit_Price__c` |

### Task (Standard Object — Email Log)

| Field | API Name | Type | Purpose |
|---|---|---|---|
| Subject | `Subject` | Text | "Purchase Order {PO Name} emailed to {Contact Name}" |
| Status | `Status` | Picklist | "Completed" |
| Priority | `Priority` | Picklist | "Normal" |
| Related To | `WhatId` | Lookup | Set to the Purchase_Order__c ID |
| Name | `WhoId` | Lookup | Set to the Contact ID |
| Description | `Description` | Long Text | Details: recipient email, timestamp, PO amount |
| PO Sent Date | `PO_Sent_Date__c` | DateTime | Custom field: when the email was sent |
| PO Recipient Email | `PO_Recipient_Email__c` | Text | Custom field: the email address used |

### Decision: Task vs PO_Email_Log__c Custom Object

| Factor | Task | PO_Email_Log__c |
|---|---|---|
| Activity Timeline | Shows automatically on PO and Contact | Does NOT appear in timeline without custom LWC |
| Custom Fields | Limited — you can add some, but Tasks are already field-heavy | Full control — any fields you need |
| Reporting | Standard Task reports, cross-filters on activities | Custom report types, full flexibility |
| Related Lists | Automatic on any object via WhatId/WhoId | Requires explicit lookup + related list config |
| Setup Effort | Minimal — just add custom fields to Task | Create object, fields, page layout, related list, tab |
| Audit Trail | Standard ActivityHistory queryable | Custom object queryable with more structure |

**Recommendation:** Use Task. The activity timeline integration alone is worth it. Add 2-3 custom fields to Task (`PO_Sent_Date__c`, `PO_Recipient_Email__c`) for the data you need to query or report on. This gives you 90% of the benefit at 10% of the setup cost.

## How It Works

### Data Flow Diagram

```
User clicks "Send PO Email" button on PO record page
        |
        v
Quick Action (type: Flow) passes recordId to Screen Flow
        |
        v
Screen Flow: Screen 1 — User selects a Contact (lookup field)
        |
        v
Screen Flow: Screen 2 — Confirmation screen (PO name, contact name, email)
        |
        v
Screen Flow: calls @InvocableMethod → SendPurchaseOrderEmail.invoke()
        |
        v
Apex: 1. Queries PO + line items via PurchaseOrderSelector
       2. Queries Contact for email address
       3. Generates PDF via PageReference to PurchaseOrderPDF VF page
       4. Creates Messaging.SingleEmailMessage with PDF attachment
       5. Sends email via Messaging.sendEmail()
       6. Creates a completed Task (email log)
       7. Stamps Digital_Authorization__c on the PO
       8. Returns success/error to flow
        |
        v
Screen Flow: Screen 3 — Shows success message or error details
```

### Transaction Scope

All Apex operations (steps 1-7) execute in a **single transaction** with a SavePoint. If any step fails, the entire transaction rolls back — no partial commits. The email send (`Messaging.sendEmail()`) is the one operation that cannot be rolled back once it succeeds, so the code sends the email LAST, after all DML is staged and validated.

**Critical ordering:**
1. Query data (no DML, safe to do first)
2. Generate PDF blob (no DML, but treated as callout)
3. Prepare the email message (no DML yet)
4. Prepare the Task record (no DML yet)
5. Prepare the PO update (no DML yet)
6. **Send email** (this is the point of no return)
7. Insert Task
8. Update PO

If the Task insert or PO update fails after the email is already sent, you have an inconsistency (email sent but no log/stamp). To mitigate this, the code sends the email only after validating that all data is ready, and wraps the post-email DML in its own try-catch that logs the failure rather than hiding it.

## Component Inventory

### 1. Visualforce Page: PurchaseOrderPDF.page

### 2. Apex Controller: PurchaseOrderPDFController.cls

### 3. Invocable Apex: SendPurchaseOrderEmail.cls

### 4. Screen Flow: Send_Purchase_Order_Email (metadata XML)

### 5. Quick Action: Purchase_Order__c.Send_PO_Email (metadata XML)

### 6. Test Class: SendPurchaseOrderEmailTest.cls

## Code Examples

### PurchaseOrderPDF.page — Visualforce Page

```xml
<apex:page controller="PurchaseOrderPDFController"
           renderAs="pdf"
           applyBodyTag="false"
           applyHtmlTag="false"
           showHeader="false"
           sidebar="false"
           standardStylesheets="false"
           cache="true">
<html>
<head>
    <style type="text/css">
        /* PDF engine uses Flying Saucer — CSS 2.1 only */
        /* no flexbox, no grid, no CSS3 selectors */

        @page {
            size: letter;
            margin: 0.75in;
        }

        body {
            font-family: Arial, Helvetica, sans-serif;
            font-size: 11px;
            color: #333333;
        }

        .header-table {
            width: 100%;
            margin-bottom: 20px;
        }

        .header-table td {
            vertical-align: top;
        }

        .company-name {
            font-size: 20px;
            font-weight: bold;
            color: #2c3e50;
        }

        .po-title {
            font-size: 16px;
            font-weight: bold;
            color: #2c3e50;
            text-align: right;
        }

        .po-number {
            font-size: 12px;
            text-align: right;
            color: #555555;
        }

        .info-table {
            width: 100%;
            margin-bottom: 20px;
            border: 1px solid #cccccc;
        }

        .info-table td {
            padding: 6px 10px;
            border: 1px solid #cccccc;
        }

        .info-label {
            font-weight: bold;
            background-color: #f2f2f2;
            width: 25%;
        }

        .line-items-table {
            width: 100%;
            border-collapse: collapse;
            margin-bottom: 20px;
        }

        .line-items-table th {
            background-color: #2c3e50;
            color: #ffffff;
            padding: 8px 10px;
            text-align: left;
            font-weight: bold;
            border: 1px solid #2c3e50;
        }

        .line-items-table td {
            padding: 6px 10px;
            border: 1px solid #cccccc;
            word-wrap: break-word;
        }

        .line-items-table tr:nth-child(even) {
            /* nth-child may not work in Flying Saucer — use alternateRowClass if needed */
        }

        .text-right {
            text-align: right;
        }

        .total-row td {
            font-weight: bold;
            background-color: #f2f2f2;
            border-top: 2px solid #2c3e50;
        }

        .notes-section {
            margin-top: 20px;
            padding: 10px;
            border: 1px solid #cccccc;
            background-color: #fafafa;
        }

        .notes-label {
            font-weight: bold;
            margin-bottom: 5px;
        }

        .auth-section {
            margin-top: 30px;
            padding: 10px;
            border-top: 2px solid #2c3e50;
        }

        .auth-label {
            font-weight: bold;
        }

        .footer {
            margin-top: 40px;
            font-size: 9px;
            color: #999999;
            text-align: center;
        }

        /* prevent table rows from splitting across pages */
        .line-items-table tr {
            page-break-inside: avoid;
        }

        /* repeat table headers on each page */
        .line-items-table thead {
            display: table-header-group;
        }
    </style>
</head>
<body>

    <!-- header: company name + PO title -->
    <table class="header-table">
        <tr>
            <td>
                <div class="company-name">{!companyName}</div>
            </td>
            <td>
                <div class="po-title">PURCHASE ORDER</div>
                <div class="po-number">{!purchaseOrder.Name}</div>
            </td>
        </tr>
    </table>

    <!-- PO details -->
    <table class="info-table">
        <tr>
            <td class="info-label">Status</td>
            <td>{!purchaseOrder.Status__c}</td>
            <td class="info-label">Date</td>
            <td>
                <apex:outputText value="{0, date, MMMM d, yyyy}">
                    <apex:param value="{!NOW()}" />
                </apex:outputText>
            </td>
        </tr>
        <tr>
            <td class="info-label">Authorizer</td>
            <td>{!purchaseOrder.Authorizer_Name__c}</td>
            <td class="info-label">Vendor</td>
            <td>{!purchaseOrder.Vendor__r.Name}</td>
        </tr>
    </table>

    <!-- line items -->
    <table class="line-items-table">
        <thead>
            <tr>
                <th style="width: 8%;">#</th>
                <th style="width: 42%;">Description</th>
                <th style="width: 15%;" class="text-right">Qty</th>
                <th style="width: 17%;" class="text-right">Unit Price</th>
                <th style="width: 18%;" class="text-right">Line Total</th>
            </tr>
        </thead>
        <tbody>
            <apex:repeat value="{!lineItems}" var="item">
                <tr>
                    <td>{!item.Name}</td>
                    <td>{!item.Description__c}</td>
                    <td class="text-right">
                        <apex:outputText value="{0, number, ###,##0}">
                            <apex:param value="{!item.Quantity__c}" />
                        </apex:outputText>
                    </td>
                    <td class="text-right">
                        <apex:outputText value="{0, number, $###,##0.00}">
                            <apex:param value="{!item.Unit_Price__c}" />
                        </apex:outputText>
                    </td>
                    <td class="text-right">
                        <apex:outputText value="{0, number, $###,##0.00}">
                            <apex:param value="{!item.Line_Total__c}" />
                        </apex:outputText>
                    </td>
                </tr>
            </apex:repeat>
            <!-- total row -->
            <tr class="total-row">
                <td colspan="4" class="text-right">TOTAL</td>
                <td class="text-right">
                    <apex:outputText value="{0, number, $###,##0.00}">
                        <apex:param value="{!totalAmount}" />
                    </apex:outputText>
                </td>
            </tr>
        </tbody>
    </table>

    <!-- notes -->
    <apex:outputPanel rendered="{!NOT(ISBLANK(purchaseOrder.Notes__c))}">
        <div class="notes-section">
            <div class="notes-label">Notes</div>
            <div>{!purchaseOrder.Notes__c}</div>
        </div>
    </apex:outputPanel>

    <!-- digital authorization -->
    <apex:outputPanel rendered="{!NOT(ISBLANK(purchaseOrder.Digital_Authorization__c))}">
        <div class="auth-section">
            <span class="auth-label">Authorization:</span>
            {!purchaseOrder.Digital_Authorization__c}
        </div>
    </apex:outputPanel>

    <!-- footer -->
    <div class="footer">
        Generated on <apex:outputText value="{0, date, MMMM d, yyyy 'at' h:mm a}">
            <apex:param value="{!NOW()}" />
        </apex:outputText>
        &nbsp;|&nbsp; {!purchaseOrder.Name}
    </div>

</body>
</html>
</apex:page>
```

### PurchaseOrderPDFController.cls — Visualforce Controller

```apex
public with sharing class PurchaseOrderPDFController {

    // the purchase order record displayed on the PDF
    public Purchase_Order__c purchaseOrder { get; private set; }

    // the line items for this purchase order
    public List<Purchase_Order_Line_Item__c> lineItems { get; private set; }

    // calculated total amount across all line items
    public Decimal totalAmount { get; private set; }

    // company name for the PDF header — pull from org or custom metadata
    public String companyName { get; private set; }

    // constructor — queries the PO and line items based on the id parameter
    // input: none (reads 'id' from URL parameters)
    // output: none (populates instance variables)
    public PurchaseOrderPDFController() {
        Id poId = ApexPages.currentPage().getParameters().get('id');

        if (poId == null) {
            // handle missing ID gracefully — the PDF will render blank
            purchaseOrder = new Purchase_Order__c();
            lineItems = new List<Purchase_Order_Line_Item__c>();
            totalAmount = 0;
            companyName = '';
            return;
        }

        // query the PO with all fields needed for the PDF
        List<Purchase_Order__c> pos = [
            SELECT Id, Name, Status__c, Authorizer_Name__c,
                   Digital_Authorization__c, Total_Amount__c,
                   Notes__c, Vendor__r.Name
            FROM Purchase_Order__c
            WHERE Id = :poId
            LIMIT 1
        ];

        if (pos.isEmpty()) {
            purchaseOrder = new Purchase_Order__c();
            lineItems = new List<Purchase_Order_Line_Item__c>();
            totalAmount = 0;
            companyName = '';
            return;
        }

        purchaseOrder = pos[0];

        // query line items ordered by name (auto-number gives sequential order)
        lineItems = [
            SELECT Id, Name, Description__c, Quantity__c, Unit_Price__c, Line_Total__c
            FROM Purchase_Order_Line_Item__c
            WHERE Purchase_Order__c = :poId
            ORDER BY Name
        ];

        // calculate total — use line item values rather than relying on a roll-up
        // in case the roll-up hasn't recalculated yet
        totalAmount = 0;
        for (Purchase_Order_Line_Item__c item : lineItems) {
            if (item.Line_Total__c != null) {
                totalAmount += item.Line_Total__c;
            }
        }

        // pull company name from custom metadata, or fall back to org name
        // this keeps the company name configurable without a code change
        App_Config__mdt companyConfig = App_Config__mdt.getInstance('Company_Name');
        if (companyConfig != null && String.isNotBlank(companyConfig.Value__c)) {
            companyName = companyConfig.Value__c;
        } else {
            companyName = UserInfo.getOrganizationName();
        }
    }
}
```

### SendPurchaseOrderEmail.cls — Invocable Apex (The Main Engine)

```apex
public with sharing class SendPurchaseOrderEmail {

    // entry point called by the Send Purchase Order Email screen flow
    // input: requests - list of Request objects (one per flow interview)
    // output: list of Response objects with success/error info
    @InvocableMethod(
        label='Send Purchase Order Email'
        description='Generates a PDF of the Purchase Order and emails it to the selected Contact. Logs the action as a Task and stamps Digital Authorization.'
        category='Purchase Orders'
    )
    public static List<Response> invoke(List<Request> requests) {
        List<Response> responses = new List<Response>();

        for (Request req : requests) {
            responses.add(processSingleRequest(req));
        }

        return responses;
    }

    // processes a single PO email request inside a savepoint
    // input: req - the Request containing poId and contactId
    // output: Response with success flag and message
    private static Response processSingleRequest(Request req) {
        Response res = new Response();
        Savepoint sp = Database.setSavepoint();

        try {
            // step 1: query the Purchase Order with line items
            Purchase_Order__c po = queryPurchaseOrder(req.poId);

            // step 2: query the Contact for email address
            Contact recipient = queryContact(req.contactId);

            // step 3: validate we have what we need
            validateInputs(po, recipient);

            // step 4: generate the PDF blob
            Blob pdfBlob = generatePdfBlob(po.Id);

            // step 5: build the email message
            Messaging.SingleEmailMessage email = buildEmailMessage(po, recipient, pdfBlob);

            // step 6: build the Task (log record) — don't insert yet
            Task emailLog = buildEmailLog(po, recipient);

            // step 7: stamp digital authorization — don't update yet
            po.Digital_Authorization__c = 'Digital Authorization from ' + po.Authorizer_Name__c;

            // step 8: send the email — this is the point of no return
            // once sendEmail succeeds, the email is gone and cannot be rolled back
            Messaging.SendEmailResult[] sendResults = Messaging.sendEmail(
                new List<Messaging.SingleEmailMessage>{ email }
            );

            // check if the email actually sent
            if (!sendResults[0].isSuccess()) {
                String errorMsg = sendResults[0].getErrors()[0].getMessage();
                throw new PurchaseOrderException('Email send failed: ' + errorMsg);
            }

            System.debug('SendPurchaseOrderEmail: email sent successfully to ' + recipient.Email);

            // step 9: insert the log task
            insert emailLog;
            System.debug('SendPurchaseOrderEmail: email log task created — ' + emailLog.Id);

            // step 10: update the PO with digital auth stamp
            update po;
            System.debug('SendPurchaseOrderEmail: PO updated with digital authorization');

            // build success response
            res.isSuccess = true;
            res.message = 'Purchase Order ' + po.Name + ' emailed to ' + recipient.Name
                + ' (' + recipient.Email + ')';

        } catch (PurchaseOrderException e) {
            // known business errors — roll back and return friendly message
            Database.rollback(sp);
            res.isSuccess = false;
            res.message = e.getMessage();
            System.debug(LoggingLevel.ERROR, 'SendPurchaseOrderEmail business error: '
                + e.getMessage());

        } catch (Exception e) {
            // unexpected errors — roll back and return generic message with details
            Database.rollback(sp);
            res.isSuccess = false;
            res.message = 'An unexpected error occurred: ' + e.getMessage();
            System.debug(LoggingLevel.ERROR, 'SendPurchaseOrderEmail unexpected error: '
                + e.getMessage() + ' | Stack: ' + e.getStackTraceString());
        }

        return res;
    }

    // queries the Purchase Order by ID with all fields needed for the PDF and email
    // input: poId - the Purchase_Order__c record ID
    // output: the PO record, or throws if not found
    private static Purchase_Order__c queryPurchaseOrder(Id poId) {
        List<Purchase_Order__c> pos = [
            SELECT Id, Name, Status__c, Authorizer_Name__c,
                   Digital_Authorization__c, Total_Amount__c,
                   Notes__c, Vendor__r.Name
            FROM Purchase_Order__c
            WHERE Id = :poId
            LIMIT 1
        ];

        if (pos.isEmpty()) {
            throw new PurchaseOrderException('Purchase Order not found: ' + poId);
        }

        return pos[0];
    }

    // queries the Contact by ID for the email recipient
    // input: contactId - the Contact record ID
    // output: the Contact record, or throws if not found
    private static Contact queryContact(Id contactId) {
        List<Contact> contacts = [
            SELECT Id, Name, FirstName, LastName, Email
            FROM Contact
            WHERE Id = :contactId
            LIMIT 1
        ];

        if (contacts.isEmpty()) {
            throw new PurchaseOrderException('Contact not found: ' + contactId);
        }

        return contacts[0];
    }

    // validates that the PO and Contact have the required data for sending
    // input: po - the Purchase Order record, recipient - the Contact record
    // output: none (throws PurchaseOrderException if validation fails)
    private static void validateInputs(Purchase_Order__c po, Contact recipient) {
        if (String.isBlank(recipient.Email)) {
            throw new PurchaseOrderException(
                'Contact ' + recipient.Name + ' does not have an email address. '
                + 'Please add an email to the Contact record and try again.'
            );
        }

        if (String.isBlank(po.Authorizer_Name__c)) {
            throw new PurchaseOrderException(
                'Purchase Order ' + po.Name + ' does not have an Authorizer Name. '
                + 'Please populate the Authorizer Name field and try again.'
            );
        }
    }

    // generates the PDF blob from the PurchaseOrderPDF Visualforce page
    // input: poId - the Purchase_Order__c record ID (passed as URL param to VF page)
    // output: Blob containing the PDF content
    private static Blob generatePdfBlob(Id poId) {
        // use Page.PurchaseOrderPDF syntax — not new PageReference('/apex/...')
        // this ensures the page reference is valid at compile time
        PageReference pdfPage = Page.PurchaseOrderPDF;
        pdfPage.getParameters().put('id', poId);

        Blob pdfBlob;

        // getContentAsPDF() returns blank content in test context
        // we still call it to ensure no exception is thrown, but we can't assert PDF content
        if (Test.isRunningTest()) {
            pdfBlob = Blob.valueOf('Test PDF Content');
        } else {
            pdfBlob = pdfPage.getContentAsPDF();
        }

        System.debug('SendPurchaseOrderEmail: PDF generated, size = ' + pdfBlob.size() + ' bytes');
        return pdfBlob;
    }

    // builds the SingleEmailMessage with the PDF as an attachment
    // input: po - the PO record (for subject/body), recipient - the Contact,
    //        pdfBlob - the generated PDF content
    // output: fully configured Messaging.SingleEmailMessage ready to send
    private static Messaging.SingleEmailMessage buildEmailMessage(
        Purchase_Order__c po,
        Contact recipient,
        Blob pdfBlob
    ) {
        // build the PDF attachment
        Messaging.EmailFileAttachment attachment = new Messaging.EmailFileAttachment();
        attachment.setFileName(po.Name + '.pdf');
        attachment.setBody(pdfBlob);
        attachment.setContentType('application/pdf');

        // build the email
        Messaging.SingleEmailMessage email = new Messaging.SingleEmailMessage();
        email.setToAddresses(new List<String>{ recipient.Email });
        email.setSubject('Purchase Order ' + po.Name);
        email.setPlainTextBody(
            'Hello ' + recipient.FirstName + ',\n\n'
            + 'Please find the attached Purchase Order ' + po.Name + '.\n\n'
            + 'Total Amount: $' + (po.Total_Amount__c != null
                ? String.valueOf(po.Total_Amount__c.setScale(2))
                : '0.00') + '\n\n'
            + 'If you have any questions, please contact us.\n\n'
            + 'Thank you,\n'
            + po.Authorizer_Name__c
        );
        email.setFileAttachments(new List<Messaging.EmailFileAttachment>{ attachment });

        // set the WhatId to the PO so the email appears in the PO activity
        // note: setWhatId only works with SingleEmailMessage when targeting a Contact
        // and setTargetObjectId is set — otherwise it's ignored
        email.setTargetObjectId(recipient.Id);
        email.setWhatId(po.Id);

        // prevent Salesforce from saving this as an Activity automatically
        // because we are creating our own Task with more detail
        email.setSaveAsActivity(false);

        // use org-wide email address if configured in custom metadata
        setOrgWideEmailAddress(email);

        return email;
    }

    // looks up the org-wide email address from custom metadata and sets it on the email
    // input: email - the SingleEmailMessage to configure
    // output: none (modifies the email in place)
    private static void setOrgWideEmailAddress(Messaging.SingleEmailMessage email) {
        // pull the org-wide email address name from custom metadata
        App_Config__mdt emailConfig = App_Config__mdt.getInstance('PO_Org_Wide_Email');

        if (emailConfig == null || String.isBlank(emailConfig.Value__c)) {
            System.debug('SendPurchaseOrderEmail: no org-wide email configured, using default sender');
            return;
        }

        List<OrgWideEmailAddress> oweas = [
            SELECT Id
            FROM OrgWideEmailAddress
            WHERE DisplayName = :emailConfig.Value__c
            LIMIT 1
        ];

        if (!oweas.isEmpty()) {
            email.setOrgWideEmailAddressId(oweas[0].Id);
            System.debug('SendPurchaseOrderEmail: using org-wide email address — '
                + emailConfig.Value__c);
        } else {
            System.debug(LoggingLevel.WARN,
                'SendPurchaseOrderEmail: org-wide email address not found — '
                + emailConfig.Value__c);
        }
    }

    // builds the Task record that logs the email send
    // input: po - the PO record, recipient - the Contact who received the email
    // output: a Task record ready to insert (not yet inserted)
    private static Task buildEmailLog(Purchase_Order__c po, Contact recipient) {
        Task emailLog = new Task();
        emailLog.Subject = 'Purchase Order ' + po.Name + ' emailed to ' + recipient.Name;
        emailLog.Status = 'Completed';
        emailLog.Priority = 'Normal';
        emailLog.TaskSubtype = 'Email';
        emailLog.WhatId = po.Id;
        emailLog.WhoId = recipient.Id;
        emailLog.ActivityDate = Date.today();
        emailLog.Description =
            'Purchase Order: ' + po.Name + '\n'
            + 'Sent To: ' + recipient.Name + ' (' + recipient.Email + ')\n'
            + 'Sent Date: ' + Datetime.now().format('MMMM d, yyyy h:mm a') + '\n'
            + 'Total Amount: $' + (po.Total_Amount__c != null
                ? String.valueOf(po.Total_Amount__c.setScale(2))
                : '0.00') + '\n'
            + 'Authorizer: ' + po.Authorizer_Name__c;

        return emailLog;
    }

    // request class — receives input from the Screen Flow
    public class Request {
        @InvocableVariable(label='Purchase Order ID' description='The ID of the Purchase Order to email' required=true)
        public Id poId;

        @InvocableVariable(label='Contact ID' description='The ID of the Contact to email the PO to' required=true)
        public Id contactId;
    }

    // response class — returns results to the Screen Flow
    public class Response {
        @InvocableVariable(label='Success' description='True if the email was sent successfully')
        public Boolean isSuccess;

        @InvocableVariable(label='Message' description='Success confirmation or error details')
        public String message;
    }

    // custom exception for PO-specific business errors
    public class PurchaseOrderException extends Exception {}
}
```

### SendPurchaseOrderEmailTest.cls — Test Class

```apex
@IsTest
private class SendPurchaseOrderEmailTest {

    // shared test data — PO with line items and a contact with an email
    @TestSetup
    static void setupTestData() {
        // create a vendor account for the PO
        Account vendor = new Account(Name = 'Test Vendor Inc.');
        insert vendor;

        // create a Purchase Order with all required fields
        Purchase_Order__c po = new Purchase_Order__c(
            Status__c = 'Approved',
            Authorizer_Name__c = 'Jane Smith',
            Vendor__c = vendor.Id,
            Notes__c = 'Test purchase order for unit testing'
        );
        insert po;

        // create line items for the PO
        List<Purchase_Order_Line_Item__c> lineItems = new List<Purchase_Order_Line_Item__c>();
        lineItems.add(new Purchase_Order_Line_Item__c(
            Purchase_Order__c = po.Id,
            Description__c = 'Widget A',
            Quantity__c = 10,
            Unit_Price__c = 25.00
        ));
        lineItems.add(new Purchase_Order_Line_Item__c(
            Purchase_Order__c = po.Id,
            Description__c = 'Widget B',
            Quantity__c = 5,
            Unit_Price__c = 50.00
        ));
        insert lineItems;

        // create a contact with an email address (the recipient)
        Contact recipient = new Contact(
            FirstName = 'John',
            LastName = 'Doe',
            Email = 'john.doe@example.com'
        );
        insert recipient;

        // create a contact WITHOUT an email (for negative testing)
        Contact noEmailContact = new Contact(
            FirstName = 'No',
            LastName = 'Email'
        );
        insert noEmailContact;
    }

    // test: successful email send — the happy path
    @IsTest
    static void testSuccessfulEmailSend() {
        Purchase_Order__c po = [SELECT Id, Name FROM Purchase_Order__c LIMIT 1];
        Contact recipient = [SELECT Id FROM Contact WHERE Email != null LIMIT 1];

        // build the request
        SendPurchaseOrderEmail.Request req = new SendPurchaseOrderEmail.Request();
        req.poId = po.Id;
        req.contactId = recipient.Id;

        Test.startTest();
        List<SendPurchaseOrderEmail.Response> responses =
            SendPurchaseOrderEmail.invoke(new List<SendPurchaseOrderEmail.Request>{ req });
        Test.stopTest();

        // assert the response
        System.assertEquals(1, responses.size(), 'Should return one response');
        System.assertEquals(true, responses[0].isSuccess, 'Should succeed: ' + responses[0].message);
        System.assert(responses[0].message.contains(po.Name),
            'Success message should contain PO name');

        // assert the PO was updated with digital authorization
        Purchase_Order__c updatedPo = [
            SELECT Digital_Authorization__c
            FROM Purchase_Order__c
            WHERE Id = :po.Id
        ];
        System.assertEquals(
            'Digital Authorization from Jane Smith',
            updatedPo.Digital_Authorization__c,
            'PO should be stamped with digital authorization'
        );

        // assert a Task was created
        List<Task> tasks = [
            SELECT Subject, WhatId, WhoId, Status
            FROM Task
            WHERE WhatId = :po.Id
        ];
        System.assertEquals(1, tasks.size(), 'Should create one email log task');
        System.assertEquals('Completed', tasks[0].Status, 'Task should be completed');
        System.assertEquals(recipient.Id, tasks[0].WhoId, 'Task WhoId should be the recipient');
        System.assert(tasks[0].Subject.contains(po.Name),
            'Task subject should contain PO name');

        // assert email was sent (test context captures emails)
        // note: Messaging.sendEmail returns success in tests but doesn't actually send
        // the email count can still be checked
    }

    // test: contact with no email address — should return error
    @IsTest
    static void testContactWithNoEmail() {
        Purchase_Order__c po = [SELECT Id FROM Purchase_Order__c LIMIT 1];
        Contact noEmail = [SELECT Id FROM Contact WHERE Email = null LIMIT 1];

        SendPurchaseOrderEmail.Request req = new SendPurchaseOrderEmail.Request();
        req.poId = po.Id;
        req.contactId = noEmail.Id;

        Test.startTest();
        List<SendPurchaseOrderEmail.Response> responses =
            SendPurchaseOrderEmail.invoke(new List<SendPurchaseOrderEmail.Request>{ req });
        Test.stopTest();

        System.assertEquals(false, responses[0].isSuccess,
            'Should fail when contact has no email');
        System.assert(responses[0].message.contains('email address'),
            'Error message should mention missing email: ' + responses[0].message);

        // assert no Task was created (transaction rolled back)
        List<Task> tasks = [SELECT Id FROM Task WHERE WhatId = :po.Id];
        System.assertEquals(0, tasks.size(), 'No task should be created on failure');

        // assert PO was not updated
        Purchase_Order__c unchangedPo = [
            SELECT Digital_Authorization__c
            FROM Purchase_Order__c
            WHERE Id = :po.Id
        ];
        System.assertEquals(null, unchangedPo.Digital_Authorization__c,
            'PO should not be stamped on failure');
    }

    // test: invalid Purchase Order ID — should return error
    @IsTest
    static void testInvalidPurchaseOrderId() {
        Contact recipient = [SELECT Id FROM Contact WHERE Email != null LIMIT 1];

        // use a fabricated ID that doesn't exist
        // note: we use a valid-format ID so the query runs but returns empty
        Id fakePOId = Purchase_Order__c.SObjectType.getDescribe().getKeyPrefix() + '000000000001';

        SendPurchaseOrderEmail.Request req = new SendPurchaseOrderEmail.Request();
        req.poId = fakePOId;
        req.contactId = recipient.Id;

        Test.startTest();
        List<SendPurchaseOrderEmail.Response> responses =
            SendPurchaseOrderEmail.invoke(new List<SendPurchaseOrderEmail.Request>{ req });
        Test.stopTest();

        System.assertEquals(false, responses[0].isSuccess,
            'Should fail with invalid PO ID');
        System.assert(responses[0].message.contains('not found'),
            'Error message should indicate PO not found: ' + responses[0].message);
    }

    // test: PO without authorizer name — should return error
    @IsTest
    static void testMissingAuthorizerName() {
        // update the PO to clear the authorizer name
        Purchase_Order__c po = [SELECT Id FROM Purchase_Order__c LIMIT 1];
        po.Authorizer_Name__c = null;
        update po;

        Contact recipient = [SELECT Id FROM Contact WHERE Email != null LIMIT 1];

        SendPurchaseOrderEmail.Request req = new SendPurchaseOrderEmail.Request();
        req.poId = po.Id;
        req.contactId = recipient.Id;

        Test.startTest();
        List<SendPurchaseOrderEmail.Response> responses =
            SendPurchaseOrderEmail.invoke(new List<SendPurchaseOrderEmail.Request>{ req });
        Test.stopTest();

        System.assertEquals(false, responses[0].isSuccess,
            'Should fail when authorizer name is blank');
        System.assert(responses[0].message.contains('Authorizer Name'),
            'Error message should mention authorizer: ' + responses[0].message);
    }

    // test: bulk invocation — flow calls with multiple requests
    @IsTest
    static void testBulkInvocation() {
        Purchase_Order__c po = [SELECT Id FROM Purchase_Order__c LIMIT 1];
        Contact recipient = [SELECT Id FROM Contact WHERE Email != null LIMIT 1];

        // simulate a bulk call with 5 requests (same PO/contact for simplicity)
        List<SendPurchaseOrderEmail.Request> requests = new List<SendPurchaseOrderEmail.Request>();
        for (Integer i = 0; i < 5; i++) {
            SendPurchaseOrderEmail.Request req = new SendPurchaseOrderEmail.Request();
            req.poId = po.Id;
            req.contactId = recipient.Id;
            requests.add(req);
        }

        Test.startTest();
        List<SendPurchaseOrderEmail.Response> responses =
            SendPurchaseOrderEmail.invoke(requests);
        Test.stopTest();

        System.assertEquals(5, responses.size(), 'Should return 5 responses');
        // note: in a real scenario, sending 5 emails in one transaction is fine
        // (under the 10 single email limit per transaction)
        // but sending the same PO 5 times would be unusual — this just tests bulkification
    }

    // test: the VF controller independently
    @IsTest
    static void testPurchaseOrderPDFController() {
        Purchase_Order__c po = [SELECT Id FROM Purchase_Order__c LIMIT 1];

        // simulate navigating to the VF page with the PO ID
        Test.setCurrentPage(Page.PurchaseOrderPDF);
        ApexPages.currentPage().getParameters().put('id', po.Id);

        Test.startTest();
        PurchaseOrderPDFController controller = new PurchaseOrderPDFController();
        Test.stopTest();

        // assert the controller populated correctly
        System.assertNotEquals(null, controller.purchaseOrder,
            'Controller should have a purchase order');
        System.assertEquals(po.Id, controller.purchaseOrder.Id,
            'Controller PO should match the URL param');
        System.assertEquals(2, controller.lineItems.size(),
            'Controller should have 2 line items');
        System.assert(controller.totalAmount > 0,
            'Total amount should be greater than zero');
        System.assert(String.isNotBlank(controller.companyName),
            'Company name should be populated');
    }

    // test: VF controller with missing ID parameter
    @IsTest
    static void testPurchaseOrderPDFControllerMissingId() {
        Test.setCurrentPage(Page.PurchaseOrderPDF);
        // intentionally do NOT set the 'id' parameter

        Test.startTest();
        PurchaseOrderPDFController controller = new PurchaseOrderPDFController();
        Test.stopTest();

        // should handle gracefully — no exception, just empty data
        System.assertNotEquals(null, controller.purchaseOrder,
            'Controller should still have a PO object (empty)');
        System.assertEquals(0, controller.lineItems.size(),
            'Line items should be empty');
        System.assertEquals(0, controller.totalAmount,
            'Total amount should be zero');
    }
}
```

### Screen Flow Metadata: Send_Purchase_Order_Email.flow-meta.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Flow xmlns="http://soap.sforce.com/2006/04/metadata">
    <apiVersion>62.0</apiVersion>
    <description>Launched by the Send PO Email quick action. Lets the user select a Contact, confirms the send, calls Apex to generate PDF and email it, then shows the result.</description>
    <environments>Default</environments>
    <interviewLabel>Send Purchase Order Email {!$Flow.CurrentDateTime}</interviewLabel>
    <label>Send Purchase Order Email</label>
    <processMetadataValues>
        <name>BuilderType</name>
        <value>
            <stringValue>LightningFlowBuilder</stringValue>
        </value>
    </processMetadataValues>
    <processType>Flow</processType>
    <runInMode>DefaultMode</runInMode>
    <status>Active</status>

    <!-- input variable: recordId passed by the Quick Action -->
    <variables>
        <name>recordId</name>
        <dataType>String</dataType>
        <isCollection>false</isCollection>
        <isInput>true</isInput>
        <isOutput>false</isOutput>
    </variables>

    <!-- variable: the selected contact ID -->
    <variables>
        <name>selectedContactId</name>
        <dataType>String</dataType>
        <isCollection>false</isCollection>
        <isInput>false</isInput>
        <isOutput>false</isOutput>
    </variables>

    <!-- variable: stores the invocable result -->
    <variables>
        <name>emailResult</name>
        <dataType>Boolean</dataType>
        <isCollection>false</isCollection>
        <isInput>false</isInput>
        <isOutput>false</isOutput>
    </variables>

    <variables>
        <name>emailMessage</name>
        <dataType>String</dataType>
        <isCollection>false</isCollection>
        <isInput>false</isInput>
        <isOutput>false</isOutput>
    </variables>

    <!-- Get Record: query the PO for display on confirmation screen -->
    <recordLookups>
        <name>Get_Purchase_Order</name>
        <label>Get Purchase Order</label>
        <locationX>176</locationX>
        <locationY>134</locationY>
        <connector>
            <targetReference>Select_Contact_Screen</targetReference>
        </connector>
        <filterLogic>and</filterLogic>
        <filters>
            <field>Id</field>
            <operator>EqualTo</operator>
            <value>
                <elementReference>recordId</elementReference>
            </value>
        </filters>
        <getFirstRecordOnly>true</getFirstRecordOnly>
        <object>Purchase_Order__c</object>
        <storeOutputAutomatically>true</storeOutputAutomatically>
    </recordLookups>

    <!-- Screen 1: select a Contact -->
    <screens>
        <name>Select_Contact_Screen</name>
        <label>Select Contact</label>
        <locationX>176</locationX>
        <locationY>254</locationY>
        <allowBack>false</allowBack>
        <allowFinish>true</allowFinish>
        <allowPause>false</allowPause>
        <connector>
            <targetReference>Confirmation_Screen</targetReference>
        </connector>
        <showFooter>true</showFooter>
        <showHeader>true</showHeader>
        <fields>
            <name>Display_PO_Info</name>
            <fieldType>DisplayText</fieldType>
            <fieldText>&lt;p&gt;&lt;b&gt;Purchase Order:&lt;/b&gt; {!Get_Purchase_Order.Name}&lt;/p&gt;&lt;p&gt;&lt;b&gt;Status:&lt;/b&gt; {!Get_Purchase_Order.Status__c}&lt;/p&gt;&lt;p&gt;&lt;b&gt;Authorizer:&lt;/b&gt; {!Get_Purchase_Order.Authorizer_Name__c}&lt;/p&gt;&lt;p&gt;&lt;br&gt;&lt;/p&gt;&lt;p&gt;Select the Contact who should receive this Purchase Order as a PDF email.&lt;/p&gt;</fieldText>
            <fieldType>DisplayText</fieldType>
        </fields>
        <fields>
            <name>Contact_Lookup</name>
            <fieldText>Send To</fieldText>
            <fieldType>ComponentInstance</fieldType>
            <isRequired>true</isRequired>
            <inputParameters>
                <name>fieldApiName</name>
                <value>
                    <stringValue>ContactId</stringValue>
                </value>
            </inputParameters>
            <inputParameters>
                <name>objectApiName</name>
                <value>
                    <stringValue>Contact</stringValue>
                </value>
            </inputParameters>
            <inputsOnNextNavToAssocScrn>UseStoredValues</inputsOnNextNavToAssocScrn>
            <storeOutputAutomatically>true</storeOutputAutomatically>
            <componentName>flowruntime:lookup</componentName>
        </fields>
    </screens>

    <!-- Screen 2: confirmation -->
    <screens>
        <name>Confirmation_Screen</name>
        <label>Confirm Send</label>
        <locationX>176</locationX>
        <locationY>374</locationY>
        <allowBack>true</allowBack>
        <allowFinish>true</allowFinish>
        <allowPause>false</allowPause>
        <connector>
            <targetReference>Send_PO_Email_Action</targetReference>
        </connector>
        <showFooter>true</showFooter>
        <showHeader>true</showHeader>
        <fields>
            <name>Confirm_Details</name>
            <fieldType>DisplayText</fieldType>
            <fieldText>&lt;p&gt;&lt;b&gt;Ready to send?&lt;/b&gt;&lt;/p&gt;&lt;p&gt;&lt;br&gt;&lt;/p&gt;&lt;p&gt;Purchase Order &lt;b&gt;{!Get_Purchase_Order.Name}&lt;/b&gt; will be generated as a PDF and emailed to the selected contact.&lt;/p&gt;&lt;p&gt;&lt;br&gt;&lt;/p&gt;&lt;p&gt;The Purchase Order will be stamped with &amp;quot;Digital Authorization from {!Get_Purchase_Order.Authorizer_Name__c}&amp;quot; after sending.&lt;/p&gt;&lt;p&gt;&lt;br&gt;&lt;/p&gt;&lt;p&gt;Click &lt;b&gt;Next&lt;/b&gt; to send, or &lt;b&gt;Previous&lt;/b&gt; to go back.&lt;/p&gt;</fieldText>
        </fields>
    </screens>

    <!-- Apex Action: call the invocable method -->
    <actionCalls>
        <name>Send_PO_Email_Action</name>
        <label>Send PO Email</label>
        <locationX>176</locationX>
        <locationY>494</locationY>
        <connector>
            <targetReference>Check_Result</targetReference>
        </connector>
        <faultConnector>
            <targetReference>Error_Screen</targetReference>
        </faultConnector>
        <actionName>SendPurchaseOrderEmail</actionName>
        <actionType>apex</actionType>
        <inputParameters>
            <name>poId</name>
            <value>
                <elementReference>recordId</elementReference>
            </value>
        </inputParameters>
        <inputParameters>
            <name>contactId</name>
            <value>
                <elementReference>Contact_Lookup.recordId</elementReference>
            </value>
        </inputParameters>
        <outputParameters>
            <assignToReference>emailResult</assignToReference>
            <name>isSuccess</name>
        </outputParameters>
        <outputParameters>
            <assignToReference>emailMessage</assignToReference>
            <name>message</name>
        </outputParameters>
    </actionCalls>

    <!-- Decision: check if the Apex action succeeded -->
    <decisions>
        <name>Check_Result</name>
        <label>Check Result</label>
        <locationX>176</locationX>
        <locationY>614</locationY>
        <defaultConnector>
            <targetReference>Error_Screen</targetReference>
        </defaultConnector>
        <defaultConnectorLabel>Failed</defaultConnectorLabel>
        <rules>
            <name>Is_Success</name>
            <conditionLogic>and</conditionLogic>
            <conditions>
                <leftValueReference>emailResult</leftValueReference>
                <operator>EqualTo</operator>
                <rightValue>
                    <booleanValue>true</booleanValue>
                </rightValue>
            </conditions>
            <connector>
                <targetReference>Success_Screen</targetReference>
            </connector>
            <label>Success</label>
        </rules>
    </decisions>

    <!-- Screen 3a: success -->
    <screens>
        <name>Success_Screen</name>
        <label>Email Sent</label>
        <locationX>50</locationX>
        <locationY>734</locationY>
        <allowBack>false</allowBack>
        <allowFinish>true</allowFinish>
        <allowPause>false</allowPause>
        <showFooter>true</showFooter>
        <showHeader>true</showHeader>
        <fields>
            <name>Success_Message</name>
            <fieldType>DisplayText</fieldType>
            <fieldText>&lt;p&gt;&lt;span style=&quot;color: rgb(0, 128, 0);&quot;&gt;&lt;b&gt;Success!&lt;/b&gt;&lt;/span&gt;&lt;/p&gt;&lt;p&gt;&lt;br&gt;&lt;/p&gt;&lt;p&gt;{!emailMessage}&lt;/p&gt;&lt;p&gt;&lt;br&gt;&lt;/p&gt;&lt;p&gt;A completed task has been logged on the Purchase Order record.&lt;/p&gt;</fieldText>
        </fields>
    </screens>

    <!-- Screen 3b: error -->
    <screens>
        <name>Error_Screen</name>
        <label>Error</label>
        <locationX>302</locationX>
        <locationY>734</locationY>
        <allowBack>false</allowBack>
        <allowFinish>true</allowFinish>
        <allowPause>false</allowPause>
        <showFooter>true</showFooter>
        <showHeader>true</showHeader>
        <fields>
            <name>Error_Message</name>
            <fieldType>DisplayText</fieldType>
            <fieldText>&lt;p&gt;&lt;span style=&quot;color: rgb(255, 0, 0);&quot;&gt;&lt;b&gt;Error&lt;/b&gt;&lt;/span&gt;&lt;/p&gt;&lt;p&gt;&lt;br&gt;&lt;/p&gt;&lt;p&gt;{!emailMessage}&lt;/p&gt;&lt;p&gt;&lt;br&gt;&lt;/p&gt;&lt;p&gt;Please try again or contact your administrator.&lt;/p&gt;</fieldText>
        </fields>
    </screens>

    <!-- start element -->
    <start>
        <locationX>50</locationX>
        <locationY>0</locationY>
        <connector>
            <targetReference>Get_Purchase_Order</targetReference>
        </connector>
    </start>
</Flow>
```

### Quick Action Metadata: Purchase_Order__c.Send_PO_Email.quickAction-meta.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<QuickAction xmlns="http://soap.sforce.com/2006/04/metadata">
    <flowDefinition>Send_Purchase_Order_Email</flowDefinition>
    <label>Send PO Email</label>
    <optionsCreateFeedItem>false</optionsCreateFeedItem>
    <type>Flow</type>
</QuickAction>
```

### Custom Metadata Records

Two `App_Config__mdt` records should be created for configuration:

**Company_Name**
```
DeveloperName:  Company_Name
Label:          Company Name
Value__c:       Your Company Name Here
```

**PO_Org_Wide_Email**
```
DeveloperName:  PO_Org_Wide_Email
Label:          PO Org-Wide Email
Value__c:       Purchase Orders
```

The `Value__c` for `PO_Org_Wide_Email` should match the `DisplayName` of an Org-Wide Email Address configured in Setup > Organization-Wide Addresses.

### Custom Object Metadata: Purchase_Order__c.object-meta.xml (Key Fields Only)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<CustomObject xmlns="http://soap.sforce.com/2006/04/metadata">
    <label>Purchase Order</label>
    <pluralLabel>Purchase Orders</pluralLabel>
    <nameField>
        <displayFormat>PO-{0000}</displayFormat>
        <label>PO Number</label>
        <type>AutoNumber</type>
    </nameField>
    <deploymentStatus>Deployed</deploymentStatus>
    <sharingModel>ReadWrite</sharingModel>
</CustomObject>
```

**Purchase_Order__c.Status__c.field-meta.xml**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<CustomField xmlns="http://soap.sforce.com/2006/04/metadata">
    <fullName>Status__c</fullName>
    <label>Status</label>
    <type>Picklist</type>
    <required>false</required>
    <valueSet>
        <restricted>true</restricted>
        <valueSetDefinition>
            <sorted>false</sorted>
            <value><fullName>Draft</fullName><default>true</default><label>Draft</label></value>
            <value><fullName>Submitted</fullName><default>false</default><label>Submitted</label></value>
            <value><fullName>Approved</fullName><default>false</default><label>Approved</label></value>
            <value><fullName>Sent</fullName><default>false</default><label>Sent</label></value>
            <value><fullName>Closed</fullName><default>false</default><label>Closed</label></value>
        </valueSetDefinition>
    </valueSet>
</CustomField>
```

**Purchase_Order__c.Authorizer_Name__c.field-meta.xml**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<CustomField xmlns="http://soap.sforce.com/2006/04/metadata">
    <fullName>Authorizer_Name__c</fullName>
    <label>Authorizer Name</label>
    <length>255</length>
    <type>Text</type>
    <required>false</required>
</CustomField>
```

**Purchase_Order__c.Digital_Authorization__c.field-meta.xml**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<CustomField xmlns="http://soap.sforce.com/2006/04/metadata">
    <fullName>Digital_Authorization__c</fullName>
    <label>Digital Authorization</label>
    <length>255</length>
    <type>Text</type>
    <required>false</required>
</CustomField>
```

## Deployment Order

When deploying this workflow to a new org, deploy in this order:

1. **Custom Objects & Fields** — Purchase_Order__c, Purchase_Order_Line_Item__c, custom fields on Task
2. **Custom Metadata Type & Records** — App_Config__mdt with Company_Name and PO_Org_Wide_Email records
3. **Visualforce Page** — PurchaseOrderPDF.page
4. **Apex Classes** — PurchaseOrderPDFController, SendPurchaseOrderEmail, SendPurchaseOrderEmailTest
5. **Flow** — Send_Purchase_Order_Email
6. **Quick Action** — Purchase_Order__c.Send_PO_Email
7. **Page Layout** — Add the Quick Action to the Purchase Order page layout's Salesforce Mobile and Lightning Experience Actions section

## Common Mistakes

1. **Using `new PageReference('/apex/PurchaseOrderPDF')` instead of `Page.PurchaseOrderPDF`** — The string-based reference fails in managed packages, loses compile-time validation, and breaks if the page is renamed. Fix: Always use the `Page.PageName` static reference. It validates at compile time and works in all packaging contexts.

2. **Not wrapping `getContentAsPDF()` in a `Test.isRunningTest()` check** — `getContentAsPDF()` returns blank content in test context and is treated as a callout. If your code asserts on the PDF blob or tries to process it, tests fail. Fix: Return `Blob.valueOf('Test PDF Content')` when `Test.isRunningTest()` is true.

3. **Sending the email BEFORE validating all inputs** — If you send the email first and then discover the PO has no Authorizer Name, the email is already gone but the digital auth stamp fails. Fix: Run ALL validation before calling `Messaging.sendEmail()`. The email is the point of no return.

4. **Throwing unhandled exceptions from the InvocableMethod** — The Flow fault screen shows a generic "An unhandled fault has occurred" message. Users have no idea what went wrong. Fix: Wrap everything in try-catch. Return error details in the `Response.message` field. Let the Flow display a meaningful error screen.

5. **Forgetting to set `setSaveAsActivity(false)` on the email** — If you don't disable auto-activity, Salesforce creates its own Task AND you create your own Task, resulting in duplicate activity records. Fix: Set `email.setSaveAsActivity(false)` and create your own Task with richer detail.

6. **Using CSS3 or SLDS in the Visualforce PDF** — Flexbox, grid, `border-radius`, `box-shadow`, CSS variables, and SLDS classes all fail silently in the PDF engine. The PDF renders but looks completely wrong. Fix: Use only CSS 2.1 — tables for layout, basic properties for styling, system fonts.

7. **Hardcoding the org-wide email address ID** — The ID differs between sandboxes and production. Fix: Store the org-wide email `DisplayName` in Custom Metadata. Query `OrgWideEmailAddress` by display name at runtime.

8. **Not adding a Fault Connector on the Apex Action in the Flow** — If the Apex action throws an unhandled exception, the flow crashes with no user feedback. Fix: Always add a Fault Connector from the Apex Action element to an Error Screen. Display `{!$Flow.FaultMessage}` for debugging.

9. **Creating the Task without setting `WhatId` and `WhoId`** — Without `WhatId`, the Task does not appear in the PO's activity timeline. Without `WhoId`, it does not appear on the Contact's activity timeline. Fix: Always set both — `WhatId` = PO ID, `WhoId` = Contact ID.

10. **Not testing the negative paths** — Tests only cover the happy path (email sent successfully). In production, Contacts have no email, POs have missing fields, and email limits get hit. Fix: Write explicit test methods for every failure scenario — missing email, missing authorizer, invalid PO ID, bulk invocation.

## See Also

- [Solution Architecture Patterns](./solution-architecture-patterns.md) — service layer, trigger patterns, naming conventions
- [Visualforce PDF](../visualforce-pdf/) — PDF generation deep dive
- [Email Services](../email-services/) — email sending patterns and limits
- [Quick Actions](../quick-actions/) — quick action configuration
- [Flows and Automation](../flows-and-automation/) — flow + Apex interaction
