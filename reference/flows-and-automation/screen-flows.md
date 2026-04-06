# Screen Flows -- Apex Developer's Guide

> **When to read this:** When building a screen flow that collects user input, when connecting a screen flow to Apex via `@InvocableMethod`, when building a custom LWC screen component for a flow, or when deciding whether to build a screen flow vs a full custom LWC.

## Rules

- When building a screen flow, always set the run context explicitly (User, System with Sharing, or System without Sharing) because the default is User context, which respects the running user's object permissions, field-level security, and record sharing -- and that will silently hide fields or block DML if the user lacks access.
- Never put DML elements (Create Records, Update Records, Delete Records) inside a Loop element because each iteration counts against the 150 DML statement governor limit. Collect records into a collection variable inside the loop, then perform a single DML outside the loop.
- When using system context (with or without sharing), remember that certain screen components -- Lookup, Address, Dependent Picklist, File Upload, and Dynamic Forms for Flows -- still execute their own queries in the running user's context. System mode does not override their internal data access.
- Always add fault paths to every DML element and every Apex Action element in a screen flow because unhandled failures kill the flow with a generic "An unhandled fault has occurred" error page. Route faults to a dedicated error screen that explains the problem in plain language.
- When a screen flow calls an `@InvocableMethod`, always catch exceptions inside the Apex method and return error information in a Result wrapper rather than letting the exception propagate, because unhandled exceptions roll back the entire transaction and show the user a useless error message. See `reference/apex-fundamentals/invocable-methods.md` for the Request/Result pattern.
- Never enable the Previous button on screens that follow DML elements or Apex Actions that perform DML because going back re-executes elements between screens, causing duplicate operations. Disable Previous unless you have explicitly handled idempotency.
- When embedding a screen flow in Experience Cloud, always test as both a guest user and a community user separately because the guest user profile has its own restrictive object and field-level permissions.
- When building a custom LWC for a screen flow, always fire `FlowAttributeChangeEvent` to update output values instead of directly modifying `@api` properties, because the flow runtime must be notified of changes to maintain accurate state for reactivity and conditional visibility.
- When choosing between a screen flow and a custom LWC for user-facing processes, always default to screen flow unless you need custom UI beyond what flow screens offer (complex conditional rendering, drag-and-drop, animations, real-time external API calls). Flows are declarative, admin-maintainable, and do not require deployment.
- Always name screen flow elements with descriptive, action-oriented labels (e.g., `Collect_Contact_Info`, `Display_Confirmation`) because generic names like `Screen_1` make debugging and maintenance painful.

## How It Works

### Screen Flow Execution Model

Screen flows are interactive flows that present one or more screens to users, collecting input, displaying information, or guiding users through a business process. They pause at each Screen element and wait for user interaction before continuing. Between screens, the flow can execute logic elements (Decisions, Assignments, Loops), data elements (Get/Create/Update/Delete Records), and action elements (Apex Actions, Subflows, Send Email).

Each screen acts as a transaction boundary -- DML performed before a screen is committed before the screen renders. This means that if a user abandons the flow mid-way, any records created before the last screen are already committed to the database. This is critical for Apex developers to understand: your `@InvocableMethod` shares the calling flow's transaction, and if it runs between two screens, its DML commits when the next screen renders.

The flow runtime provides built-in navigation: Next (advance to the next screen), Previous (go back), Pause (save the interview for later), and Finish (complete the flow). The Finish button appears automatically on the last screen.

### Run Context: User vs System

Screen flows offer three run contexts, configured on the Start element:

| Context | Object/Field Permissions | Record Sharing | Use When |
|---------|-------------------------|----------------|----------|
| **User** (default) | Enforced | Enforced | Standard internal user processes |
| **System - With Sharing** | Bypassed | Enforced | User needs to see all fields but only their own records |
| **System - Without Sharing** | Bypassed | Bypassed | Guest users, system-level operations (validate input carefully) |

When the flow calls an `@InvocableMethod`, the Apex class's own sharing declaration (`with sharing` / `without sharing`) governs Apex-level data access. The flow's run context governs data access for flow elements (Get Records, Create Records, etc.) but does NOT override the Apex class's sharing keywords. These are two independent access control layers.

### Reactive Screens and Conditional Visibility

Reactive screens (API version 59.0+, GA since Spring '23) allow components on the same screen to react to each other in real time. When a user changes a value in one component, other components on the same screen can update their visibility, values, or options without requiring navigation to a new screen.

**How conditional visibility works:** Select any component on a screen, look for "Set Component Visibility" in the right panel, and add conditions referencing other components on the same screen, flow variables, or formulas. The component appears only when conditions evaluate to true.

**Dynamic visibility with custom LWC components:** When a custom LWC fires `FlowAttributeChangeEvent`, the flow runtime updates the attribute value and re-evaluates all conditional visibility rules on the same screen. This is why you must use `FlowAttributeChangeEvent` instead of directly setting `@api` properties -- the runtime needs the event to trigger reactivity.

**Spring '26 additions:** The new Message component supports dynamic display with configurable Message Type (Info, Success, Warning, Error) based on screen criteria. The new File Preview component can display supported formats (PNG, JPG, PDF) directly within the flow by passing a Content Document ID.

### How Screen Flows Call Apex

Screen flows call Apex through the **Apex Action** element, which invokes an `@InvocableMethod`. The flow batches multiple interviews into a single Apex call -- if a record-triggered flow fires for 200 records and each calls a subflow containing an Apex Action, the Apex method receives `List<Request>` with 200 elements.

The Apex method shares the calling flow's transaction and governor limits. If the flow already consumed 40 SOQL queries before calling Apex, you have 60 remaining. Key implications:

- DML in the Apex method commits when the next screen renders (not when the method returns)
- If the Apex method has `callout=true`, no DML can precede it in the same transaction -- so do not place Create/Update/Delete Records elements before the Apex Action
- Exceptions thrown by the Apex method trigger the fault path on the Apex Action element

See `reference/apex-fundamentals/invocable-methods.md` for the full `@InvocableMethod` pattern including the Request/Result inner class, bulkification, and error handling strategies.

## Screen Flow Distribution Methods

| Method | How to Set Up | Input Variables | Best For |
|--------|---------------|-----------------|----------|
| **Quick Action** | Setup > Object > Buttons, Links, Actions > New Action > Type: Flow | Automatic `recordId` from record | Record-context actions (Send PO Email, Approve, Validate) |
| **Lightning Record Page** | App Builder > drag Flow component > select flow > map `recordId` | Automatic `recordId` from page | Record-context wizards, data enrichment |
| **Experience Cloud Page** | Experience Builder > drag Flow component onto page | Map variables in component properties | Customer/partner-facing processes |
| **Custom LWC** | Use `<lightning-flow>` base component in your LWC | Pass via `inputVariables` property | Full control over flow lifecycle |
| **Direct URL** | `/flow/Flow_API_Name?var1=value1&retURL=/path` | Query string parameters | Email links, external buttons |
| **Utility Bar** | App Builder > Utility Items > add Flow | No automatic record context | App-wide tools (log a call, quick create) |

For the Purchase Order PDF workflow, **Quick Action** (type: Flow) on the Purchase_Order__c object is the correct distribution method. See `reference/quick-actions/quick-actions-and-screen-flows.md`.

## Custom LWC Screen Components

When standard flow screen components are not enough, you can build a custom Lightning Web Component that runs inside a flow screen. The component uses the `lightning__FlowScreen` target and communicates with the flow runtime via events.

### Meta XML Configuration

```xml
<?xml version="1.0" encoding="UTF-8"?>
<LightningComponentBundle xmlns="http://soap.sforce.com/2006/04/metadata">
    <apiVersion>62.0</apiVersion>
    <isExposed>true</isExposed>
    <targets>
        <target>lightning__FlowScreen</target>
    </targets>
    <targetConfigs>
        <targetConfig targets="lightning__FlowScreen">
            <!-- inputOnly: flow passes data TO the component -->
            <property name="recordId" type="String" label="Record ID" role="inputOnly" />
            <!-- outputOnly: component passes data BACK to the flow -->
            <property name="selectedContactId" type="String" label="Selected Contact" role="outputOnly" />
            <!-- no role: bidirectional (input and output) -->
            <property name="quantity" type="Integer" label="Quantity" default="1" />
        </targetConfig>
    </targetConfigs>
</LightningComponentBundle>
```

**Supported property types:** `String`, `Integer`, `Boolean`, `Date`, `DateTime`, `@salesforce/schema/ObjectApiName.FieldApiName` (sobject field references), `apex://ClassName.InnerClassName` (Apex-defined types for complex structured data).

### JavaScript: Events, Navigation, and Validation

```javascript
import { LightningElement, api } from 'lwc';
import { FlowAttributeChangeEvent, FlowNavigationNextEvent } from 'lightning/flowSupport';

export default class ContactSelector extends LightningElement {
    @api recordId;            // input from flow (the PO record ID)
    @api selectedContactId;   // output to flow

    // notify the flow runtime that an output value changed
    // CRITICAL: always use FlowAttributeChangeEvent -- never set @api props directly
    handleContactSelected(event) {
        this.selectedContactId = event.detail.value;
        const attributeChangeEvent = new FlowAttributeChangeEvent(
            'selectedContactId',       // must match the @api property name exactly
            this.selectedContactId     // the new value
        );
        this.dispatchEvent(attributeChangeEvent);
    }

    // programmatically navigate to the next screen
    handleNext() {
        const navigateNextEvent = new FlowNavigationNextEvent();
        this.dispatchEvent(navigateNextEvent);
    }

    // optional: validate before allowing navigation
    // the flow runtime calls this automatically when the user clicks Next/Finish
    @api
    validate() {
        if (!this.selectedContactId) {
            return {
                isValid: false,
                errorMessage: 'Please select a Contact before continuing.'
            };
        }
        return { isValid: true };
    }
}
```

**Available navigation events:**

| Event | Purpose |
|-------|---------|
| `FlowNavigationNextEvent` | Navigate to the next screen |
| `FlowNavigationBackEvent` | Navigate to the previous screen |
| `FlowNavigationFinishEvent` | Finish the flow |
| `FlowNavigationPauseEvent` | Pause the flow interview |

**Reactivity best practices for custom LWC components:**

- Always fire `FlowAttributeChangeEvent` to update output values -- never modify `@api` properties directly. The flow runtime needs the event to trigger conditional visibility and cross-component reactivity.
- Use the get/set pattern for `@api` properties when you need to react to changes from other components on the same screen.
- Never fire `FlowAttributeChangeEvent` and `FlowNavigationXxxEvent` at the same time -- this causes race conditions because the component may not have rendered updated values before navigation begins.

### Embedding a Flow in a Custom LWC

When you need full control over the flow lifecycle (custom launch buttons, post-flow logic, conditional rendering), embed the flow in an LWC using `<lightning-flow>`:

```html
<!-- parentComponent.html -->
<template>
    <lightning-flow
        flow-api-name="Send_Purchase_Order_Email"
        flow-input-variables={inputVariables}
        onstatuschange={handleStatusChange}>
    </lightning-flow>
</template>
```

```javascript
// parentComponent.js
import { LightningElement, api } from 'lwc';

export default class ParentComponent extends LightningElement {
    @api recordId;

    // input variables must be an array of { name, type, value } objects
    get inputVariables() {
        return [
            { name: 'recordId', type: 'String', value: this.recordId },
            { name: 'source', type: 'String', value: 'Lightning Page' }
        ];
    }

    handleStatusChange(event) {
        // event.detail.status: 'STARTED', 'PAUSED', 'FINISHED', 'ERROR'
        if (event.detail.status === 'FINISHED') {
            // access output variables from event.detail.outputVariables
            // each outputVariable has: { name, dataType, value }
            const outputVars = event.detail.outputVariables;
            const success = outputVars.find(v => v.name === 'isSuccess');
            if (success && success.value) {
                // refresh the record page, show toast, etc.
            }
        }
    }
}
```

## The Contact Selection Pattern (PO Workflow)

When the user needs to select a Contact in a screen flow (as in the Purchase Order email workflow), there are three approaches:

| Approach | How It Works | Best For |
|----------|-------------|----------|
| **Lookup component** | Type-ahead server-side search. Returns a Contact ID. | Large data volumes (hundreds/thousands of contacts). The PO workflow should use this. |
| **Record Choice Set + radio buttons** | Loads all matching records into memory. Displays as radio buttons or picklist. | Small bounded lists (5-20 contacts). Hits the 50,000 SOQL row limit at scale. |
| **Get Records + Choice** | Query contacts with a Get Records element filtered to the PO's Account, then bind results to a Choice component. | When you need explicit control over filter criteria and display columns. |

For the PO workflow, use a **Lookup component** filtered to Contacts related to the PO's Account. The Lookup component executes its search query in the running user's context (even if the flow is in System context), so the user must have Read access to Contact. Pass the selected Contact Id to the Apex Action's Request wrapper alongside the `recordId`.

## When to Build a Screen Flow vs a Custom LWC

This is the decision framework for Apex developers who need to choose between a declarative screen flow and a fully custom LWC.

### Use a Screen Flow When:

- **Admin maintainability matters.** Admins can modify screen flows without code deployment. If the business process changes frequently (new fields, different routing, updated validation), a flow absorbs changes in minutes.
- **The UI is a linear wizard.** Multi-step collect-confirm-execute-result patterns map perfectly to flow screens. The flow runtime handles navigation, transaction boundaries, and pause/resume for you.
- **You need to call Apex for heavy lifting.** The Quick Action > Screen Flow > `@InvocableMethod` pattern separates UI (flow) from business logic (Apex). The flow handles screens; Apex handles PDF generation, email, callouts, complex calculations.
- **Standard components suffice.** Text inputs, lookups, picklists, radio buttons, checkboxes, data tables, and the new Message and File Preview components (Spring '26) cover most data collection needs.
- **You want built-in features for free.** Pause/resume, Previous/Next navigation, conditional visibility, fault handling with `{!$Flow.FaultMessage}`, and system-context elevation are all built into the flow runtime.

### Use a Custom LWC When:

- **You need pixel-perfect UI.** Custom branding, animations, drag-and-drop, complex layout grids, or design patterns that go beyond what flow screen components can render.
- **Real-time external data is required.** If the screen needs to call an external API on every keystroke (address autocomplete, real-time pricing), a custom LWC with imperative Apex calls is more responsive than flow round-trips.
- **Complex client-side logic.** Multi-field cross-validation, computed previews, dynamic table manipulation, or state management that would require dozens of flow variables and Decision elements.
- **Performance-critical interactions.** LWC renders on the client with no server round-trip for simple state changes. Flow screens require a server round-trip for every navigation event.
- **You need to embed a third-party library.** Chart libraries, rich text editors, signature pads, or other JavaScript libraries cannot run inside flow screen components (but can run inside a custom LWC embedded in a flow via the `lightning__FlowScreen` target).

### Use the Hybrid Pattern When:

- **Most screens are standard, but one step needs custom UI.** Build the flow with standard screen components for most steps, and drop a custom LWC (`lightning__FlowScreen` target) onto the one screen that needs it. The LWC communicates with the flow via `FlowAttributeChangeEvent` and `@api` properties.
- **You want flow orchestration with LWC power.** The flow handles navigation, transaction boundaries, and Apex invocation. The custom LWC handles one rich interaction (a signature pad, a complex lookup with preview, a file upload with validation).

## Flow Builder Navigation

### Creating a New Screen Flow
Setup > Flows > New Flow > Screen Flow > Create
> (Optionally) set "Run flow as" in the Start element: User, System Context - With Sharing, or System Context - Without Sharing

### Adding a Screen Element
Click **+** on the canvas > Screen
> Drag components from the left panel onto the screen canvas
> Configure each component's properties in the right panel

### Adding an Apex Action
Click **+** on the canvas > Action
> Search for the `@InvocableMethod` label (e.g., "Send Purchase Order PDF")
> Map input variables from flow variables to the Request wrapper fields
> Map output variables from the Result wrapper fields to flow variables
> Wire a fault connector to an error screen as a safety net

### Setting Conditional Visibility
Select any component on a screen > look for "Set Component Visibility" in the right panel
> Choose "All Conditions Are Met" / "Any Condition Is Met" / "Custom Condition Logic Is Met"
> Add conditions referencing other components on the same screen, flow variables, or formulas
> The component appears only when conditions evaluate to true
> Requires API version 59.0+

### Debugging Screen Flows (Winter '26+)
Flow Builder > Debug button > the debug panel opens as a side panel (no longer a new tab)
> Enter test input values (cached in browser between sessions)
> Step through screens and see element-by-element execution details in cards
> Use the search bar to find keywords in the debug output
> Use "View on Canvas" to jump to any element
> The panel can expand to 80% of screen width; all details can be copied to clipboard

## Configuration Examples

### The Multi-Screen Pattern: Collect > Confirm > Execute > Result

This is the standard pattern for screen flows that call Apex, used in the PO PDF workflow:

```
[Screen 1: Collect]  User selects a Contact via Lookup component
        |
[Get Records]        Fetch PO details and Contact email for confirmation display
        |
[Screen 2: Confirm]  "Send PO-0001 PDF to Jane Doe (jane@example.com)?"
        |
[Apex Action]        SendPurchaseOrderPdfAction (callout=true for PDF generation)
        |
[Decision]           {!result.isSuccess}?
   |            |
   Yes          No
   |            |
[Screen 3a]    [Screen 3b]
 Success        Error: {!result.errorMessage}
```

**Critical:** The Apex Action runs between Screen 2 and Screen 3. Its DML commits when Screen 3 renders. If the user closes the browser after Screen 2's Next button but before Screen 3 loads, the Apex has already executed. Place irreversible actions (email sends, external callouts) after the user's final confirmation.

### Reactive Screen with Conditional Visibility

**Scenario:** Show different fields based on a picklist selection on the same screen.

**Screen Components:**
1. **Issue Type** (Picklist): choices = "Billing", "Technical", "General"
2. **Invoice Number** (Text): conditional visibility = `{!Issue_Type} Equals Billing`
3. **Error Message** (Long Text Area): conditional visibility = `{!Issue_Type} Equals Technical`
4. **Product Line** (Picklist): conditional visibility = `{!Issue_Type} Equals Technical`
5. **General Description** (Long Text Area): conditional visibility = `{!Issue_Type} Equals General`

The user selects an Issue Type and the relevant fields appear instantly on the same screen without navigating away. Requires API version 59.0+.

### Custom LWC with Validation in a Screen Flow

A custom LWC that validates user input before allowing the flow to advance:

```javascript
import { LightningElement, api, wire } from 'lwc';
import { FlowAttributeChangeEvent } from 'lightning/flowSupport';
import getRelatedContacts from '@salesforce/apex/ContactLookupController.getRelatedContacts';

export default class EnhancedContactPicker extends LightningElement {
    @api accountId;             // input from flow
    @api selectedContactId;     // output to flow
    @api selectedContactEmail;  // output to flow

    contacts = [];
    error;

    // wire to Apex to load contacts related to the account
    @wire(getRelatedContacts, { accountId: '$accountId' })
    wiredContacts({ error, data }) {
        if (data) {
            this.contacts = data.map(c => ({
                label: c.FirstName + ' ' + c.LastName + ' (' + c.Email + ')',
                value: c.Id,
                email: c.Email
            }));
            this.error = undefined;
        } else if (error) {
            this.error = error.body.message;
            this.contacts = [];
        }
    }

    handleSelection(event) {
        const contactId = event.detail.value;
        this.selectedContactId = contactId;

        // look up the email for the selected contact
        const selected = this.contacts.find(c => c.value === contactId);
        this.selectedContactEmail = selected ? selected.email : null;

        // notify the flow runtime of both changes
        this.dispatchEvent(new FlowAttributeChangeEvent('selectedContactId', contactId));
        this.dispatchEvent(new FlowAttributeChangeEvent('selectedContactEmail', this.selectedContactEmail));
    }

    // the flow runtime calls validate() when the user clicks Next/Finish
    @api
    validate() {
        if (!this.selectedContactId) {
            return {
                isValid: false,
                errorMessage: 'Please select a Contact to send the PO to.'
            };
        }
        if (!this.selectedContactEmail) {
            return {
                isValid: false,
                errorMessage: 'The selected Contact does not have an email address.'
            };
        }
        return { isValid: true };
    }
}
```

## Common Mistakes

1. **Running screen flows in User context when guest users need access** -- Guest user profiles have minimal permissions by default. The flow fails silently or returns no data. Fix: Either grant object/field permissions to the guest user profile, or run the flow in System Context - Without Sharing (and validate all input carefully to prevent data exposure).

2. **Letting Apex exceptions propagate to the flow fault path** -- Unhandled exceptions from `@InvocableMethod` roll back the entire transaction and show "An unhandled fault has occurred" with a raw stack trace. Fix: Catch exceptions inside the Apex method and return error information in the Result wrapper. The flow checks `{!result.isSuccess}` and branches to a friendly error screen. Wire a fault connector anyway as a safety net for uncatchable `LimitException`.

3. **Forgetting that screen flow transactions commit at each screen boundary** -- Developers assume the entire flow is one transaction that rolls back on failure. In reality, DML before a screen is committed before the screen renders. If a user abandons the flow after Screen 2, records created before Screen 2 persist. Fix: Design screen flows so that all DML occurs after the last data-collection screen. Use the Collect > Confirm > Execute > Result pattern.

4. **Placing DML flow elements before an Apex Action with callout=true** -- If the `@InvocableMethod` has `callout=true` (e.g., for `getContentAsPDF()` or HTTP callouts), any DML in the same transaction before the callout causes `System.CalloutException: You have uncommitted work pending`. Fix: Place all DML-based flow elements (Create/Update/Delete Records) AFTER the Apex Action, not before it. The Apex Action must execute before any DML in the transaction.

5. **Directly modifying @api properties in a custom flow LWC instead of firing FlowAttributeChangeEvent** -- The flow runtime does not detect direct property modifications. Conditional visibility on other components will not re-evaluate, and the flow may not pick up the updated value on navigation. Fix: Always dispatch `FlowAttributeChangeEvent` with the property name and new value. Then set the local property.

6. **Firing FlowAttributeChangeEvent and FlowNavigationNextEvent simultaneously** -- This causes a race condition. The navigation may start before the attribute change is processed, meaning the flow advances with stale values. Fix: Fire the attribute change event first, then wait for the next render cycle (e.g., use `setTimeout` or `Promise.resolve().then()`) before dispatching the navigation event.

7. **Using a Record Choice Set for Contact selection in large orgs** -- Record Choice Sets load all matching records into memory upfront. If the Account has hundreds of Contacts, this wastes SOQL rows and memory. At 50,000+ matching records, it hits the SOQL row limit. Fix: Use the Lookup component for large data volumes. It supports type-ahead server-side search.

8. **Assuming system context bypasses everything in screen flows** -- Lookup components, Address components, Dependent Picklists, File Upload, and Dynamic Forms for Flows still query data using the running user's permissions, even when the flow is set to System Context. Fix: Test these components as the actual end user. If the user lacks Read access to the Lookup object, the component returns no results regardless of the flow's run context.

9. **Not testing reactive screens in the correct API version** -- Conditional visibility reactivity requires API version 59.0+. Flows created in older API versions do not support same-screen reactivity. Fix: Check the flow's API version in flow properties and update to 59.0 or later.

10. **Building a full custom LWC when a screen flow with one custom component would suffice** -- Developers default to building everything in LWC because they are comfortable with code. This creates a maintenance burden (deployments, test coverage, version management) for what could be a declarative flow with zero Apex. Fix: Use the decision framework above. If 80% of screens are standard input collection, build it as a flow and drop a custom LWC onto the one screen that needs it.

## See Also

- [Flow + Apex Coexistence Patterns](./flow-apex-coexistence.md) -- order of execution, error handling, automation density
- [Invocable Methods](../apex-fundamentals/invocable-methods.md) -- the @InvocableMethod pattern, bulkification, callout=true
- [Quick Actions & Screen Flows](../quick-actions/quick-actions-and-screen-flows.md) -- quick action setup, recordId passing
- [Purchase Order PDF Workflow](../solution-architecture/purchase-order-pdf-workflow.md) -- end-to-end working example
- [Scheduled & Platform Event Flows](./scheduled-and-platform-event-flows.md) -- async flow patterns and Apex alternatives
