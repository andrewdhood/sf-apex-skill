# Visualforce Controllers and Page Security

> **When to read this:** You are building a Visualforce page and need to choose the right controller type, configure sharing context for security, create reusable components, embed Visualforce in Lightning Experience, or harden a page against CSRF/XSS/clickjacking.

## Rules

- When building a Visualforce page for a single record (detail, edit, or PDF), always start with a StandardController because it gives you automatic CRUD, merge field binding, and standard button overrides for free — only add a controller extension when you need custom logic the StandardController cannot provide.
- When writing a controller extension, always accept `ApexPages.StandardController` as the sole constructor parameter because Salesforce instantiates extensions by injecting the StandardController — any other constructor signature causes a compile error when the page loads.
- When building a Visualforce page that is not tied to any single SObject (dashboards, multi-object search, wizards), always use a custom controller because the StandardController requires an `id` parameter and will throw an error or show blank data if no record context exists.
- When building a list view page with pagination, always use `StandardSetController` (either declaratively via `recordSetVar` or programmatically via `ApexPages.StandardSetController`) because it handles pagination, sorting, and filter integration automatically.
- Always declare an explicit sharing keyword (`with sharing`, `without sharing`, or `inherited sharing`) on every Visualforce controller and extension class because omitting it defaults to `without sharing` behavior, which silently exposes records the running user should not see.
- Always default to `with sharing` on Visualforce controllers unless you have a documented, security-reviewed reason to use `without sharing` — this enforces the running user's record-level access and is required for AppExchange security review.
- When a `with sharing` controller calls a service class declared `without sharing`, the service class's sharing context wins for its own queries — always be intentional about sharing boundaries and document why each class uses the keyword it does.
- When creating reusable Visualforce components with `<apex:component>`, always define inputs as `<apex:attribute>` elements with explicit `type`, `description`, and `required` attributes because this makes the component self-documenting and causes clear compile-time errors when required attributes are missing.
- When embedding Lightning components in a Visualforce page, always include `<apex:includeLightning />` before the `$Lightning.use()` call because this tag loads the Lightning Out JavaScript library that the embedding depends on.
- When navigating from a Visualforce page in Lightning Experience, always use `sforce.one.navigateToSObject()` (or the appropriate `sforce.one` method) instead of `window.location` or `PageReference` redirects because VF pages run in an iframe in Lightning Experience and direct URL changes break the navigation stack.
- When outputting user-controlled data in a Visualforce page, always escape it with `{!HTMLENCODE(value)}` in HTML context and `{!JSENCODE(value)}` in JavaScript context because unescaped merge fields are the primary vector for stored XSS in Visualforce.
- Never set `escape="false"` on `<apex:outputText>` unless you are rendering trusted, sanitized HTML that you control — `escape="false"` disables automatic HTML encoding and opens the page to XSS.
- When a Visualforce page performs a state-changing action on load (creating, updating, or deleting records via GET request), always implement manual CSRF protection because `<apex:form>` only auto-protects POST submissions — GET-based actions have no automatic CSRF token.
- Always assign Visualforce pages to the correct profiles and permission sets in Setup (Visualforce Pages → Security) because even if a user can guess the URL, they cannot load the page without this explicit access grant.

## How It Works

### Controller Type Decision Matrix

| Scenario | Controller Type | Why |
|----------|----------------|-----|
| Single-record detail/edit page | `standardController="Account"` | Auto-queries record, provides save/edit/delete/cancel actions |
| Single-record page with child records or calculations | StandardController + Extension | Extension queries children, controller handles the parent |
| Single-record PDF with custom formatting | StandardController + Extension | Extension provides formatted data, controller binds the record |
| Multi-object dashboard or wizard | Custom Controller (`controller="MyCtrl"`) | No single SObject context; full control over queries |
| List view with pagination and filters | `recordSetVar="accounts"` (StandardSetController) | Automatic pagination, sorting, filter set integration |
| List view with custom query or complex filtering | Custom Controller wrapping `ApexPages.StandardSetController` | You control the query; SSC handles pagination mechanics |

### StandardController: Automatic Record Binding

When you set `standardController="Account"` on `<apex:page>`, Salesforce auto-instantiates an `ApexPages.StandardController` for the Account object. It reads the `id` URL parameter, queries the record, and makes all fields available as merge fields (`{!Account.Name}`, `{!Account.Industry}`, etc.). It also provides standard actions: `save`, `edit`, `delete`, `cancel`, `list`.

The StandardController only queries fields that are actually referenced on the page. If you reference `{!Account.Name}` and `{!Account.Industry}`, it queries only those two fields. This is efficient but means you cannot access unreferenced fields in an extension without querying them yourself.

### Controller Extensions: Adding Logic to StandardController

An extension class supplements the StandardController with custom properties and actions. The constructor receives the StandardController instance, giving the extension access to the current record via `getRecord()` or `getId()`.

Multiple extensions can be listed on a single page: `extensions="ExtA,ExtB"`. When both define the same property or action name, the **leftmost** extension wins. The StandardController's methods are used only for names not defined by any extension.

### Custom Controllers: Full Manual Control

A custom controller is an Apex class with a public no-argument constructor. It has no automatic record binding — you must query everything yourself. Custom controllers cannot be used with standard button overrides or the standard `save`/`edit`/`delete` actions. Use them when the page has no single-record context.

### StandardSetController: List Views and Pagination

`StandardSetController` works with collections of records. Declaratively, you set `recordSetVar="varName"` on `<apex:page>` and iterate with `<apex:repeat>` or `<apex:pageBlockTable>`. Programmatically, you instantiate `ApexPages.StandardSetController` with a `Database.QueryLocator` or `List<SObject>` and expose pagination actions (`next`, `previous`, `first`, `last`) and properties (`hasNext`, `hasPrevious`, `resultSize`, `pageSize`).

The default page size is 20. Maximum is 10,000 records in the query locator, but the practical display limit per page should stay at 100 or fewer for performance.

### Sharing Context and Security Layers

Record access on a Visualforce page is governed by the **controller's sharing keyword**, not by the page itself. There are three layers of security:

1. **Page-level access**: The user's profile or permission set must have the VF page enabled (Setup → Visualforce Pages → Security). Without this, the user gets "Insufficient Privileges."
2. **Object/Field-level access (CRUD/FLS)**: The user's profile must have Read (and Edit for forms) on the objects and fields shown. Sharing keywords do NOT enforce CRUD/FLS — that requires `WITH USER_MODE` in SOQL or `Security.stripInaccessible()`.
3. **Record-level access (sharing)**: Controlled by the controller's sharing keyword. `with sharing` = the user sees only records they have access to via OWD, sharing rules, role hierarchy, and manual shares. `without sharing` = the controller sees all records regardless.

**How sharing cascades across class calls:**

```
VF Page
  └─ Controller (with sharing)
       └─ ServiceA (without sharing)  ← runs WITHOUT sharing for its own queries
            └─ UtilityB (inherited sharing)  ← inherits WITHOUT sharing from ServiceA
       └─ ServiceC (with sharing)  ← runs WITH sharing regardless of caller
```

Each class enforces its own declared sharing mode. `inherited sharing` defers to the caller. This is why documenting the sharing keyword on every class matters — a `without sharing` service called from a `with sharing` controller opens a sharing bypass for anything the service queries.

**For PDF generation:** The user who triggers the PDF (or the running user context for `getContentAsPDF()`) must have access to the record. If the controller uses `with sharing` and the user cannot see the record, the PDF renders with blank or missing data. If the controller uses `without sharing`, the PDF renders all data regardless of the user's access. Choose deliberately based on whether the PDF is user-facing or system-generated.

**For guest user / Sites pages:** Guest users run under the Guest User profile with extremely limited access. A `with sharing` controller on a public Site page means the guest user can only see records explicitly shared with the Guest User profile via sharing rules or OWD=Public Read. Use `without sharing` carefully and combine it with explicit field-level filtering in your controller to avoid leaking sensitive data.

### Visualforce Components (`<apex:component>`)

Visualforce components let you create reusable UI fragments. A component is a `.component` file that supports `<apex:attribute>` for inputs and can optionally have its own controller. Components are referenced from pages with `<c:ComponentName attributeName="value" />`.

**Key rules:**
- A component can have a custom controller via the `controller` attribute on `<apex:component>`.
- The component controller is a separate class from the page controller. They do not automatically share data.
- To pass data from the page to the component's controller, use the `assignTo` attribute on `<apex:attribute>` to bind the attribute to a controller property.
- Components cannot have their own `standardController` — only custom controllers.
- Components support `<apex:componentBody>` for slot-like content injection from the parent page.

### Visualforce in Lightning Experience

Visualforce pages run inside an **iframe** when displayed in Lightning Experience. This has significant implications:

**Embedding VF in Lightning:**
- **Lightning App Builder**: Add a VF page as a component in a Lightning record page, app page, or home page. The page must have "Available for Lightning Experience, Lightning Communities, and the mobile app" checked.
- **Lightning Tabs**: Create a Visualforce Tab (Setup → Tabs → Visualforce Tabs) to expose a VF page as a top-level tab in Lightning Experience.
- **Utility Bar**: VF pages can appear in the utility bar as a persistent panel.

**Embedding Lightning in VF (Lightning Out):**
- Add `<apex:includeLightning />` to load the Lightning Out library.
- Call `$Lightning.use("c:MyApp", callback)` where `MyApp` is a Lightning Application with `extends="ltng:outApp"`.
- In the callback, call `$Lightning.createComponent()` to render Aura or LWC components into a DOM element on the VF page.
- This requires the Lightning dependency app to list all components it uses in its `.app` file.

**Navigation (`sforce.one`):**
- `sforce.one` is automatically available on VF pages in Lightning Experience and the Salesforce mobile app.
- Use `sforce.one.navigateToSObject(recordId)` to navigate to a record.
- Use `sforce.one.navigateToURL('/lightning/o/Account/list')` for arbitrary URLs.
- Do NOT use `window.location` — the VF page is in an iframe and setting `window.location` navigates only the iframe, not the outer Lightning shell.
- To detect the current UI context, check `typeof sforce !== 'undefined' && sforce.one` — if truthy, you are in Lightning/mobile.

**Styling (`<apex:slds>`):**
- Add `<apex:slds />` to your page to load the Lightning Design System CSS.
- This is separate from `lightningStylesheets="true"`, which reskins standard VF components to look like Lightning. `<apex:slds>` gives you the full SLDS CSS framework for custom markup.
- Wrap your SLDS-styled content in `<div class="slds-scope">` to prevent CSS conflicts with standard VF styles.

**When to use VF in Lightning vs. migrate to LWC:**
- Keep VF for: PDF rendering (`renderAs="pdf"`), complex pages with massive ViewState that would require full rewrites, pages embedded in Classic that must also work in Lightning.
- Migrate to LWC for: new development, interactive record pages, anything that needs modern Lightning features (Lightning Message Service, wire adapters, reactive data).
- VF pages in Lightning load slower than native LWC due to iframe overhead and additional HTTP requests.

### VF Page Security

**CSRF Protection:**
- `<apex:form>` automatically includes a hidden CSRF token (`com.salesforce.visualforce.ViewStateCSRF`). When the form is submitted via POST, Salesforce validates this token. If it is missing or tampered with, the request is rejected.
- GET requests are NOT automatically protected. If your page performs DML on page load (constructor or action method invoked by the `action` attribute on `<apex:page>`), you must implement your own CSRF token: generate a random token, store it in the user's session, pass it as a URL parameter, and validate it in the controller before performing any DML.
- Best practice: never perform state-changing operations on page load via GET. Use a form submission (POST) to trigger DML.

**Clickjack Protection:**
- Salesforce sets `X-Frame-Options` headers on VF pages based on org-level Session Settings (Setup → Session Settings → Clickjack Protection).
- Three levels: "Allow framing by any page" (no protection), "Allow framing by the same origin only" (SAMEORIGIN), "Don't allow framing" (DENY).
- For VF pages that must be iframed on external domains, configure trusted domains in Setup → Session Settings → Trusted Domains for Inline Frames.
- The `showHeader="false"` attribute does NOT affect clickjack protection — that is controlled entirely at the session settings level.

**Content Security Policy:**
- VF pages do not enforce CSP headers the same way Lightning components do. VF pages allow inline JavaScript and inline styles by default.
- When using Lightning Out (`<apex:includeLightning />`) in a VF page, the embedded Lightning components ARE subject to Lightning's CSP rules. Add required domains to CSP Trusted Sites (Setup → CSP Trusted Sites).
- If a VF page loads external scripts or makes fetch/XHR calls to external APIs, add those domains to Remote Site Settings (Setup → Remote Site Settings) for Apex callouts and CSP Trusted Sites for client-side JavaScript.

**Page Access Control:**
- VF pages require explicit profile or permission set assignment. Go to Setup → Visualforce Pages → find the page → Security button → move profiles from Available to Enabled.
- Alternatively, assign VF page access in a Permission Set (App Permissions or in the Visualforce Page Access section).
- Even if the controller class is `global`, the user cannot load the page without page-level access. The page URL returns "Insufficient Privileges."
- For Salesforce Sites (public-facing), the Guest User Profile must have the VF page enabled, plus object/field permissions for any data the page displays.

**`readOnly="true"` on `<apex:page>`:**
- Increases the maximum collection size for `<apex:repeat>`, `<apex:dataTable>`, and `<apex:dataList>` from 1,000 to 10,000 records.
- Increases the SOQL query row limit from 50,000 to 1,000,000 rows for the page's controller.
- The page cannot perform any DML. Any attempt to insert, update, delete, or undelete throws a runtime exception.
- Useful for report-style pages and large data exports. Not useful for edit forms.

**Output Escaping (XSS Prevention):**
- Standard VF components (`<apex:outputText>`, `<apex:outputField>`, `<apex:inputField>`) auto-escape output by default. The `escape` attribute defaults to `true`.
- Raw merge fields in HTML context (`<div>{!Account.Name}</div>`) are NOT automatically escaped in all contexts. If the field value contains `<script>`, it could execute.
- Always use `{!HTMLENCODE(value)}` when outputting user-editable text in raw HTML context outside of VF components.
- Use `{!JSENCODE(value)}` when injecting Apex values into JavaScript strings.
- Use `{!JSINHTMLENCODE(value)}` when injecting values into JavaScript that is itself inside an HTML attribute (e.g., `onclick="doThing('{!JSINHTMLENCODE(val)}')"`) — this double-encodes for both contexts.
- Use `{!URLENCODE(value)}` when building URLs with dynamic parameters.

## Code Examples

### StandardController with Extension (Invoice PDF)

**Visualforce Page:**

```xml
<apex:page standardController="Invoice__c"
           extensions="InvoicePDFExtension"
           renderAs="pdf"
           applyBodyTag="false"
           applyHtmlTag="false"
           showHeader="false"
           sidebar="false"
           standardStylesheets="false">

<html>
<head>
    <style type="text/css">
        @page {
            size: letter;
            margin: 0.75in;
            @bottom-center {
                content: element(footer);
            }
        }
        body {
            font-family: Arial, Helvetica, sans-serif;
            font-size: 10pt;
            color: #333;
        }
        .header { border-bottom: 2px solid #003366; padding-bottom: 10px; margin-bottom: 20px; }
        .header h1 { color: #003366; font-size: 16pt; margin: 0; }
        .info-table { width: 100%; border-collapse: collapse; margin-bottom: 15px; }
        .info-table td { padding: 3px 8px; font-size: 9pt; vertical-align: top; }
        .info-table .label { font-weight: bold; width: 120px; color: #003366; }
        .line-items { width: 100%; border-collapse: collapse; }
        .line-items th { background: #003366; color: #fff; padding: 6px 8px; font-size: 9pt; text-align: left; }
        .line-items td { padding: 5px 8px; border-bottom: 1px solid #ccc; font-size: 9pt; }
        .currency { text-align: right; }
        .footer { position: running(footer); font-size: 7pt; color: #999; text-align: center; }
    </style>
</head>
<body>
    <div class="footer">
        Invoice {!Invoice__c.Name} | Page <span class="pagenumber"/> of <span class="pagecount"/>
    </div>

    <!-- reusable header component — defined below -->
    <c:PDFLetterhead companyName="{!companyName}"
                     companyAddress="{!companyAddress}"
                     logoResource="CompanyLogo" />

    <h1 style="color: #003366;">Invoice {!Invoice__c.Name}</h1>

    <table class="info-table">
        <tr>
            <td class="label">Invoice Date:</td>
            <td>
                <apex:outputText value="{0,date,MM/dd/yyyy}">
                    <apex:param value="{!Invoice__c.Invoice_Date__c}" />
                </apex:outputText>
            </td>
            <td class="label">Due Date:</td>
            <td>
                <apex:outputText value="{0,date,MM/dd/yyyy}">
                    <apex:param value="{!Invoice__c.Due_Date__c}" />
                </apex:outputText>
            </td>
        </tr>
        <tr>
            <td class="label">Bill To:</td>
            <td>{!Invoice__c.Account__r.Name}</td>
            <td class="label">Status:</td>
            <td>{!Invoice__c.Status__c}</td>
        </tr>
    </table>

    <table class="line-items">
        <thead>
            <tr>
                <th>#</th>
                <th>Description</th>
                <th>Qty</th>
                <th class="currency">Unit Price</th>
                <th class="currency">Amount</th>
            </tr>
        </thead>
        <tbody>
            <apex:repeat value="{!lineItems}" var="li">
                <tr>
                    <td>{!li.Line_Number__c}</td>
                    <td>{!li.Description__c}</td>
                    <td>{!li.Quantity__c}</td>
                    <td class="currency">
                        <apex:outputText value="{0, number, $#,##0.00}">
                            <apex:param value="{!li.Unit_Price__c}" />
                        </apex:outputText>
                    </td>
                    <td class="currency">
                        <apex:outputText value="{0, number, $#,##0.00}">
                            <apex:param value="{!li.Line_Total__c}" />
                        </apex:outputText>
                    </td>
                </tr>
            </apex:repeat>
        </tbody>
    </table>

    <div style="text-align: right; margin-top: 15px;">
        <strong style="font-size: 12pt; color: #003366;">
            Total:
            <apex:outputText value="{0, number, $#,##0.00}">
                <apex:param value="{!Invoice__c.Total_Amount__c}" />
            </apex:outputText>
        </strong>
    </div>

</body>
</html>
</apex:page>
```

**Controller Extension:**

```apex
public with sharing class InvoicePDFExtension {
    // the invoice record bound by the StandardController
    private Invoice__c invoice;

    // line items for the invoice table
    public List<Invoice_Line_Item__c> lineItems { get; private set; }

    // company details surfaced for the PDF header component
    public String companyName { get; private set; }
    public String companyAddress { get; private set; }

    // constructor — receives the StandardController injected by the VF page
    // @param stdController: auto-injected StandardController for Invoice__c
    public InvoicePDFExtension(ApexPages.StandardController stdController) {
        this.invoice = (Invoice__c) stdController.getRecord();

        System.debug('InvoicePDFExtension init for Invoice: ' + this.invoice.Id);

        // query child line items — the StandardController only handles the parent
        this.lineItems = [
            SELECT Id, Line_Number__c, Description__c, Quantity__c,
                   Unit_Price__c, Line_Total__c
            FROM Invoice_Line_Item__c
            WHERE Invoice__c = :this.invoice.Id
            ORDER BY Line_Number__c ASC
        ];

        System.debug('Loaded ' + this.lineItems.size() + ' line items');

        // pull company info from a custom setting or org defaults
        // hardcoded here for clarity — in production, query Organization or a Custom Setting
        this.companyName = 'Acme Corporation';
        this.companyAddress = '123 Main St, Suite 100, Springfield, IL 62704';
    }
}
```

### Reusable PDF Header Component

**Component file (`PDFLetterhead.component`):**

```xml
<apex:component>
    <!-- define the inputs this component accepts -->
    <apex:attribute name="companyName"
                    type="String"
                    required="true"
                    description="Company name displayed in the header" />

    <apex:attribute name="companyAddress"
                    type="String"
                    required="true"
                    description="Company mailing address" />

    <apex:attribute name="logoResource"
                    type="String"
                    required="false"
                    description="Static Resource name for the company logo" />

    <apex:attribute name="showDivider"
                    type="Boolean"
                    default="true"
                    description="Whether to show the horizontal rule below the header" />

    <!-- render the header markup -->
    <div style="text-align: center; margin-bottom: 15px;">
        <!-- logo from static resource — only render if a resource name was provided -->
        <apex:image url="{!URLFOR($Resource[logoResource])}"
                    width="140"
                    rendered="{!NOT(ISBLANK(logoResource))}" />

        <div style="font-size: 14pt; font-weight: bold; color: #003366; margin-top: 5px;">
            {!companyName}
        </div>
        <div style="font-size: 8pt; color: #666666; margin-top: 2px;">
            {!companyAddress}
        </div>
    </div>

    <apex:outputPanel rendered="{!showDivider}">
        <hr style="border: none; border-top: 2px solid #003366; margin-bottom: 15px;" />
    </apex:outputPanel>
</apex:component>
```

**Using the component from any PDF page:**

```xml
<!-- in your Visualforce page body -->
<c:PDFLetterhead companyName="Acme Corporation"
                 companyAddress="123 Main St, Suite 100, Springfield, IL 62704"
                 logoResource="CompanyLogo" />

<!-- or with dynamic values from the controller -->
<c:PDFLetterhead companyName="{!companyName}"
                 companyAddress="{!companyAddress}"
                 logoResource="CompanyLogo"
                 showDivider="false" />
```

### Component with Its Own Controller

**Component file (`AccountSummary.component`):**

```xml
<apex:component controller="AccountSummaryController">
    <!-- the account Id is passed from the parent page and bound to the controller -->
    <apex:attribute name="accountId"
                    type="Id"
                    required="true"
                    assignTo="{!acctId}"
                    description="The Account record Id to summarize" />

    <div class="account-summary">
        <h3>{!account.Name}</h3>
        <p>Industry: {!account.Industry}</p>
        <p>Open Opportunities: {!openOppCount}</p>
        <p>Total Revenue: 
            <apex:outputText value="{0, number, $#,##0.00}">
                <apex:param value="{!totalRevenue}" />
            </apex:outputText>
        </p>
    </div>
</apex:component>
```

**Component Controller:**

```apex
public with sharing class AccountSummaryController {
    // bound from the apex:attribute via assignTo
    // when the page sets accountId="001XXXX", this property receives that value
    public Id acctId { get; set; }

    // the account record — queried when the getter is first called
    public Account account {
        get {
            if (account == null && acctId != null) {
                account = [
                    SELECT Id, Name, Industry
                    FROM Account
                    WHERE Id = :acctId
                    LIMIT 1
                ];
            }
            return account;
        }
        private set;
    }

    // count of open opportunities
    public Integer openOppCount {
        get {
            if (openOppCount == null && acctId != null) {
                openOppCount = [
                    SELECT COUNT()
                    FROM Opportunity
                    WHERE AccountId = :acctId
                    AND IsClosed = false
                ];
            }
            return openOppCount;
        }
        private set;
    }

    // total closed-won revenue
    public Decimal totalRevenue {
        get {
            if (totalRevenue == null && acctId != null) {
                AggregateResult ar = [
                    SELECT SUM(Amount) total
                    FROM Opportunity
                    WHERE AccountId = :acctId
                    AND IsWon = true
                ];
                totalRevenue = (Decimal) ar.get('total');
                if (totalRevenue == null) {
                    totalRevenue = 0;
                }
            }
            return totalRevenue;
        }
        private set;
    }
}
```

### Custom Controller (Multi-Object Dashboard)

```apex
public with sharing class DashboardController {
    // recent accounts created in the last 30 days
    public List<Account> recentAccounts { get; private set; }

    // open high-priority cases
    public List<Case> highPriorityCases { get; private set; }

    // summary statistics
    public Integer totalOpenOpps { get; private set; }
    public Decimal pipelineValue { get; private set; }

    // no-argument constructor — required for custom controllers
    public DashboardController() {
        System.debug('DashboardController initialized');

        // query recent accounts
        this.recentAccounts = [
            SELECT Id, Name, Industry, CreatedDate
            FROM Account
            WHERE CreatedDate = LAST_N_DAYS:30
            ORDER BY CreatedDate DESC
            LIMIT 10
        ];

        // query high-priority open cases
        this.highPriorityCases = [
            SELECT Id, CaseNumber, Subject, Status, Priority, Account.Name
            FROM Case
            WHERE IsClosed = false
            AND Priority = 'High'
            ORDER BY CreatedDate DESC
            LIMIT 10
        ];

        // aggregate open opportunity stats
        AggregateResult oppStats = [
            SELECT COUNT(Id) total, SUM(Amount) pipeline
            FROM Opportunity
            WHERE IsClosed = false
        ];
        this.totalOpenOpps = (Integer) oppStats.get('total');
        this.pipelineValue = (Decimal) oppStats.get('pipeline');
        if (this.pipelineValue == null) {
            this.pipelineValue = 0;
        }

        System.debug('Dashboard loaded: ' + this.recentAccounts.size() + ' accounts, '
            + this.highPriorityCases.size() + ' cases, '
            + this.totalOpenOpps + ' open opps');
    }
}
```

**Visualforce page using the custom controller:**

```xml
<apex:page controller="DashboardController"
           title="Executive Dashboard"
           showHeader="true"
           sidebar="false"
           lightningStylesheets="true">

    <apex:pageBlock title="Pipeline Summary">
        <apex:pageBlockSection columns="2">
            <apex:pageBlockSectionItem>
                <apex:outputLabel value="Open Opportunities" />
                <apex:outputText value="{!totalOpenOpps}" />
            </apex:pageBlockSectionItem>
            <apex:pageBlockSectionItem>
                <apex:outputLabel value="Pipeline Value" />
                <apex:outputText value="{0, number, $#,##0.00}">
                    <apex:param value="{!pipelineValue}" />
                </apex:outputText>
            </apex:pageBlockSectionItem>
        </apex:pageBlockSection>
    </apex:pageBlock>

    <apex:pageBlock title="Recently Created Accounts">
        <apex:pageBlockTable value="{!recentAccounts}" var="a">
            <apex:column value="{!a.Name}" />
            <apex:column value="{!a.Industry}" />
            <apex:column value="{!a.CreatedDate}" />
        </apex:pageBlockTable>
    </apex:pageBlock>

    <apex:pageBlock title="High Priority Cases">
        <apex:pageBlockTable value="{!highPriorityCases}" var="c">
            <apex:column value="{!c.CaseNumber}" />
            <apex:column value="{!c.Subject}" />
            <apex:column value="{!c.Status}" />
            <apex:column value="{!c.Account.Name}" />
        </apex:pageBlockTable>
    </apex:pageBlock>
</apex:page>
```

### StandardSetController with Pagination

**Controller Extension:**

```apex
public with sharing class AccountListExtension {
    // the set controller handles pagination state
    private ApexPages.StandardSetController setCtrl;

    // constructor — receives the StandardSetController from the list page
    // @param controller: auto-injected StandardSetController for Account
    public AccountListExtension(ApexPages.StandardSetController controller) {
        this.setCtrl = controller;
        // set page size — default is 20, max display recommended is 100
        this.setCtrl.setPageSize(25);

        System.debug('AccountListExtension init, total records: ' + this.setCtrl.getResultSize());
    }

    // expose records as a typed list for the page
    public List<Account> getAccounts() {
        return (List<Account>) this.setCtrl.getRecords();
    }

    // pagination properties for enabling/disabling prev/next buttons
    public Boolean hasNext {
        get { return this.setCtrl.getHasNext(); }
    }

    public Boolean hasPrevious {
        get { return this.setCtrl.getHasPrevious(); }
    }

    public Integer pageNumber {
        get { return this.setCtrl.getPageNumber(); }
    }

    public Integer totalPages {
        get {
            Integer totalRecords = this.setCtrl.getResultSize();
            Integer pageSize = this.setCtrl.getPageSize();
            // ceiling division
            return (Integer) Math.ceil((Decimal) totalRecords / pageSize);
        }
    }

    // navigation actions
    public void nextPage() {
        this.setCtrl.next();
    }

    public void previousPage() {
        this.setCtrl.previous();
    }

    public void firstPage() {
        this.setCtrl.first();
    }

    public void lastPage() {
        this.setCtrl.last();
    }
}
```

**Visualforce List Page:**

```xml
<apex:page standardController="Account"
           recordSetVar="accounts"
           extensions="AccountListExtension"
           lightningStylesheets="true">

    <apex:form>
        <apex:pageBlock title="Accounts">
            <apex:pageBlockTable value="{!accounts}" var="a">
                <apex:column value="{!a.Name}" />
                <apex:column value="{!a.Industry}" />
                <apex:column value="{!a.Phone}" />
                <apex:column value="{!a.AnnualRevenue}" />
            </apex:pageBlockTable>

            <!-- pagination controls -->
            <apex:panelGrid columns="5" style="margin-top: 10px;">
                <apex:commandButton value="« First"
                                    action="{!firstPage}"
                                    disabled="{!NOT(hasPrevious)}"
                                    reRender="form" />
                <apex:commandButton value="‹ Previous"
                                    action="{!previousPage}"
                                    disabled="{!NOT(hasPrevious)}"
                                    reRender="form" />
                <apex:outputText value="Page {!pageNumber} of {!totalPages}" />
                <apex:commandButton value="Next ›"
                                    action="{!nextPage}"
                                    disabled="{!NOT(hasNext)}"
                                    reRender="form" />
                <apex:commandButton value="Last »"
                                    action="{!lastPage}"
                                    disabled="{!NOT(hasNext)}"
                                    reRender="form" />
            </apex:panelGrid>
        </apex:pageBlock>
    </apex:form>
</apex:page>
```

### StandardSetController with Custom Query

```apex
public with sharing class OpenCaseListController {
    // wrap a custom query in StandardSetController for pagination
    private ApexPages.StandardSetController setCtrl;

    public OpenCaseListController() {
        // use Database.getQueryLocator for efficient pagination over large result sets
        Database.QueryLocator ql = Database.getQueryLocator([
            SELECT Id, CaseNumber, Subject, Status, Priority, CreatedDate, Account.Name
            FROM Case
            WHERE IsClosed = false
            ORDER BY CreatedDate DESC
        ]);

        this.setCtrl = new ApexPages.StandardSetController(ql);
        this.setCtrl.setPageSize(50);

        System.debug('OpenCaseListController init, result size: ' + this.setCtrl.getResultSize());
    }

    public List<Case> getCases() {
        return (List<Case>) this.setCtrl.getRecords();
    }

    public Boolean getHasNext() { return this.setCtrl.getHasNext(); }
    public Boolean getHasPrevious() { return this.setCtrl.getHasPrevious(); }

    public void next() { this.setCtrl.next(); }
    public void previous() { this.setCtrl.previous(); }
}
```

### Embedding Lightning Components in Visualforce (Lightning Out)

**Lightning Application (dependency app):**

```xml
<!-- MyLightningOutApp.app -->
<aura:application extends="ltng:outApp" access="global">
    <!-- declare every component this app will create -->
    <aura:dependency resource="c:AccountCard" />
    <aura:dependency resource="lightning:card" />
    <aura:dependency resource="lightning:formattedNumber" />
</aura:application>
```

**Visualforce Page:**

```xml
<apex:page showHeader="true" sidebar="false" standardStylesheets="false">
    <!-- step 1: load the Lightning Out JavaScript library -->
    <apex:includeLightning />

    <!-- step 2: provide a container div for the Lightning component -->
    <div id="lightning-container" style="min-height: 200px;"></div>

    <script>
        // step 3: initialize the Lightning Out app
        // the first argument is the dependency app name (namespace:appName)
        $Lightning.use("c:MyLightningOutApp", function() {

            // step 4: create the component inside the container div
            $Lightning.createComponent(
                "c:AccountCard",           // component name
                {                          // attributes to pass
                    recordId: "{!$CurrentPage.parameters.id}"
                },
                "lightning-container",     // DOM element id to render into
                function(cmp, status, errorMessage) {
                    // callback after component is created
                    if (status === "SUCCESS") {
                        console.log("Lightning component loaded successfully");
                    } else if (status === "INCOMPLETE") {
                        console.warn("No response from server");
                    } else if (status === "ERROR") {
                        console.error("Error creating component: " + errorMessage);
                    }
                }
            );
        });
    </script>
</apex:page>
```

### Lightning-Aware Navigation from Visualforce

```xml
<apex:page standardController="Account" extensions="AccountActionExtension">
    <apex:form>
        <apex:pageBlock title="Account Actions">
            <apex:commandButton value="Create Related Case"
                                action="{!createCase}"
                                oncomplete="navigateToRecord('{!newCaseId}');" />
        </apex:pageBlock>
    </apex:form>

    <script>
        // detect whether we are in Lightning Experience or Classic
        // and navigate accordingly
        function navigateToRecord(recordId) {
            if (!recordId) {
                return;
            }

            // sforce.one is available in Lightning Experience and SF mobile app
            if (typeof sforce !== 'undefined' && sforce.one) {
                // Lightning Experience — navigate using the one.app container
                sforce.one.navigateToSObject(recordId);
            } else {
                // Salesforce Classic — standard URL redirect
                window.location.href = '/' + recordId;
            }
        }
    </script>
</apex:page>
```

### Secure Output Escaping Examples

```xml
<apex:page standardController="Account">
    <!-- SAFE: apex:outputField auto-escapes and respects FLS -->
    <apex:outputField value="{!Account.Name}" />

    <!-- SAFE: apex:outputText with escape="true" (default) -->
    <apex:outputText value="{!Account.Description}" />

    <!-- DANGEROUS: raw merge field in HTML — could contain <script> tags -->
    <div>{!Account.Description}</div>

    <!-- SAFE: use HTMLENCODE when outputting in raw HTML context -->
    <div>{!HTMLENCODE(Account.Description)}</div>

    <!-- SAFE: use JSENCODE when injecting into JavaScript -->
    <script>
        var accountName = '{!JSENCODE(Account.Name)}';
        console.log('Account: ' + accountName);
    </script>

    <!-- DANGEROUS: unescaped in JavaScript — XSS if Name contains quotes or script -->
    <script>
        var accountName = '{!Account.Name}';  // DO NOT DO THIS
    </script>

    <!-- SAFE: JSINHTMLENCODE for JS inside HTML attributes -->
    <button onclick="alert('{!JSINHTMLENCODE(Account.Name)}')">Show Name</button>

    <!-- SAFE: URLENCODE for dynamic URL parameters -->
    <a href="/apex/DetailPage?name={!URLENCODE(Account.Name)}">View Details</a>
</apex:page>
```

### Test Class for Controller Extension and Custom Controller

```apex
@isTest
private class ControllerSecurityTest {

    @TestSetup
    static void makeData() {
        Account acct = new Account(Name = 'Test Account', Industry = 'Technology');
        insert acct;

        Invoice__c inv = new Invoice__c(
            Account__c = acct.Id,
            Invoice_Date__c = Date.today(),
            Due_Date__c = Date.today().addDays(30),
            Status__c = 'Draft'
        );
        insert inv;

        List<Invoice_Line_Item__c> items = new List<Invoice_Line_Item__c>();
        for (Integer i = 1; i <= 3; i++) {
            items.add(new Invoice_Line_Item__c(
                Invoice__c = inv.Id,
                Line_Number__c = i,
                Description__c = 'Service Item ' + i,
                Quantity__c = i * 2,
                Unit_Price__c = 100.00
            ));
        }
        insert items;
    }

    @isTest
    static void testInvoicePDFExtension() {
        Invoice__c inv = [SELECT Id FROM Invoice__c LIMIT 1];

        // simulate the StandardController as if the page loaded with this record
        ApexPages.StandardController stdCtrl = new ApexPages.StandardController(inv);

        Test.startTest();
        InvoicePDFExtension ext = new InvoicePDFExtension(stdCtrl);
        Test.stopTest();

        // verify line items were queried
        System.assertEquals(3, ext.lineItems.size(), 'Should load 3 line items');
        // verify company info is populated
        System.assertNotEquals(null, ext.companyName, 'Company name should be populated');
    }

    @isTest
    static void testDashboardCustomController() {
        Test.startTest();
        DashboardController ctrl = new DashboardController();
        Test.stopTest();

        // verify data was queried — we created 1 Account in test setup
        System.assertEquals(1, ctrl.recentAccounts.size(), 'Should find the test account');
        System.assertNotEquals(null, ctrl.pipelineValue, 'Pipeline value should not be null');
    }

    @isTest
    static void testAccountListExtensionPagination() {
        // create enough accounts to test pagination
        List<Account> accounts = new List<Account>();
        for (Integer i = 0; i < 30; i++) {
            accounts.add(new Account(Name = 'Pagination Test ' + i));
        }
        insert accounts;

        // set up the StandardSetController as the page would
        ApexPages.StandardSetController setCtrl = new ApexPages.StandardSetController(
            Database.getQueryLocator([SELECT Id, Name, Industry, Phone FROM Account ORDER BY Name])
        );

        Test.startTest();
        AccountListExtension ext = new AccountListExtension(setCtrl);
        Test.stopTest();

        // verify pagination state — page size is 25, we have 31 records (30 + 1 from setup)
        System.assertEquals(1, ext.pageNumber, 'Should start on page 1');
        System.assertEquals(true, ext.hasNext, 'Should have a next page');
        System.assertEquals(false, ext.hasPrevious, 'Should not have a previous page on page 1');

        // navigate to next page
        ext.nextPage();
        System.assertEquals(2, ext.pageNumber, 'Should be on page 2 after next');
        System.assertEquals(true, ext.hasPrevious, 'Should have a previous page on page 2');
    }

    @isTest
    static void testAccountSummaryComponentController() {
        Account acct = [SELECT Id FROM Account WHERE Name = 'Test Account' LIMIT 1];

        Test.startTest();
        AccountSummaryController ctrl = new AccountSummaryController();
        ctrl.acctId = acct.Id;

        // access the properties — lazy-loaded on first get
        Account result = ctrl.account;
        Integer oppCount = ctrl.openOppCount;
        Decimal revenue = ctrl.totalRevenue;
        Test.stopTest();

        System.assertEquals('Test Account', result.Name, 'Should query the correct account');
        System.assertEquals(0, oppCount, 'Should have 0 open opps in test');
        System.assertEquals(0, revenue, 'Should have 0 revenue with no closed opps');
    }
}
```

## Common Mistakes

1. **Using a custom controller when a StandardController + extension would suffice** — Custom controllers lose standard button override support, standard actions (save, edit, delete, cancel), and automatic merge field resolution. If your page operates on a single SObject, start with StandardController and add an extension for custom logic. Only use a custom controller when there is genuinely no single-record context.

2. **Omitting the sharing keyword on a Visualforce controller** — When no sharing keyword is declared, the class runs as `without sharing` in most contexts. This means every SOQL query in the controller ignores the running user's sharing rules and returns all records. The developer may not notice during testing (admins see everything), but it creates a security vulnerability in production. Always explicitly declare `with sharing` or document why `without sharing` is needed.

3. **Assuming `without sharing` on the controller bypasses CRUD/FLS** — Sharing keywords only control record-level access (which records appear in SOQL results). A user without Read permission on the Account object still sees "Insufficient Privileges" or blank fields even if the controller is `without sharing`. CRUD and FLS must be checked separately with `WITH USER_MODE`, `Security.stripInaccessible()`, or describe calls.

4. **Calling `stdController.getRecord()` and expecting all fields to be populated** — The StandardController lazily queries only fields referenced in the page markup. In the extension constructor, `getRecord()` returns a record with only the `Id` populated (and any fields the page references). If your extension needs fields not on the page, either add hidden `<apex:outputField>` tags to the page, call `stdController.addFields(fieldList)` before `getRecord()`, or query the fields yourself.

5. **Forgetting to assign the VF page to user profiles** — A perfectly working VF page returns "Insufficient Privileges" for any user whose profile (or permission set) does not have explicit access. Go to Setup → Visualforce Pages → [page name] → Security → move profiles to Enabled. This is separate from object permissions and is easy to overlook during deployment.

6. **Using `window.location` for navigation in Lightning Experience** — VF pages run in an iframe in Lightning Experience. Setting `window.location.href` navigates the iframe, not the outer Lightning shell. The user either sees a broken page or the VF page loads inside the iframe with no Lightning chrome. Always check for `sforce.one` and use its navigation methods (`navigateToSObject`, `navigateToURL`, `navigateToList`) when in Lightning context.

7. **Setting `escape="false"` on `<apex:outputText>` without sanitizing the value** — `escape="false"` disables HTML encoding, allowing raw HTML to render. If the value comes from a user-editable field (like a Description or Comments field), an attacker can inject `<script>` tags that execute in other users' browsers. Only use `escape="false"` for trusted, developer-controlled HTML. For user data, leave `escape="true"` (the default) or use `{!HTMLENCODE(value)}`.

8. **Performing DML in a GET-request page action without CSRF protection** — If your page has `action="{!doSomething}"` on `<apex:page>` and that method performs DML, any link to that page URL triggers the DML. An attacker can craft a link that, when clicked by an authenticated user, creates/updates/deletes records. Move state-changing logic to form submissions (POST) where the automatic CSRF token protects the action.

9. **Reinitializing `StandardSetController` on every page action in a pagination controller** — If your getter recreates the `StandardSetController` from a fresh query on every request, pagination state is lost. The user clicks "Next" but always sees page 1. Store the `StandardSetController` instance in a class variable and reuse it across requests. The ViewState preserves the instance between postbacks.

10. **Not wrapping SLDS-styled content in `<div class="slds-scope">` when using `<apex:slds>`** — Without the scoping div, SLDS styles can collide with standard Visualforce styles, causing visual glitches — misaligned buttons, wrong fonts, broken layouts. Always wrap your SLDS markup: `<div class="slds-scope"> ... your SLDS HTML ... </div>`.

## See Also

- [Visualforce PDF Generation](visualforce-pdf-generation.md) — rendering VF pages as PDFs with the Flying Saucer engine, `getContentAsPDF()`, and complete PDF template examples
- [Apex Fundamentals](../apex-fundamentals/apex-fundamentals.md) — sharing keywords, CRUD/FLS enforcement, governor limits, and class structure
- [Solution Architecture Patterns](../solution-architecture/solution-architecture-patterns.md) — when to use Visualforce vs LWC vs Flow for a given requirement
