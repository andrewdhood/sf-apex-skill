# Salesforce Apex & Solution Architecture Skill Builder

## Project Purpose

This project builds a Claude skill for Salesforce Apex development, solution architecture, and deployment — with a specific focus on PDF generation via Visualforce, email automation with PDF attachments, quick actions with screen flows, and the Apex patterns that tie them together. The skill teaches Claude to write correct, production-grade Apex, understand the platform's constraints, and design solutions that work end-to-end.

## Primary Use Case: Purchase Order PDF Workflow

The developer is building a Purchase Order system with this workflow:
1. A **Quick Action** on the Purchase Order custom object launches a **Screen Flow**
2. The screen flow lets the user **select a Contact** to send the PDF to
3. An **Apex invocable action** generates a PDF from a **Visualforce page** that renders the Purchase Order with custom fields
4. The PDF is **emailed** to the selected Contact as an attachment
5. After sending, the system **logs the action** (either as a completed Task or on a custom logging object)
6. If the PDF was emailed, the Purchase Order record is updated with **"Digital Authorization from {Authorizer_Name__c}"**

This workflow touches nearly every major Apex pattern: Visualforce rendering, PDF generation, email services, DML, field updates, quick actions, screen flows, and invocable methods.

## Current Phase: Content Research & Distillation

## Directory Structure

```
sf-apex-skill/
├── CLAUDE.md                   ← you are here
├── SKILL.md                    ← skill definition (trigger logic + instructions)
├── reference/                  ← distilled reference docs
│   ├── apex-fundamentals/      ← classes, triggers, governor limits, testing, sharing
│   ├── visualforce-pdf/        ← VF page rendering, PDF generation, styling, custom fields
│   ├── email-services/         ← SingleEmailMessage, attachments, org-wide addresses, templates
│   ├── quick-actions/          ← quick action types, screen flow actions, invocable methods
│   ├── deployment/             ← sf CLI, change sets, packaging, CI/CD, test coverage
│   ├── solution-architecture/  ← design patterns, object modeling, integration patterns
│   └── flows-and-automation/   ← flow + Apex interaction, invocable actions, platform events
├── templates/                  ← starter code templates
│   ├── visualforce-pdf/        ← VF page + controller for PDF rendering
│   ├── email-with-attachment/  ← Apex class for sending email with PDF attachment
│   ├── quick-action-screen-flow/ ← screen flow + invocable action pattern
│   ├── apex-trigger/           ← trigger + handler pattern
│   └── apex-test-class/        ← test class patterns
├── checklists/
├── exa-data/                   ← raw search results + processing log
└── scripts/
```

## Code Style Preferences

### Apex
- Comments with `// ` (one space after slashes)
- Disembodied narrator tone: `// begin sorting through accounts`, `// remember that we put xyz here for this reason`
- Every method gets param/output comments
- Debug statements at key points
- Never single-line conditionals (except ternary `?`)
- Consider governor limits and SOQL best practices
- Valid Apex, not Java
- Bulkified by default — all triggers and invocable methods must handle collections
- `with sharing` by default, `without sharing` only when explicitly justified

### Visualforce
- Use `renderAs="pdf"` for PDF generation
- CSS inline or in `<style>` blocks (external stylesheets don't reliably render in PDFs)
- Use `<apex:page>` with `standardController` or custom controller
- Always specify `applyBodyTag="false"` and `applyHtmlTag="false"` for full PDF layout control

### Test Classes
- Minimum 75% coverage, aim for 90%+
- Test bulk scenarios (200+ records)
- Use `@TestSetup` for shared test data
- Assert specific outcomes, not just "no exception thrown"
- Test negative scenarios and edge cases

## Key Domain Knowledge to Preserve

1. **PDF Generation via Visualforce**: `PageReference.getContentAsPDF()` renders a VF page as PDF in Apex. The VF page must be accessible to the running user. In test context, `getContentAsPDF()` returns a blob but doesn't actually render — you can't assert PDF content in tests, only that no exception was thrown.

2. **Email with PDF Attachment**: Use `Messaging.SingleEmailMessage` with `Messaging.EmailFileAttachment`. Set the attachment body to the PDF blob from `getContentAsPDF()`. Use org-wide email addresses for professional sender identity.

3. **Quick Action + Screen Flow Pattern**: Create a Quick Action of type "Flow" on the object. The screen flow receives the record ID automatically. The flow calls an `@InvocableMethod` Apex class to do the heavy lifting (PDF generation, email, logging).

4. **Digital Authorization Pattern**: After successful email send, update the Purchase Order record: `po.Digital_Authorization__c = 'Digital Authorization from ' + po.Authorizer_Name__c`. This should happen in the same transaction as the email send for consistency.

5. **Logging Pattern**: Either create a Task (standard, shows in activity timeline) or a custom `PO_Email_Log__c` object (more structured, queryable). Tasks are simpler but less flexible for reporting.

6. **Governor Limits in Email Context**: `Messaging.sendEmail()` counts against the 10 single email limit per transaction (or 5,000 per day org-wide). `getContentAsPDF()` counts as a callout in some contexts. Test methods can't verify actual email delivery.
