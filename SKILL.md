---
name: sf-apex-development
description: >
  Write production-grade Salesforce Apex code, design solution architecture, and build
  end-to-end automation patterns. Use this skill whenever the user mentions Apex class,
  Apex trigger, Visualforce, PDF generation, renderAs PDF, getContentAsPDF, email
  attachment, SingleEmailMessage, quick action, invocable method, InvocableMethod,
  InvocableVariable, Apex deployment, sf project deploy, test coverage, governor limits,
  SOQL, DML, sharing keywords, with sharing, without sharing, trigger handler pattern,
  service layer, Apex testing, @isTest, Test.startTest, Queueable, Batch Apex,
  Schedulable, @future, ContentVersion, ContentDocumentLink, Messaging.sendEmail,
  EmailFileAttachment, org-wide email address, Approval.ProcessSubmitRequest,
  AuraHandledException, Platform Events from Apex, or any Salesforce server-side
  development. Also trigger when the user asks to generate a PDF, email a PDF,
  create a quick action that calls Apex, build a purchase order workflow, stamp
  a digital authorization, or design a Salesforce solution architecture. Even if
  the user just says "write an Apex class" or "help with my trigger," use this skill.
---

# Salesforce Apex & Solution Architecture

Write production-grade Apex code with correct governor limit awareness, proper
bulkification, sharing model compliance, and comprehensive test coverage. Design
end-to-end solutions combining Apex, Visualforce, Flows, and Quick Actions.

## Before Writing Any Apex

1. **Determine the sharing context.** Use `with sharing` by default. Only use
   `without sharing` when explicitly justified (e.g., guest user data access,
   system-level operations). Use `inherited sharing` for utility classes.
   Read `reference/apex-fundamentals/apex-fundamentals.md`.

2. **Check governor limits.** Know the transaction limits before designing:

   | Limit | Number |
   |---|---|
   | SOQL queries | 100 (200 async) |
   | DML statements | 150 |
   | Records retrieved (SOQL) | 50,000 |
   | Records processed (DML) | 10,000 |
   | CPU time | 10,000 ms (60,000 ms async) |
   | Heap size | 6 MB (12 MB async) |
   | Callouts | 100 |
   | Single emails per transaction | 10 |
   | Single emails per day (org) | 5,000 |

3. **Plan the architecture.** For anything beyond a simple trigger or class:
   Read `reference/solution-architecture/solution-architecture-patterns.md`.

## PDF Generation via Visualforce

When generating PDFs from Salesforce:

1. Create a Visualforce page with `renderAs="pdf"` and use `applyBodyTag="false"`
   and `applyHtmlTag="false"` for full layout control
2. Use **inline CSS only** — external stylesheets don't reliably render in PDFs
3. Generate the PDF blob in Apex via `PageReference.getContentAsPDF()`
4. `getContentAsPDF()` counts as a callout — no DML before it in the same transaction
5. Save as `ContentVersion` + `ContentDocumentLink` to attach to a record

Read `reference/visualforce-pdf/visualforce-pdf-generation.md` for complete code.

## Email with PDF Attachment

When sending emails with PDF attachments:

1. Use `Messaging.SingleEmailMessage` with `Messaging.EmailFileAttachment`
2. Use `setTargetObjectId()` (Contact/Lead ID) for merge fields and activity logging,
   NOT `setToAddresses()` (which doesn't log activities)
3. Use org-wide email addresses for professional sender identity
4. Always check `Messaging.SendEmailResult` — `sendEmail()` doesn't throw on failure
5. Limit: 10 SingleEmailMessage per transaction, 5,000 per day per org

Read `reference/email-services/email-with-attachments.md` for complete code.

## Quick Action → Screen Flow → Apex Pattern

The standard pattern for user-initiated Apex actions:

1. **Quick Action** (type: Flow) on the custom object launches a Screen Flow
2. **Screen Flow** receives `recordId` automatically, collects user input
3. **@InvocableMethod** Apex class does the heavy lifting
4. Flow displays success/error to the user

This pattern separates UI (Flow) from business logic (Apex) and is the recommended
approach for actions like "Send PO Email," "Generate Invoice," etc.

Read `reference/quick-actions/quick-actions-and-screen-flows.md` for quick action setup
and `reference/apex-fundamentals/invocable-methods.md` for the Apex side.

## Purchase Order PDF Workflow

The complete end-to-end pattern for the PO email workflow:

```
Quick Action → Screen Flow (select Contact) → Invocable Apex:
  1. Query Purchase Order + line items
  2. Generate PDF via PageReference.getContentAsPDF()
  3. Create SingleEmailMessage with PDF attachment
  4. Send email to selected Contact
  5. Log as Task (setSaveAsActivity=true)
  6. Stamp PO: "Digital Authorization from {Authorizer_Name__c}"
```

Read `reference/solution-architecture/purchase-order-pdf-workflow.md` for complete
working code for every component.

## Invocable Methods

When calling Apex from Flow:

- Use `@InvocableMethod` with `List<Request>` input and `List<Result>` output
- Always use the inner class pattern (`Request` / `Result` with `@InvocableVariable`)
- The method MUST handle `List<>` — Flow batches multiple interviews into one call
- Invocable methods share the calling flow's governor limits (same transaction)
- Use `callout=true` if the method makes HTTP callouts or calls `getContentAsPDF()`

Read `reference/apex-fundamentals/invocable-methods.md`.

## Apex Testing

- Minimum 75% coverage required for production deployment, aim for 90%+
- Use `@TestSetup` for shared test data creation
- Use `Test.startTest()` / `Test.stopTest()` to reset governor limits
- Assert specific outcomes: `System.assertEquals(expected, actual, 'message')`
- Test bulk scenarios (200+ records)
- `getContentAsPDF()` returns empty blob in tests — assert no exception, not content
- `Messaging.sendEmail()` in tests doesn't deliver — check `SendEmailResult` only

## Code Style

When generating Apex for this developer:

- **Comments**: `// ` with one space, disembodied narrator tone
- **Every method**: document params and return value
- **Debug statements**: at key decision points
- **No single-line conditionals** except ternary `?`
- **Sharing**: `with sharing` by default, document why if `without sharing`
- **Bulkification**: all triggers and invocable methods handle collections
- **Variable names**: meaningful in context, not `temp` or `data`
- **Valid Apex**: not Java. `System.debug()` not `System.out.println()`

## Reference Docs

Read the relevant reference doc BEFORE generating code:

| Topic | File |
|---|---|
| Apex Fundamentals (sharing, limits, triggers, testing, async) | `reference/apex-fundamentals/apex-fundamentals.md` |
| Invocable Methods (Flow-to-Apex bridge) | `reference/apex-fundamentals/invocable-methods.md` |
| Visualforce PDF Generation | `reference/visualforce-pdf/visualforce-pdf-generation.md` |
| Email Services with Attachments | `reference/email-services/email-with-attachments.md` |
| Quick Actions & Screen Flows | `reference/quick-actions/quick-actions-and-screen-flows.md` |
| Solution Architecture Patterns | `reference/solution-architecture/solution-architecture-patterns.md` |
| Purchase Order PDF Workflow (end-to-end) | `reference/solution-architecture/purchase-order-pdf-workflow.md` |
| Apex Deployment & CI/CD | `reference/deployment/apex-deployment.md` |

## Checklists

| Checklist | File |
|---|---|
| (coming soon) | |
