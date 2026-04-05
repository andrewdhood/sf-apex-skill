# Visualforce & PDF Deployment Considerations

> **When to read this:** Before deploying a Visualforce PDF solution (pages, controllers, static resources, and the Apex service class that calls `getContentAsPDF()`) to any Salesforce org, or when troubleshooting deployment failures involving VF page references.

## Rules

- When deploying a VF PDF solution, always deploy in this exact order: Static Resources first, then Apex controllers/extensions, then Visualforce pages, then Apex service classes that call `Page.MyPage` or `getContentAsPDF()` -- because Salesforce validates all compile-time references at deploy time and rejects any reference to a component that does not yet exist in the target org.
- Never deploy an Apex class that contains `Page.PurchaseOrderPDF` (a compile-time page reference) before the Visualforce page `PurchaseOrderPDF` exists in the target org, because the compiler resolves `Page.*` references at save/deploy time and throws `Variable does not exist: Page.PurchaseOrderPDF` if the page is missing.
- When deploying the entire solution in a single `--source-dir force-app` deployment, Salesforce resolves internal dependency ordering automatically -- always prefer a single full deployment over incremental wave deploys when possible, because it eliminates ordering headaches.
- When incremental deployment is unavoidable (change sets, partial package.xml, CI/CD pipelines that deploy modules separately), always split into explicit waves and deploy them sequentially, because Salesforce does not guarantee cross-type dependency resolution within a single change set.
- Always pin the Visualforce page and its Apex controller/extension to the same API version, because mismatched versions cause inconsistent behavior -- the VF page renders under its own API version's rules while the controller executes under a different version's rules, which can produce blank fields, broken merge syntax, or changed CSP behavior.
- Never deploy a Visualforce page that references a Static Resource (`$Resource.CompanyLogo`) before that Static Resource exists in the target org, because the VF page deployment validates `$Resource` merge field references and fails if the resource is missing.
- When granting Visualforce page access for PDF generation via Apex, always grant access on the running user's profile (or a permission set assigned to them), because `getContentAsPDF()` enforces the running user's Visualforce page access even when the Apex class uses `without sharing` -- the sharing keyword only affects record-level access, not page-level access.
- Always include test classes in the same deployment as the Apex classes they test, because production deployments require 75% coverage from the tests that run during that deployment -- if the test class is missing, coverage drops.
- When testing code that calls `getContentAsPDF()`, always use `Test.isRunningTest()` to substitute a dummy blob, because `getContentAsPDF()` returns a blank/minimal blob in test context and cannot render actual PDF content during tests.
- Never reference a Visualforce page by string URL (`'/apex/PurchaseOrderPDF'`) when you can use the compile-time `Page.PurchaseOrderPDF` syntax, because the compile-time reference creates an explicit dependency that the deployment system can resolve, while a string reference is invisible to the compiler and silently breaks at runtime if the page is missing.

## How It Works

### The VF PDF Dependency Chain

A Visualforce PDF solution has a strict dependency chain. Each layer depends on the layers below it:

```
Layer 4:  Apex Service Classes (PurchaseOrderPDFService)
            calls Page.PurchaseOrderPDF.getContentAsPDF()
            ↓ depends on
Layer 3:  Visualforce Pages (PurchaseOrderPDF.page)
            uses controller="PurchaseOrderPDFExtension"
            uses {!URLFOR($Resource.CompanyLogo)}
            ↓ depends on
Layer 2:  Apex Controllers / Extensions (PurchaseOrderPDFExtension)
            queries Purchase_Order__c, PO_Line_Item__c
            ↓ depends on
Layer 1:  Custom Objects, Fields, Static Resources
            Purchase_Order__c, PO_Line_Item__c, CompanyLogo
```

**Why Layer 4 must deploy after Layer 3:** The Apex compiler resolves `Page.PurchaseOrderPDF` at compile time. If the page does not exist in the target org when the class is compiled, deployment fails with `Variable does not exist: Page.PurchaseOrderPDF`. This is different from most Apex references -- custom object and field references are resolved more flexibly, but `Page.*` references are strict.

**Why Layer 3 must deploy after Layers 1 and 2:** The Visualforce page declares its controller or extension in the `<apex:page>` tag. If the controller class does not exist, the page deployment fails with `Apex class 'PurchaseOrderPDFExtension' does not exist`. Similarly, `$Resource.CompanyLogo` references are validated at save time.

**The circular dependency trap:** The VF page references its controller (Layer 3 depends on Layer 2), and the service class references the VF page (Layer 4 depends on Layer 3). But the controller does NOT reference the page -- it is referenced BY the page. So the dependency is one-directional at each layer, and no true circular dependency exists. The confusion arises when developers put `Page.MyPage` references inside the controller itself (for redirects or PDF generation). Keep PDF generation logic in a separate service class (Layer 4), not in the controller (Layer 2).

### Single Deploy vs. Wave Deploy

**Single deploy (recommended):** When you deploy the entire `force-app` directory at once, Salesforce's Metadata API resolves dependency ordering internally. It compiles Apex classes, validates VF pages, and links references in the correct order. This is the simplest and most reliable approach.

```bash
# one command, everything deploys together, dependencies resolved automatically
sf project deploy start --source-dir force-app --target-org my-sandbox
```

**Wave deploy (when incremental is required):** When you cannot deploy everything at once -- for example, when using change sets, deploying only changed files in CI/CD, or when different teams own different layers -- you must deploy in waves.

```bash
# wave 1: foundation (objects, fields, static resources)
sf project deploy start \
    --source-dir force-app/main/default/objects \
    --source-dir force-app/main/default/staticresources \
    --target-org my-sandbox \
    --test-level NoTestRun

# wave 2: apex controllers and extensions (no Page.* references)
sf project deploy start \
    --source-dir force-app/main/default/classes/PurchaseOrderPDFExtension.cls \
    --source-dir force-app/main/default/classes/PurchaseOrderPDFController.cls \
    --target-org my-sandbox \
    --test-level NoTestRun

# wave 3: visualforce pages and components
sf project deploy start \
    --source-dir force-app/main/default/pages \
    --source-dir force-app/main/default/components \
    --target-org my-sandbox \
    --test-level NoTestRun

# wave 4: service classes that reference Page.* and test classes
sf project deploy start \
    --source-dir force-app/main/default/classes/PurchaseOrderPDFService.cls \
    --source-dir force-app/main/default/classes/PurchaseOrderPDFServiceTest.cls \
    --source-dir force-app/main/default/classes/PurchaseOrderEmailService.cls \
    --source-dir force-app/main/default/classes/PurchaseOrderEmailServiceTest.cls \
    --target-org my-sandbox \
    --test-level NoTestRun
```

### VF Page Metadata Structure

Every Visualforce page consists of two files that must be deployed together:

**`PurchaseOrderPDF.page`** -- the actual markup (HTML, CSS, Visualforce tags).

**`PurchaseOrderPDF.page-meta.xml`** -- the metadata descriptor.

The meta.xml controls how the page behaves and who can access it.

### API Version Pinning

Visualforce pages have their own API version, separate from the Apex controller's API version. Each component executes under its own version's rules. This matters because Salesforce changes behavior between API versions:

- **API 30.0+**: `confirmationTokenRequired` was introduced (CSRF protection).
- **API 40.0+**: Content Security Policy (CSP) headers changed, affecting how external scripts and styles load.
- **API 44.0+**: Lightning stylesheets became available for VF pages (`lightningStylesheets="true"`).
- **API 50.0+**: Stricter CSP enforcement for certain resource types.

When the VF page is on API 30.0 but the controller is on API 62.0, the page renders under API 30.0 rules (older CSP, no Lightning stylesheets) while the controller runs under API 62.0 rules. This mismatch can cause blank fields, styling inconsistencies, or security header differences. Always keep them aligned.

To update the API version, you must redeploy both the `.page-meta.xml` (which declares the version) and the `.page` file (they deploy as a pair).

### Test Strategy for getContentAsPDF()

`getContentAsPDF()` behaves differently in test context:
- It returns a non-null `Blob`, but the content is blank or minimal -- not an actual PDF.
- You cannot assert on PDF content (page count, text, layout).
- It still counts as a callout in terms of governor limits.
- It may throw exceptions if the page or controller has compile errors, missing fields, or permission issues.

The test strategy is to separate concerns:

1. **Test the controller/extension independently** -- verify queries, data formatting, and line item loading using `ApexPages.StandardController` directly.
2. **Test the PDF service class** -- verify that `getContentAsPDF()` does not throw an exception and returns a non-null blob. Use `Test.isRunningTest()` to substitute a dummy blob for downstream logic.
3. **Test the email send independently** -- pass a dummy blob to the email service and verify the email is queued.
4. **Test the end-to-end flow** -- verify the orchestration (generate, email, stamp) works without testing actual PDF content.

## Code Examples

### Complete .page-meta.xml for a PDF Page

```xml
<?xml version="1.0" encoding="UTF-8"?>
<ApexPage xmlns="http://soap.sforce.com/2006/04/metadata">
    <apiVersion>62.0</apiVersion>
    <availableInTouch>false</availableInTouch>
    <confirmationTokenRequired>false</confirmationTokenRequired>
    <description>Renders a Purchase Order as a PDF document</description>
    <label>Purchase Order PDF</label>
</ApexPage>
```

| Field | Type | Purpose |
|-------|------|---------|
| `apiVersion` | decimal | The Salesforce API version the page runs under. Controls behavior, CSP headers, and available features. Always match the controller's API version. |
| `availableInTouch` | boolean | Whether the page is accessible in the Salesforce mobile app. Set to `false` for PDF-only pages -- they render as PDF, which mobile cannot display inline. |
| `confirmationTokenRequired` | boolean | Enables CSRF protection. Set to `false` for pages that are only rendered server-side via `getContentAsPDF()`. Set to `true` for pages accessed by users via browser. |
| `description` | string | Human-readable description shown in Setup. |
| `label` | string | Display name shown in Setup > Visualforce Pages. |

### Complete .component-meta.xml for a VF Component

```xml
<?xml version="1.0" encoding="UTF-8"?>
<ApexComponent xmlns="http://soap.sforce.com/2006/04/metadata">
    <apiVersion>62.0</apiVersion>
    <description>Reusable company header for PDF documents</description>
    <label>PDF Company Header</label>
</ApexComponent>
```

### Static Resource for PDF Images

**Upload the Static Resource:**

A Static Resource for a logo image has this meta.xml:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<StaticResource xmlns="http://soap.sforce.com/2006/04/metadata">
    <cacheControl>Public</cacheControl>
    <contentType>image/png</contentType>
    <description>Company logo for PDF documents</description>
</StaticResource>
```

The file pair is:
- `force-app/main/default/staticresources/CompanyLogo.resource` -- the actual image file (renamed to `.resource`)
- `force-app/main/default/staticresources/CompanyLogo.resource-meta.xml` -- the metadata above

**Reference in Visualforce:**

```xml
<!-- single file static resource -->
<apex:image url="{!URLFOR($Resource.CompanyLogo)}" width="150" />

<!-- zipped static resource with multiple images -->
<apex:image url="{!URLFOR($Resource.PDFAssets, 'images/logo.png')}" width="150" />
<apex:image url="{!URLFOR($Resource.PDFAssets, 'images/signature-bg.png')}" width="200" />
```

**Critical for PDF rendering:** The `$Resource` merge field resolves to a server-relative URL, which the Flying Saucer PDF engine can fetch internally. External URLs (`https://example.com/logo.png`) and absolute Salesforce URLs (`https://myorg.my.salesforce.com/resource/...`) frequently fail in the PDF engine because it makes its own HTTP request and may not have the correct authentication or network access. Always use Static Resources with `$Resource` merge fields for images in PDF pages.

### Complete package.xml for a VF PDF Solution

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Package xmlns="http://soap.sforce.com/2006/04/metadata">
    <version>62.0</version>

    <!-- layer 1: custom objects and fields -->
    <types>
        <members>Purchase_Order__c</members>
        <members>PO_Line_Item__c</members>
        <name>CustomObject</name>
    </types>

    <!-- layer 1: static resources (logos, images for the PDF) -->
    <types>
        <members>CompanyLogo</members>
        <name>StaticResource</name>
    </types>

    <!-- layers 2 and 4: all Apex classes (controllers, services, tests) -->
    <types>
        <members>PurchaseOrderPDFExtension</members>
        <members>PurchaseOrderPDFController</members>
        <members>PurchaseOrderPDFService</members>
        <members>PurchaseOrderEmailService</members>
        <members>PurchaseOrderAuthorizationService</members>
        <members>PurchaseOrderPDFServiceTest</members>
        <members>PurchaseOrderEmailServiceTest</members>
        <name>ApexClass</name>
    </types>

    <!-- layer 3: visualforce pages -->
    <types>
        <members>PurchaseOrderPDF</members>
        <name>ApexPage</name>
    </types>

    <!-- layer 3: visualforce components (if any) -->
    <types>
        <members>PDFCompanyHeader</members>
        <name>ApexComponent</name>
    </types>

    <!-- profile access for VF pages -->
    <types>
        <members>Admin</members>
        <members>Standard</members>
        <name>Profile</name>
    </types>
</Package>
```

**When deploying with this package.xml**, the Metadata API resolves dependencies automatically across the types listed. You do not need to split it into waves -- the API handles the ordering. Wave deployment is only necessary when deploying subsets (e.g., deploying Apex without the VF pages, or deploying via change sets).

### Wave Deployment Script for CI/CD

```bash
#!/bin/bash
# deploy-vf-pdf-waves.sh
# deploy a VF PDF solution in dependency order when incremental deploy is required
# usage: ./deploy-vf-pdf-waves.sh <org-alias>

set -euo pipefail

ORG="${1:?Usage: ./deploy-vf-pdf-waves.sh <org-alias>}"

echo "// wave 1: deploying static resources"
sf project deploy start \
    --source-dir force-app/main/default/staticresources \
    --target-org "${ORG}" \
    --test-level NoTestRun \
    --wait 10

echo "// wave 2: deploying Apex controllers and extensions"
sf project deploy start \
    --source-dir force-app/main/default/classes/PurchaseOrderPDFExtension.cls \
    --source-dir force-app/main/default/classes/PurchaseOrderPDFController.cls \
    --target-org "${ORG}" \
    --test-level NoTestRun \
    --wait 10

echo "// wave 3: deploying Visualforce pages and components"
sf project deploy start \
    --source-dir force-app/main/default/pages \
    --source-dir force-app/main/default/components \
    --target-org "${ORG}" \
    --test-level NoTestRun \
    --wait 10

echo "// wave 4: deploying service classes and test classes"
sf project deploy start \
    --source-dir force-app/main/default/classes/PurchaseOrderPDFService.cls \
    --source-dir force-app/main/default/classes/PurchaseOrderPDFServiceTest.cls \
    --source-dir force-app/main/default/classes/PurchaseOrderEmailService.cls \
    --source-dir force-app/main/default/classes/PurchaseOrderEmailServiceTest.cls \
    --source-dir force-app/main/default/classes/PurchaseOrderAuthorizationService.cls \
    --target-org "${ORG}" \
    --test-level NoTestRun \
    --wait 10

echo "// all waves complete"
```

### PDF Service with Test.isRunningTest() Pattern

```apex
public with sharing class PurchaseOrderPDFService {

    // flag to allow tests to bypass the getContentAsPDF() callout
    // set this to true in test methods to substitute a dummy blob
    @TestVisible
    private static Boolean useDummyPDF = false;

    // generate a PDF blob for the given Purchase Order
    // @param poId: the Id of the Purchase_Order__c record
    // @return Blob: the rendered PDF content (dummy content in test context)
    public static Blob generatePDF(Id poId) {
        System.debug('Generating PDF for PO: ' + poId);

        // build the page reference to the Visualforce page
        // this is a compile-time reference -- the page must exist at deploy time
        PageReference pdfPage = Page.PurchaseOrderPDF;

        // pass the record id so the StandardController loads the correct record
        pdfPage.getParameters().put('id', poId);

        Blob pdfBlob;

        if (Test.isRunningTest() || useDummyPDF) {
            // getContentAsPDF() returns a blank blob in test context
            // substitute a dummy so downstream code (email attachment, file save)
            // can be tested without relying on the PDF engine
            pdfBlob = Blob.valueOf('Test PDF Content');
            System.debug('Using dummy PDF blob for test context');
        } else {
            // getContentAsPDF() counts as a callout
            // cannot be called after DML in the same transaction
            // the running user must have profile access to the VF page
            pdfBlob = pdfPage.getContentAsPDF();
        }

        System.debug('PDF blob size: ' + pdfBlob.size() + ' bytes');
        return pdfBlob;
    }
}
```

### Test Class Pattern for VF PDF Deployment

```apex
@isTest
private class PurchaseOrderPDFServiceTest {

    @TestSetup
    static void makeData() {
        // create test data that the VF page controller will query
        Purchase_Order__c po = new Purchase_Order__c(
            PO_Date__c = Date.today(),
            Status__c = 'Draft',
            Authorizer_Name__c = 'Test Authorizer',
            Payment_Terms__c = 'Net 30'
        );
        insert po;

        List<PO_Line_Item__c> items = new List<PO_Line_Item__c>();
        for (Integer i = 1; i <= 3; i++) {
            items.add(new PO_Line_Item__c(
                Purchase_Order__c = po.Id,
                Line_Number__c = i,
                Product_Name__c = 'Test Product ' + i,
                Quantity__c = 10,
                Unit_Price__c = 50.00
            ));
        }
        insert items;
    }

    // test 1: verify the controller extension loads data correctly
    // this tests Layer 2 independently of the PDF engine
    @isTest
    static void testControllerExtension() {
        Purchase_Order__c po = [SELECT Id FROM Purchase_Order__c LIMIT 1];
        ApexPages.StandardController stdCtrl = new ApexPages.StandardController(po);

        Test.startTest();
        PurchaseOrderPDFExtension ext = new PurchaseOrderPDFExtension(stdCtrl);
        Test.stopTest();

        System.assertEquals(3, ext.lineItems.size(),
            'Extension should query 3 line items');
    }

    // test 2: verify the PDF service doesn't throw and returns a blob
    // getContentAsPDF() returns a blank blob in test context
    // we can only assert non-null and no exception
    @isTest
    static void testGeneratePDF() {
        Purchase_Order__c po = [SELECT Id FROM Purchase_Order__c LIMIT 1];

        Test.startTest();
        Blob pdfBlob = PurchaseOrderPDFService.generatePDF(po.Id);
        Test.stopTest();

        System.assertNotEquals(null, pdfBlob,
            'PDF blob should not be null');
        System.assert(pdfBlob.size() > 0,
            'PDF blob should have content');
    }

    // test 3: verify the email service works with a dummy blob
    // this tests Layer 4 email logic without depending on PDF rendering
    @isTest
    static void testEmailServiceWithDummyBlob() {
        Purchase_Order__c po = [SELECT Id, Name FROM Purchase_Order__c LIMIT 1];

        Account acct = new Account(Name = 'Test Vendor');
        insert acct;

        Contact con = new Contact(
            FirstName = 'Test',
            LastName = 'Recipient',
            Email = 'test@example.com',
            AccountId = acct.Id
        );
        insert con;

        // use a dummy blob — do not rely on getContentAsPDF() for email testing
        Blob dummyPDF = Blob.valueOf('Dummy PDF for email test');

        Test.startTest();
        Boolean success = PurchaseOrderEmailService.sendPurchaseOrderPDF(
            po.Id,
            con.Id,
            dummyPDF,
            po.Name + '.pdf'
        );
        Test.stopTest();

        System.assertEquals(true, success,
            'Email should queue successfully with dummy PDF blob');
    }

    // test 4: verify the @TestVisible flag works for integration tests
    @isTest
    static void testUseDummyPDFFlag() {
        Purchase_Order__c po = [SELECT Id FROM Purchase_Order__c LIMIT 1];

        // use the @TestVisible flag to explicitly control PDF generation
        PurchaseOrderPDFService.useDummyPDF = true;

        Test.startTest();
        Blob pdfBlob = PurchaseOrderPDFService.generatePDF(po.Id);
        Test.stopTest();

        System.assertNotEquals(null, pdfBlob,
            'Dummy PDF blob should be returned when flag is set');
    }
}
```

### VF Page Access via Profile (Metadata)

To grant VF page access via a profile in metadata, include the profile in your deployment with the Visualforce page access entry:

```xml
<!-- in Admin.profile-meta.xml (partial) -->
<pageAccesses>
    <apexPage>PurchaseOrderPDF</apexPage>
    <enabled>true</enabled>
</pageAccesses>
```

To grant access via a permission set instead:

```xml
<!-- in PO_PDF_Access.permissionset-meta.xml (partial) -->
<?xml version="1.0" encoding="UTF-8"?>
<PermissionSet xmlns="http://soap.sforce.com/2006/04/metadata">
    <label>PO PDF Access</label>
    <description>Grants access to the Purchase Order PDF Visualforce page</description>
    <pageAccesses>
        <apexPage>PurchaseOrderPDF</apexPage>
        <enabled>true</enabled>
    </pageAccesses>
    <!-- also grant the Apex class access that the page controller needs -->
    <classAccesses>
        <apexClass>PurchaseOrderPDFExtension</apexClass>
        <enabled>true</enabled>
    </classAccesses>
    <classAccesses>
        <apexClass>PurchaseOrderPDFService</apexClass>
        <enabled>true</enabled>
    </classAccesses>
</PermissionSet>
```

### Package.xml Including Permission Sets for VF Access

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Package xmlns="http://soap.sforce.com/2006/04/metadata">
    <version>62.0</version>

    <types>
        <members>CompanyLogo</members>
        <name>StaticResource</name>
    </types>

    <types>
        <members>PurchaseOrderPDFExtension</members>
        <members>PurchaseOrderPDFService</members>
        <members>PurchaseOrderPDFServiceTest</members>
        <name>ApexClass</name>
    </types>

    <types>
        <members>PurchaseOrderPDF</members>
        <name>ApexPage</name>
    </types>

    <types>
        <members>PO_PDF_Access</members>
        <name>PermissionSet</name>
    </types>
</Package>
```

## Common Mistakes

1. **Deploying the service class before the VF page exists in the target org** -- The Apex class contains `Page.PurchaseOrderPDF` which is a compile-time reference. If the page does not exist in the target org, the deployment fails with `Variable does not exist: Page.PurchaseOrderPDF`. Fix: deploy in waves (controllers first, then VF pages, then service classes), or deploy everything together in a single `--source-dir` deployment so the Metadata API resolves the ordering automatically.

2. **Deploying a VF page before its controller exists in the target org** -- The `<apex:page controller="MyController">` or `extensions="MyExtension"` attribute is validated at deploy time. If the referenced class is missing, the page deployment fails with `Apex class 'MyExtension' does not exist`. Fix: include the controller/extension class in the same deployment, or deploy the class first in a prior wave.

3. **Deploying a VF page that references a missing Static Resource** -- `{!URLFOR($Resource.CompanyLogo)}` is validated when the page is saved. If the Static Resource hasn't been deployed yet, the page deployment fails. Fix: deploy Static Resources before or alongside Visualforce pages. In a wave deploy, Static Resources go first.

4. **Putting `Page.MyPage` references inside the VF controller instead of a separate service class** -- This creates a circular dependency: the page requires the controller, and the controller requires the page. This works in a single deployment (the Metadata API resolves it), but breaks in wave deployments because you cannot deploy either one first. Fix: keep `Page.*` references in a dedicated service class (Layer 4) that is separate from the controller (Layer 2). The controller should never reference the page it belongs to.

5. **Mismatched API versions between the VF page and its controller** -- The page is on API 30.0 while the controller is on 62.0. Merge fields may resolve differently, CSP behavior changes, and Lightning stylesheets behave unexpectedly. In some cases this produces a blank PDF. Fix: open both `.page-meta.xml` and `.cls-meta.xml`, verify the `<apiVersion>` matches, and redeploy both files. When upgrading API versions, always upgrade the page and controller together.

6. **Forgetting to grant Visualforce page access on the running user's profile** -- `getContentAsPDF()` enforces VF page access checks. Even if the Apex class uses `without sharing` (which only bypasses record-level sharing rules), the running user's profile must have access to the VF page. If it doesn't, `getContentAsPDF()` throws an `Insufficient Privileges` error at runtime. Fix: grant VF page access via profile or permission set. Include the profile/permission set metadata in your deployment package.

7. **Asserting on actual PDF content in test classes** -- `getContentAsPDF()` returns a non-null `Blob` in test context, but the content is empty or minimal. Tests that parse the blob, check page count, or validate text content will always fail. Fix: use `Test.isRunningTest()` to substitute a dummy blob. Test the controller/extension data queries separately. Test the email attachment logic separately with a dummy blob.

8. **Not including Apex class access in permission sets alongside VF page access** -- The profile/permission set grants access to the VF page, but not to the Apex controller class. At runtime, the page loads but the controller throws an authorization error. Fix: always grant both `pageAccesses` and `classAccesses` in the permission set for the VF page's controller, extension, and any Apex classes called by the controller.

9. **Deploying a VF page with `confirmationTokenRequired="true"` when it's only used for server-side PDF generation** -- CSRF tokens are for browser-submitted forms. When a page is only accessed via `getContentAsPDF()` in Apex (no user browser session), the confirmation token check can cause `getContentAsPDF()` to fail in certain contexts. Fix: set `confirmationTokenRequired="false"` for pages that are exclusively rendered server-side via Apex. Only set it to `true` for pages that users access directly in a browser and that perform DML via forms.

10. **Deploying to a sandbox without granting VF page access, then debugging why PDF generation "works locally but not in sandbox"** -- Sandboxes refreshed from production inherit profile/permission configurations, but partial sandboxes and scratch orgs may not. Also, the running user in a CI/CD pipeline (integration user) may have a different profile than the developer who tested locally. Fix: include permission set assignments in your deployment. After deploying to a new org, run `sf org assign permset --name PO_PDF_Access --target-org my-sandbox` to ensure the running user has access.

## See Also

- [Apex Deployment & CI/CD](apex-deployment.md) -- general deployment commands, test coverage requirements, and validate-then-quick-deploy workflow
- [Visualforce PDF Generation](../visualforce-pdf/visualforce-pdf-generation.md) -- the VF page markup, controller extension, and `getContentAsPDF()` patterns
- [Email with Attachments](../email-services/email-with-attachments.md) -- sending the generated PDF as an email attachment
- [Solution Architecture: Purchase Order PDF Workflow](../solution-architecture/purchase-order-pdf-workflow.md) -- end-to-end design of the PO PDF system
