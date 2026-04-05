# Visualforce PDF Generation

> **When to read this:** You need to render a Visualforce page as a PDF — either for direct download, email attachment, or saving as a file on a record.

## Rules

- When building a Visualforce page for PDF output, always set `applyBodyTag="false"` and `applyHtmlTag="false"` on `<apex:page>` because this gives you full control over the HTML structure and prevents Salesforce from injecting its own markup that breaks PDF layout.
- Always put CSS in `<style>` blocks inside the Visualforce page itself because external stylesheets and `<apex:stylesheet>` references are unreliable in PDF rendering — the Flying Saucer engine does not always resolve them.
- Never use JavaScript in a Visualforce page that renders as PDF because the PDF rendering engine does not execute JavaScript at all — any JS-dependent content will be missing.
- When generating a PDF blob in Apex, always use `PageReference.getContentAsPDF()` instead of `getContent()` because `getContent()` returns raw HTML while `getContentAsPDF()` returns a proper PDF blob regardless of whether the page has `renderAs="pdf"`.
- Always treat `getContentAsPDF()` as a callout in your code because it counts against the callout governor limit (100 per transaction) and cannot be called after DML in the same transaction without `@future` or queueable separation.
- When referencing images in a PDF Visualforce page, always use Static Resources with `$Resource` merge syntax because external URLs and absolute Salesforce URLs frequently fail to resolve in the PDF engine.
- Never call `getContentAsPDF()` from a trigger or from a method that runs after DML in the same transaction because the callout-after-DML restriction will throw a `System.CalloutException`.
- When testing code that calls `getContentAsPDF()`, always design your test so it only asserts that no exception is thrown and that the blob is non-null because the method returns a blank/minimal blob in test context — you cannot assert on actual PDF content.
- Always set the `id` parameter on the PageReference URL when generating a PDF for a specific record because the StandardController (or your custom controller) needs the record ID to query and display the correct data.
- When displaying currency or number fields in a PDF, always use `<apex:outputField>` or `<apex:outputText>` with format attributes because plain merge fields (`{!record.Amount}`) do not respect locale formatting.

## How It Works

### The Two Approaches to PDF Generation

**Approach 1: Direct Browser Rendering with `renderAs="pdf"`**
Add `renderAs="pdf"` to the `<apex:page>` tag. When a user navigates to the page URL (or clicks a button that opens it), the browser receives a PDF file instead of an HTML page. This is the simplest approach for "click a button, download a PDF" workflows.

**Approach 2: Programmatic Generation with `getContentAsPDF()`**
Create a `PageReference` pointing to your Visualforce page, then call `.getContentAsPDF()` to get a `Blob`. This is the approach you need when the PDF must be emailed, saved as a file, or processed further in Apex. The page does NOT need `renderAs="pdf"` for this to work — `getContentAsPDF()` forces PDF rendering regardless.

### The PDF Rendering Engine

Salesforce uses the **Flying Saucer** library (an open-source Java library) to render PDFs. Flying Saucer converts XHTML + CSS into PDF format. It supports CSS 2.1 (mostly) and some CSS3 properties like `@page`. It does NOT support JavaScript, CSS3 flexbox, CSS grid, or advanced selectors. Think of it as rendering your page in a very old browser that only understands basic HTML and CSS.

### StandardController vs Custom Controller

**StandardController** is the simplest option when you only need fields from a single record and its directly related objects. The controller auto-queries the record based on the `id` URL parameter. You access fields via merge syntax: `{!Purchase_Order__c.Name}`.

**Controller Extension** adds custom logic on top of the StandardController. Use this when you need to query related child records (like line items), perform calculations, or format data in ways merge fields can't handle.

**Custom Controller** gives you full control. Use this when the page aggregates data from multiple unrelated objects or when the StandardController's auto-query doesn't cover your needs.

For a Purchase Order PDF with line items, a **StandardController + Extension** is the right choice — the StandardController handles the PO header fields, and the extension queries and exposes the line items.

## Code Examples

### Complete Visualforce Page for Purchase Order PDF

```xml
<apex:page standardController="Purchase_Order__c"
           extensions="PurchaseOrderPDFExtension"
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
        /* @page controls the PDF page itself — size, margins, headers, footers */
        @page {
            size: letter;
            margin: 0.75in 0.5in 1in 0.5in;

            /* repeating footer on every page */
            @bottom-center {
                content: element(footer);
            }
        }

        /* general reset and base styles */
        body {
            font-family: Arial, Helvetica, sans-serif;
            font-size: 10pt;
            color: #333333;
            margin: 0;
            padding: 0;
        }

        /* header section at top of first page */
        .company-header {
            text-align: center;
            border-bottom: 2px solid #003366;
            padding-bottom: 10px;
            margin-bottom: 15px;
        }

        .company-header h1 {
            font-size: 18pt;
            color: #003366;
            margin: 0;
        }

        .company-header .subtitle {
            font-size: 10pt;
            color: #666666;
        }

        /* two-column layout for PO header info */
        .header-table {
            width: 100%;
            border-collapse: collapse;
            margin-bottom: 15px;
        }

        .header-table td {
            padding: 3px 8px;
            vertical-align: top;
            font-size: 9pt;
        }

        .header-table .label {
            font-weight: bold;
            color: #003366;
            width: 130px;
        }

        /* line items table */
        .line-items {
            width: 100%;
            border-collapse: collapse;
            margin-top: 15px;
        }

        .line-items th {
            background-color: #003366;
            color: #FFFFFF;
            font-size: 9pt;
            padding: 6px 8px;
            text-align: left;
            border: 1px solid #003366;
        }

        .line-items td {
            font-size: 9pt;
            padding: 5px 8px;
            border: 1px solid #CCCCCC;
        }

        .line-items tr:nth-child(even) td {
            background-color: #F5F5F5;
        }

        .line-items .currency {
            text-align: right;
        }

        .line-items .qty {
            text-align: center;
        }

        /* totals section */
        .totals-table {
            width: 250px;
            margin-left: auto;
            margin-top: 10px;
            border-collapse: collapse;
        }

        .totals-table td {
            padding: 4px 8px;
            font-size: 9pt;
        }

        .totals-table .total-label {
            font-weight: bold;
            text-align: right;
        }

        .totals-table .total-value {
            text-align: right;
            width: 100px;
        }

        .totals-table .grand-total td {
            border-top: 2px solid #003366;
            font-size: 11pt;
            font-weight: bold;
            color: #003366;
        }

        /* signature and authorization section */
        .auth-section {
            margin-top: 30px;
            padding-top: 15px;
            border-top: 1px solid #CCCCCC;
        }

        .signature-line {
            border-bottom: 1px solid #333333;
            width: 250px;
            display: inline-block;
            margin-top: 30px;
        }

        /* footer that repeats on every page */
        .footer {
            position: running(footer);
            font-size: 7pt;
            color: #999999;
            text-align: center;
            border-top: 1px solid #CCCCCC;
            padding-top: 5px;
        }

        /* force page break before a section */
        .page-break {
            page-break-before: always;
        }

        /* prevent a row or block from splitting across pages */
        .no-break {
            page-break-inside: avoid;
        }
    </style>
</head>
<body>

    <!-- footer element — pulled into @bottom-center by CSS -->
    <div class="footer">
        Purchase Order {!Purchase_Order__c.Name} |
        Generated <apex:outputText value="{0,date,MM/dd/yyyy}">
            <apex:param value="{!NOW()}" />
        </apex:outputText> |
        Page <span class="pagenumber"/> of <span class="pagecount"/>
    </div>

    <!-- company header -->
    <div class="company-header">
        <!-- use a static resource for the logo -->
        <apex:image url="{!URLFOR($Resource.CompanyLogo)}" width="150" />
        <h1>Purchase Order</h1>
        <div class="subtitle">{!Purchase_Order__c.Name}</div>
    </div>

    <!-- PO header fields in a two-column layout -->
    <table class="header-table">
        <tr>
            <td class="label">PO Number:</td>
            <td>{!Purchase_Order__c.Name}</td>
            <td class="label">PO Date:</td>
            <td>
                <apex:outputText value="{0,date,MM/dd/yyyy}">
                    <apex:param value="{!Purchase_Order__c.PO_Date__c}" />
                </apex:outputText>
            </td>
        </tr>
        <tr>
            <td class="label">Vendor:</td>
            <td>{!Purchase_Order__c.Vendor__r.Name}</td>
            <td class="label">Status:</td>
            <td>{!Purchase_Order__c.Status__c}</td>
        </tr>
        <tr>
            <td class="label">Authorizer:</td>
            <td>{!Purchase_Order__c.Authorizer_Name__c}</td>
            <td class="label">Ship To:</td>
            <td>{!Purchase_Order__c.Shipping_Address__c}</td>
        </tr>
        <tr>
            <td class="label">Payment Terms:</td>
            <td>{!Purchase_Order__c.Payment_Terms__c}</td>
            <td class="label">Delivery Date:</td>
            <td>
                <apex:outputText value="{0,date,MM/dd/yyyy}">
                    <apex:param value="{!Purchase_Order__c.Expected_Delivery__c}" />
                </apex:outputText>
            </td>
        </tr>
    </table>

    <!-- special instructions -->
    <apex:outputPanel rendered="{!NOT(ISBLANK(Purchase_Order__c.Special_Instructions__c))}">
        <div class="no-break" style="margin-top: 10px; padding: 8px; background-color: #FFF9E6; border: 1px solid #E6D9A6;">
            <strong>Special Instructions:</strong><br/>
            <apex:outputField value="{!Purchase_Order__c.Special_Instructions__c}" />
        </div>
    </apex:outputPanel>

    <!-- line items table -->
    <table class="line-items">
        <thead>
            <tr>
                <th style="width: 40px;">#</th>
                <th>Item Description</th>
                <th style="width: 60px;" class="qty">Qty</th>
                <th style="width: 90px;" class="currency">Unit Price</th>
                <th style="width: 90px;" class="currency">Total</th>
            </tr>
        </thead>
        <tbody>
            <apex:repeat value="{!lineItems}" var="item">
                <tr class="no-break">
                    <td>{!item.Line_Number__c}</td>
                    <td>
                        {!item.Product_Name__c}
                        <apex:outputPanel rendered="{!NOT(ISBLANK(item.Description__c))}">
                            <br/>
                            <span style="color: #666666; font-size: 8pt;">{!item.Description__c}</span>
                        </apex:outputPanel>
                    </td>
                    <td class="qty">{!item.Quantity__c}</td>
                    <td class="currency">
                        <apex:outputText value="{0, number, $#,##0.00}">
                            <apex:param value="{!item.Unit_Price__c}" />
                        </apex:outputText>
                    </td>
                    <td class="currency">
                        <apex:outputText value="{0, number, $#,##0.00}">
                            <apex:param value="{!item.Line_Total__c}" />
                        </apex:outputText>
                    </td>
                </tr>
            </apex:repeat>
        </tbody>
    </table>

    <!-- totals -->
    <table class="totals-table">
        <tr>
            <td class="total-label">Subtotal:</td>
            <td class="total-value">
                <apex:outputText value="{0, number, $#,##0.00}">
                    <apex:param value="{!Purchase_Order__c.Subtotal__c}" />
                </apex:outputText>
            </td>
        </tr>
        <tr>
            <td class="total-label">Tax:</td>
            <td class="total-value">
                <apex:outputText value="{0, number, $#,##0.00}">
                    <apex:param value="{!Purchase_Order__c.Tax_Amount__c}" />
                </apex:outputText>
            </td>
        </tr>
        <tr>
            <td class="total-label">Shipping:</td>
            <td class="total-value">
                <apex:outputText value="{0, number, $#,##0.00}">
                    <apex:param value="{!Purchase_Order__c.Shipping_Cost__c}" />
                </apex:outputText>
            </td>
        </tr>
        <tr class="grand-total">
            <td class="total-label">Total:</td>
            <td class="total-value">
                <apex:outputText value="{0, number, $#,##0.00}">
                    <apex:param value="{!Purchase_Order__c.Grand_Total__c}" />
                </apex:outputText>
            </td>
        </tr>
    </table>

    <!-- authorization / signature -->
    <div class="auth-section no-break">
        <p><strong>Authorization:</strong></p>
        <p>
            Authorized by: {!Purchase_Order__c.Authorizer_Name__c}<br/>
            <span class="signature-line">&nbsp;</span><br/>
            <span style="font-size: 8pt; color: #666666;">Signature</span>
        </p>
        <apex:outputPanel rendered="{!NOT(ISBLANK(Purchase_Order__c.Digital_Authorization__c))}">
            <p style="color: #003366; font-weight: bold; margin-top: 10px;">
                {!Purchase_Order__c.Digital_Authorization__c}
            </p>
        </apex:outputPanel>
    </div>

</body>
</html>
</apex:page>
```

### Controller Extension for Line Items

```apex
public with sharing class PurchaseOrderPDFExtension {
    // the Purchase Order record from the StandardController
    private Purchase_Order__c po;

    // line items to display in the PDF table
    public List<PO_Line_Item__c> lineItems { get; private set; }

    // constructor — receives the StandardController instance
    // @param stdController: the auto-injected StandardController for Purchase_Order__c
    public PurchaseOrderPDFExtension(ApexPages.StandardController stdController) {
        // grab the record — at this point only Id is loaded by default
        this.po = (Purchase_Order__c) stdController.getRecord();

        System.debug('PurchaseOrderPDFExtension init for PO Id: ' + this.po.Id);

        // query line items for this purchase order, ordered by line number
        this.lineItems = [
            SELECT Id,
                   Line_Number__c,
                   Product_Name__c,
                   Description__c,
                   Quantity__c,
                   Unit_Price__c,
                   Line_Total__c
            FROM PO_Line_Item__c
            WHERE Purchase_Order__c = :this.po.Id
            ORDER BY Line_Number__c ASC
        ];

        System.debug('Loaded ' + this.lineItems.size() + ' line items');
    }
}
```

### Generating the PDF Blob in Apex (for Email or File Attachment)

```apex
public with sharing class PurchaseOrderPDFService {

    // generate a PDF blob for the given Purchase Order
    // @param poId: the Id of the Purchase_Order__c record
    // @return Blob: the rendered PDF content
    public static Blob generatePDF(Id poId) {
        System.debug('Generating PDF for PO: ' + poId);

        // build the page reference — point to the Visualforce page by name
        PageReference pdfPage = Page.PurchaseOrderPDF;

        // pass the record id so the StandardController picks it up
        pdfPage.getParameters().put('id', poId);

        // getContentAsPDF() renders the page as PDF and returns the blob
        // this counts as a callout — cannot be called after DML in the same txn
        Blob pdfBlob;
        if (Test.isRunningTest()) {
            // getContentAsPDF() returns a blank blob in test context
            // use a dummy blob so the rest of the code can be tested
            pdfBlob = Blob.valueOf('Test PDF Content');
        } else {
            pdfBlob = pdfPage.getContentAsPDF();
        }

        System.debug('PDF blob generated, size: ' + pdfBlob.size() + ' bytes');
        return pdfBlob;
    }

    // save the PDF as a Salesforce File (ContentVersion) linked to the PO record
    // @param poId: the Id of the Purchase_Order__c to link the file to
    // @param pdfBlob: the PDF blob content
    // @param fileName: the file name including .pdf extension
    // @return Id: the ContentDocumentLink Id
    public static Id savePDFAsFile(Id poId, Blob pdfBlob, String fileName) {
        System.debug('Saving PDF as file: ' + fileName + ' on record: ' + poId);

        // step 1 — create the ContentVersion (the actual file)
        ContentVersion cv = new ContentVersion();
        cv.Title = fileName.removeEnd('.pdf');
        cv.PathOnClient = fileName;
        cv.VersionData = pdfBlob;
        // Origin = 'C' means Content (uploaded by user, vs 'H' for Chatter)
        cv.Origin = 'C';
        insert cv;

        System.debug('ContentVersion created: ' + cv.Id);

        // step 2 — query back to get the ContentDocumentId
        // ContentVersion auto-creates a ContentDocument parent on insert
        ContentVersion insertedCV = [
            SELECT ContentDocumentId
            FROM ContentVersion
            WHERE Id = :cv.Id
            LIMIT 1
        ];

        // step 3 — create ContentDocumentLink to attach file to the PO record
        ContentDocumentLink cdl = new ContentDocumentLink();
        cdl.ContentDocumentId = insertedCV.ContentDocumentId;
        cdl.LinkedEntityId = poId;
        // 'V' = Viewer, 'C' = Collaborator, 'I' = Inferred
        cdl.ShareType = 'V';
        // 'P' = all users with permission, 'R' = record-level, 'N' = not shared
        cdl.Visibility = 'AllUsers';
        insert cdl;

        System.debug('ContentDocumentLink created: ' + cdl.Id);
        return cdl.Id;
    }
}
```

### Test Class

```apex
@isTest
private class PurchaseOrderPDFServiceTest {

    @TestSetup
    static void makeData() {
        // create a purchase order with line items for bulk testing
        Purchase_Order__c po = new Purchase_Order__c(
            PO_Date__c = Date.today(),
            Status__c = 'Draft',
            Authorizer_Name__c = 'Jane Smith',
            Payment_Terms__c = 'Net 30'
        );
        insert po;

        List<PO_Line_Item__c> items = new List<PO_Line_Item__c>();
        for (Integer i = 1; i <= 5; i++) {
            items.add(new PO_Line_Item__c(
                Purchase_Order__c = po.Id,
                Line_Number__c = i,
                Product_Name__c = 'Widget ' + i,
                Quantity__c = i * 10,
                Unit_Price__c = 25.00
            ));
        }
        insert items;
    }

    @isTest
    static void testGeneratePDF() {
        Purchase_Order__c po = [SELECT Id FROM Purchase_Order__c LIMIT 1];

        Test.startTest();
        // getContentAsPDF() returns a dummy blob in test context
        // we can only verify it doesn't throw an exception and returns non-null
        Blob pdfBlob = PurchaseOrderPDFService.generatePDF(po.Id);
        Test.stopTest();

        System.assertNotEquals(null, pdfBlob, 'PDF blob should not be null');
        System.assert(pdfBlob.size() > 0, 'PDF blob should not be empty');
    }

    @isTest
    static void testSavePDFAsFile() {
        Purchase_Order__c po = [SELECT Id, Name FROM Purchase_Order__c LIMIT 1];
        Blob testBlob = Blob.valueOf('Test PDF Content');

        Test.startTest();
        Id cdlId = PurchaseOrderPDFService.savePDFAsFile(
            po.Id,
            testBlob,
            po.Name + '.pdf'
        );
        Test.stopTest();

        System.assertNotEquals(null, cdlId, 'ContentDocumentLink Id should not be null');

        // verify the file is linked to the PO record
        List<ContentDocumentLink> links = [
            SELECT Id, ContentDocumentId, LinkedEntityId
            FROM ContentDocumentLink
            WHERE LinkedEntityId = :po.Id
        ];
        System.assertEquals(1, links.size(), 'Should have exactly one file linked to the PO');
    }

    @isTest
    static void testControllerExtension() {
        Purchase_Order__c po = [SELECT Id FROM Purchase_Order__c LIMIT 1];

        // set up the StandardController as if the page loaded with this record
        ApexPages.StandardController stdCtrl = new ApexPages.StandardController(po);

        Test.startTest();
        PurchaseOrderPDFExtension ext = new PurchaseOrderPDFExtension(stdCtrl);
        Test.stopTest();

        // verify line items were queried
        System.assertEquals(5, ext.lineItems.size(), 'Should have loaded 5 line items');
    }
}
```

### Passing URL Parameters for Custom Controllers

```apex
// when using a custom controller instead of StandardController,
// grab the record Id from the URL parameters
public class PurchaseOrderPDFController {
    public Purchase_Order__c po { get; private set; }
    public List<PO_Line_Item__c> lineItems { get; private set; }

    public PurchaseOrderPDFController() {
        // read the id from the URL: /apex/PurchaseOrderPDF?id=a00XXXXXXXXXXXX
        String poId = ApexPages.currentPage().getParameters().get('id');

        System.debug('Custom controller loading PO: ' + poId);

        if (String.isNotBlank(poId)) {
            // query all fields you need — unlike StandardController,
            // a custom controller does NOT auto-query based on page merge fields
            this.po = [
                SELECT Id, Name, PO_Date__c, Status__c, Vendor__r.Name,
                       Authorizer_Name__c, Payment_Terms__c, Shipping_Address__c,
                       Expected_Delivery__c, Special_Instructions__c,
                       Subtotal__c, Tax_Amount__c, Shipping_Cost__c, Grand_Total__c,
                       Digital_Authorization__c
                FROM Purchase_Order__c
                WHERE Id = :poId
                LIMIT 1
            ];

            this.lineItems = [
                SELECT Id, Line_Number__c, Product_Name__c, Description__c,
                       Quantity__c, Unit_Price__c, Line_Total__c
                FROM PO_Line_Item__c
                WHERE Purchase_Order__c = :poId
                ORDER BY Line_Number__c ASC
            ];
        }
    }
}
```

## Common Mistakes

1. **Using external CSS files or `<apex:stylesheet>` in PDF pages** -- The Flying Saucer PDF engine does not reliably load external stylesheets. Large external CSS files (thousands of lines) are especially problematic. Fix: put all CSS in a `<style>` block inside the Visualforce page. Keep it minimal — only include styles you actually use.

2. **Calling `getContentAsPDF()` after DML in the same transaction** -- `getContentAsPDF()` counts as a callout. Salesforce prohibits callouts after DML in the same execution context. You'll get `System.CalloutException: You have uncommitted work pending`. Fix: call `getContentAsPDF()` BEFORE any DML, or separate the PDF generation and DML into different execution contexts (queueable, future, or flow orchestration).

3. **Asserting on PDF content in test classes** -- `getContentAsPDF()` returns a blank or minimal blob in test context. Tests that try to parse or validate PDF content will always fail. Fix: use `Test.isRunningTest()` to substitute a dummy blob in your service class, and limit test assertions to "blob is non-null" and "no exception was thrown." Test the controller extension separately to verify data queries work.

4. **Forgetting to pass the record `id` parameter on the PageReference** -- If you create `Page.PurchaseOrderPDF` without setting the `id` parameter, the StandardController has no record to load and all merge fields render blank. Fix: always call `pdfPage.getParameters().put('id', recordId)` before calling `getContentAsPDF()`.

5. **Using `<img src>` with absolute Salesforce URLs or external URLs** -- The PDF engine makes its own HTTP request to fetch images. Absolute org URLs may fail due to authentication. External URLs may fail due to CSP or network restrictions. Fix: upload images as Static Resources and reference them with `{!URLFOR($Resource.ImageName)}`. For dynamic images stored in Salesforce (like a company logo on an Account), convert to base64 and use a `data:` URI — but be aware that very large base64 images may render as grey thumbnails.

6. **Not setting `applyBodyTag="false"` and `applyHtmlTag="false"`** -- Without these, Salesforce wraps your content in its own `<html>` and `<body>` tags with default styles. This causes unpredictable margins, font sizes, and layout in the PDF. Fix: always set both to `false` and write your own complete HTML structure.

7. **Expecting CSS flexbox, grid, or modern selectors to work** -- The Flying Saucer engine supports CSS 2.1 with some CSS3 additions (`@page`, `page-break-*`). Flexbox, grid, `calc()`, CSS variables, and advanced pseudo-selectors do not work. Fix: use `<table>` for layout (yes, really — tables are the most reliable layout tool for PDF rendering). Use `float` for simpler side-by-side layouts.

8. **Page breaks splitting table rows or critical content blocks** -- A table row or a signature block gets cut across two pages, making the PDF look broken. Fix: use `page-break-inside: avoid` on elements that should not be split. Use `page-break-before: always` on sections that should start on a new page. Wrap table rows in `<div class="no-break">` if needed (though this can conflict with `<table>` structure — test carefully).

9. **Running `getContentAsPDF()` in Batch Apex without the `Database.AllowsCallouts` interface** -- Since `getContentAsPDF()` is a callout, Batch Apex classes that call it must implement `Database.AllowsCallouts`. Without it, you get a runtime error. Fix: add `implements Database.Batchable<sObject>, Database.AllowsCallouts` to your batch class declaration. Be aware of the 100-callout-per-transaction limit — set your batch size accordingly.

10. **Mismatched API versions between the Visualforce page and the Apex controller** -- If the VF page is on API version 30.0 and the controller is on 55.0, field references and behavior can be inconsistent, sometimes producing blank PDFs. Fix: keep the Visualforce page, controller, and extension all on the same API version. Use the latest stable version (66.0+).

## See Also

- [Email with Attachments](../email-services/email-with-attachments.md) — sending the generated PDF blob as an email attachment
