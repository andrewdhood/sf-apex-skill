# Quick Actions & Screen Flow Integration

> **When to read this:** You are creating a button on a record page that launches a screen flow, or you need to understand how quick actions, screen flows, and Lightning page layouts connect.

## Rules

- When creating a quick action that launches a flow, always use the **Flow** action type -- not a Visualforce page wrapping a flow, not an Aura component embedding a flow. The native Flow action type handles `recordId` passing, modal sizing, and flow finish behavior automatically.
- When passing the current record's ID to a flow from a quick action, always create a text variable named exactly `recordId` (capital I, lowercase d) and mark it **Available for Input**. Salesforce populates this automatically -- never set a default value on it.
- Never hardcode record IDs or assume the flow knows which record it was launched from without the `recordId` input variable. The flow has no implicit record context -- it only knows what you explicitly pass in.
- When choosing between a Flow quick action and an LWC quick action, always default to Flow unless you need custom UI beyond what flow screens offer (drag-and-drop components, animations, complex conditional rendering). Flows are declarative, admin-maintainable, and do not require deployment.
- When adding quick actions to a record page, always check whether **Dynamic Actions** is enabled on that page in Lightning App Builder. If Dynamic Actions is enabled, the page layout's publisher layout is ignored -- actions must be configured in Lightning App Builder instead.
- Never add a quick action only to the page layout's "Salesforce Mobile and Lightning Experience Actions" section and assume it will appear. If the Lightning record page uses Dynamic Actions, those override the page layout entirely for the highlights panel.
- When designing a screen flow launched from a quick action, always keep screens minimal. The flow renders in a modal -- horizontal scrolling and deeply nested layouts render poorly. One purpose per screen.
- Always assign the quick action to the correct page layout OR Lightning App Builder page. Creating the quick action alone does not make it visible to users.
- When selecting a Contact in a screen flow, always use the **Lookup** component (not a Record Choice Set with radio buttons) when the user could be searching across hundreds or thousands of records. Lookup supports type-ahead search. Record Choice Sets load all matching records into memory upfront.
- Never rely on the flow's finish behavior to close the quick action modal automatically in all contexts. Always test the finish behavior in both desktop and mobile, as behavior can differ between Lightning Experience and the Salesforce mobile app.

## How It Works

### Quick Action Types

Salesforce supports several quick action types, each suited to different use cases:

| Type | What It Does | When to Use |
|------|-------------|-------------|
| **Flow** | Launches a screen flow in a modal | Multi-step processes, guided wizards, calling Apex via invocable actions |
| **LWC** | Renders a Lightning Web Component in a modal | Custom UI beyond flow capabilities, complex interactions, real-time validation |
| **Visualforce** | Renders a Visualforce page in a modal | Legacy pages, PDF previews, complex HTML rendering (avoid for new development) |
| **Create a Record** | Opens a pre-filled record creation form | Simple record creation with default values |
| **Update a Record** | Opens a field update form | Quick field edits on the current record |
| **Log a Call** | Opens the Log a Call form | Activity tracking (calls, meetings) |
| **Send Email** | Opens the email composer | Sending email from a record |

For the Purchase Order PDF workflow, **Flow** is the correct type. The flow handles the user interaction (Contact selection), then delegates to Apex (PDF generation, email) via an `@InvocableMethod`.

### Quick Action Scope: Global vs Object-Specific

**Object-specific quick actions** are created on a particular object (e.g., Purchase_Order__c) and have automatic relationship context. They appear on that object's record pages and can access the record ID.

**Global quick actions** live on the global publisher layout, appear in the global actions menu (top navigation bar), and have no record context. They cannot pass `recordId` to a flow.

For the Purchase Order use case, always use an object-specific quick action on the Purchase_Order__c object.

### How recordId Gets Into the Flow

When a user clicks a flow-type quick action on a record page:

1. Salesforce opens the flow in a modal overlay
2. Salesforce looks for a flow input variable named exactly `recordId` (type: Text, Available for Input)
3. If found, Salesforce automatically sets it to the current record's 18-character ID
4. The flow starts its first screen with `recordId` populated

There is no configuration required on the quick action side -- the name `recordId` is a convention that Salesforce recognizes automatically. If you name the variable anything else (`record_id`, `RecordId`, `poId`), it will not be populated.

### Screen Flow Design for Quick Actions

A screen flow launched from a quick action runs in a modal window. Key design considerations:

**Contact Selection** -- Use a **Lookup** screen component, not a Record Choice Set:
- **Lookup component**: Type-ahead search, server-side filtering, handles large data volumes. The user sees a standard Salesforce lookup field. Returns the selected record's ID.
- **Record Choice Set with radio buttons/picklist**: Loads all matching records in a single query upfront. Works for small, bounded lists (e.g., 5-20 contacts on an account). Fails at scale.
- **Choice Lookup component**: Hybrid approach -- combines lookup-style search with choice set filtering. Useful when you need pre-filtered results with search capability.

For the Purchase Order flow, a Lookup component filtered to Contacts is the right choice unless you specifically want to limit selection to Contacts related to the PO's Account.

**Flow finish behavior**: When the flow finishes (reaches the end or a screen with no Next button), the modal closes. In Lightning Experience, the record page typically refreshes. In the Salesforce mobile app, behavior may vary -- test both.

### Adding Quick Actions to Record Pages

There are two distinct systems for controlling which actions appear on a record page:

**Publisher Layout (Page Layout Editor):**
- Configured in Setup > Object Manager > [Object] > Page Layouts
- The "Salesforce Mobile and Lightning Experience Actions" section controls actions in the highlights panel
- Actions appear in the order they are arranged in the layout editor
- This is the traditional approach and still works if Dynamic Actions is not enabled

**Dynamic Actions (Lightning App Builder):**
- Enabled per-page in Lightning App Builder by selecting the Highlights Panel component and toggling "Enable Dynamic Actions"
- Once enabled, the page layout's action section is **completely ignored** for that page
- Actions are added/removed/reordered directly in Lightning App Builder
- Each action can have **visibility filters** (e.g., show "Send PO PDF" only when Status = "Approved")
- Supports conditional visibility based on record field values, user profile, permissions, and more

**When to use which:**
- Use **Dynamic Actions** when you need conditional action visibility (e.g., only show the "Send PDF" action when the PO is in "Approved" status), or when different user profiles should see different actions.
- Use **Publisher Layout** when all users should see the same actions unconditionally, or when you want simpler administration.

Dynamic Actions is the modern approach and is generally preferred for new development. It is available for custom objects and most standard objects.

### Mobile vs Desktop Behavior

- Both desktop (Lightning Experience) and mobile (Salesforce mobile app) read from the same "Salesforce Mobile and Lightning Experience Actions" section of the page layout (unless Dynamic Actions overrides it).
- On desktop, quick actions appear in the highlights panel at the top of the record page, in a dropdown menu if there are too many.
- On mobile, quick actions appear in the action bar at the bottom of the record page. The first few are visible; the rest are behind a "more" menu.
- Flow-type quick actions render the flow in a modal on both platforms, but the modal sizing and scroll behavior differ. Test your flow screens on both.
- LWC-based quick actions are **not supported** on the Salesforce mobile app (as of API version 62.0). If mobile is a requirement, use a Flow quick action instead.

### Flow vs LWC Quick Actions: Decision Guide

| Criterion | Flow Quick Action | LWC Quick Action |
|-----------|------------------|-----------------|
| Custom UI needed | No -- flow screens suffice | Yes -- need pixel-perfect control |
| Admin-maintainable | Yes -- admins can modify flows | No -- requires developer deployment |
| Mobile support | Yes | No (not on Salesforce mobile app) |
| Apex integration | Via @InvocableMethod | Via @AuraEnabled |
| Modal behavior | Automatic open/close/navigation | Manual -- you control everything |
| Complex validation | Limited to flow validation rules | Full JavaScript validation |
| Development speed | Faster (declarative) | Slower (code + deploy) |

**Default to Flow.** Only use LWC when the flow screen components genuinely cannot support the required UX.

## Code Examples

### Quick Action Metadata: Flow Type (quickAction-meta.xml)

Object-specific quick actions on custom objects live in the object's metadata directory. The file is named `[ObjectApiName].[ActionName].quickAction-meta.xml`.

For a Purchase Order object with an action called "Send_PDF":

```xml
<?xml version="1.0" encoding="UTF-8"?>
<QuickAction xmlns="http://soap.sforce.com/2006/04/metadata">
    <flowDefinition>Send_Purchase_Order_PDF</flowDefinition>
    <label>Send PO PDF</label>
    <optionsCreateFeedItem>false</optionsCreateFeedItem>
    <type>Flow</type>
</QuickAction>
```

File path in SFDX project:
```
force-app/main/default/objects/Purchase_Order__c/quickActions/Send_PDF.quickAction-meta.xml
```

Key fields:
- `type`: Must be `Flow` for flow-type quick actions
- `flowDefinition`: The API name of the screen flow (not the flow version, just the flow name)
- `label`: What the user sees on the button
- `optionsCreateFeedItem`: Set to `false` unless you want a Chatter feed item every time the action runs

### Quick Action Metadata: LWC Type (for comparison)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<QuickAction xmlns="http://soap.sforce.com/2006/04/metadata">
    <actionSubtype>ScreenAction</actionSubtype>
    <label>Send PO PDF</label>
    <lightningWebComponent>sendPurchaseOrderPdf</lightningWebComponent>
    <optionsCreateFeedItem>false</optionsCreateFeedItem>
    <type>LightningWebComponent</type>
</QuickAction>
```

### Quick Action Metadata: Create Record Type (for comparison)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<QuickAction xmlns="http://soap.sforce.com/2006/04/metadata">
    <label>New Contact</label>
    <optionsCreateFeedItem>true</optionsCreateFeedItem>
    <targetObject>Contact</targetObject>
    <type>Create</type>
</QuickAction>
```

### Screen Flow: recordId Input Variable Setup

In Flow Builder, create the input variable:

1. Open Flow Builder for your screen flow
2. Click **Manager** tab (left panel) > **New Resource**
3. Resource Type: **Variable**
4. API Name: `recordId` (exact casing required)
5. Data Type: **Text**
6. Check **Available for Input**
7. Leave **Available for Output** unchecked
8. Leave **Default Value** blank

The flow can then reference `{!recordId}` anywhere -- in Get Records filters, in Apex Action inputs, in Assignment elements, etc.

### Screen Flow: Contact Lookup Screen

On your flow screen, add a **Lookup** component:

- **API Name**: `contactLookup`
- **Object API Name**: `Contact`
- **Label**: `Select Recipient`
- **Required**: checked
- **Field API Name**: `Id` (the lookup returns the Contact record)

To filter Contacts to only those related to the PO's Account:

1. Add a **Get Records** element before the screen:
   - Object: `Purchase_Order__c`
   - Filter: `Id` equals `{!recordId}`
   - Store: first record in `{!purchaseOrderRecord}`
2. On the Lookup component, add a filter:
   - Field: `AccountId`
   - Operator: Equals
   - Value: `{!purchaseOrderRecord.Account__c}`

### Page Layout XML: Adding a Quick Action

In the page layout metadata, quick actions appear in the `platformActionList` section:

```xml
<Layout xmlns="http://soap.sforce.com/2006/04/metadata">
    <!-- ... other layout sections ... -->
    <platformActionList>
        <actionListContext>Record</actionListContext>
        <platformActionListItems>
            <actionName>Send_PDF</actionName>
            <actionType>QuickAction</actionType>
            <sortOrder>0</sortOrder>
        </platformActionListItems>
        <platformActionListItems>
            <actionName>Edit</actionName>
            <actionType>StandardButton</actionType>
            <sortOrder>1</sortOrder>
        </platformActionListItems>
        <platformActionListItems>
            <actionName>FeedItem.TextPost</actionName>
            <actionType>QuickAction</actionType>
            <sortOrder>2</sortOrder>
        </platformActionListItems>
    </platformActionList>
</Layout>
```

### SFDX Deployment: Retrieving and Deploying Quick Actions

```bash
# retrieve the quick action
sf project retrieve start --metadata QuickAction:Purchase_Order__c.Send_PDF

# retrieve the page layout that includes the action assignment
sf project retrieve start --metadata Layout:Purchase_Order__c-Purchase\ Order\ Layout

# deploy both together
sf project deploy start --source-dir force-app/main/default/objects/Purchase_Order__c/quickActions \
  --source-dir force-app/main/default/layouts
```

## Common Mistakes

1. **Naming the flow variable `RecordId` or `record_id` instead of `recordId`** -- The variable name is case-sensitive and must be exactly `recordId` with a capital I. Salesforce will not populate any other variation. Fix: Rename the variable to `recordId` in Flow Builder.

2. **Creating the quick action but never adding it to the page layout or Lightning App Builder** -- The action exists in metadata but is invisible to users. Fix: Add it to the "Salesforce Mobile and Lightning Experience Actions" section of the page layout, or add it in Lightning App Builder if Dynamic Actions is enabled.

3. **Adding the action to the page layout when Dynamic Actions is enabled** -- Dynamic Actions completely overrides the page layout's action section. Changes to the page layout have zero effect. Fix: Open Lightning App Builder, select the Highlights Panel, and add the action there.

4. **Using a Record Choice Set for Contact selection when there could be thousands of Contacts** -- Record Choice Sets load all matching records into a collection in memory. With large data volumes, this causes performance problems or hits the 50,000-record SOQL query limit. Fix: Use a Lookup screen component instead, which performs server-side search.

5. **Forgetting to mark the `recordId` variable as Available for Input** -- The variable exists but Salesforce cannot write to it, so it stays null. Every Get Records element using `{!recordId}` returns nothing. Fix: Edit the variable and check "Available for Input".

6. **Setting a default value on the `recordId` variable** -- If you set a default (e.g., a hardcoded ID for testing), it will override the automatically-passed value in some contexts. Fix: Always leave the default value blank.

7. **Assuming the flow modal will auto-close and refresh the page in all contexts** -- On desktop Lightning Experience, the modal closes and the page refreshes when the flow finishes. On mobile, the refresh behavior is inconsistent. In embedded components or communities, it may not refresh at all. Fix: Test in every context where users will use the action. Add a confirmation screen as the last flow screen so users know the action completed.

8. **Using an LWC quick action when mobile support is required** -- LWC-based quick actions do not work on the Salesforce mobile app. Users on mobile will not see the action at all. Fix: Use a Flow quick action instead, which works on both desktop and mobile.

9. **Referencing the flow by version name in the quick action metadata** -- The `flowDefinition` field takes the flow's API name (e.g., `Send_Purchase_Order_PDF`), not a versioned name. Salesforce automatically runs the active version. Fix: Use just the flow API name.

10. **Building the flow without considering what happens if the user clicks the X to close the modal** -- If the user dismisses the modal mid-flow, no DML has occurred (unless an autolaunched subflow already ran). But if you have a flow that calls an Apex action on screen 2 of 3, and the user closes after screen 2, the Apex action already executed. Fix: Put destructive or irreversible actions (email sends, external callouts) on the final screen or after the final screen's Next button click. Use a confirmation screen before the action.

## See Also

- [Invocable Methods -- Flow-to-Apex Bridge](../apex-fundamentals/invocable-methods.md)
- [Visualforce PDF Generation](../visualforce-pdf/) (if available)
- [Email Services](../email-services/) (if available)
