# Apex Email Patterns

> **When to read this:** You need to send or receive email from Apex, use email templates programmatically, track email activity on records, handle inbound email processing, or understand the full landscape of email governor limits and testing patterns.

## Rules

### Outbound Email

- When sending an email to a Contact, Lead, or User, always use `setTargetObjectId()` instead of `setToAddresses()` because `setTargetObjectId()` enables merge field resolution in templates, creates an activity record when `setSaveAsActivity(true)`, and respects the recipient's email opt-out preferences.
- Never use both `setTargetObjectId()` and `setToAddresses()` for the same recipient because the recipient will receive duplicate copies of the email.
- When sending to an arbitrary email address that is not a Contact, Lead, or User record, use `setToAddresses()` and accept that you lose activity logging, merge field resolution, and opt-out enforcement.
- Always use `setOrgWideEmailAddressId()` for any user-facing email because without it the "From" address is the running user's email, which looks unprofessional, varies by context, and causes deliverability issues with SPF/DKIM. Query the `OrgWideEmailAddress` record by `Address` or `DisplayName` -- never hardcode the Id because Ids differ across environments.
- When building the email body in Apex (not using a template), always set BOTH `setHtmlBody()` and `setPlainTextBody()` because some email clients only render plain text, and spam filters flag HTML-only emails that lack a plain text alternative.
- When using an email template, never also set `setHtmlBody()` or `setPlainTextBody()` because `setTemplateId()` silently overrides any programmatic body -- your code runs without error but the custom body is discarded.
- When you need merge fields in a programmatic body without a stored template, use `setTreatBodiesAsTemplate(true)` with `setTargetObjectId()` and optionally `setWhatId()` because this lets you embed `{!Contact.FirstName}` syntax directly in your `setHtmlBody()` string and Salesforce resolves them at send time.
- Always batch all email messages into a single `List<Messaging.SingleEmailMessage>` and make ONE `Messaging.sendEmail()` call per transaction because the governor limit is 10 `sendEmail()` invocations per transaction, not 10 messages -- a single call with 10 messages in the list uses only 1 invocation.
- Never call `Messaging.sendEmail()` inside a loop because each call consumes one of your 10 invocations per transaction, and you will hit the limit with as few as 11 iterations.
- Always check `Messaging.SendEmailResult.isSuccess()` after calling `sendEmail()` because the method does not throw an exception for all failure modes -- invalid template IDs, missing required fields, and some address issues silently return `isSuccess() = false`.
- When using `MassEmailMessage`, always use `setTargetObjectIds()` (plural) with Contact or Lead IDs and always provide `setTemplateId()` because `MassEmailMessage` requires a template and does not support programmatic body -- there is no `setHtmlBody()` on `MassEmailMessage`.
- Never use `setOrgWideEmailAddressId()` and `setSenderDisplayName()` on the same message because `setOrgWideEmailAddressId()` overrides `setSenderDisplayName()` and `setReplyTo()` -- the org-wide address controls both the From name and reply-to.

### Inbound Email

- When implementing `Messaging.InboundEmailHandler`, always return a `Messaging.InboundEmailResult` with `success = true` even if your business logic encounters a recoverable error, because returning `success = false` causes Salesforce to send a bounce reply to the sender, which is rarely what you want for a valid email that just couldn't be processed.
- Always wrap the body of `handleInboundEmail()` in a try-catch because an unhandled exception causes Salesforce to send a system bounce message to the sender, exposing internal error details.
- Never trust the `fromAddress` on inbound email for authentication decisions without additional validation because email "From" headers are trivially spoofed -- validate against known addresses, check for specific subject line tokens, or use other authentication mechanisms.
- When parsing inbound email attachments, always check both `binaryAttachments` and `textAttachments` for null before iterating because Salesforce returns null (not an empty list) when there are no attachments of that type.

### Activity Tracking

- Always call `setSaveAsActivity(true)` when sending to a Contact or Lead because this creates an EmailMessage record (with Enhanced Email) or Task record (without) in the activity timeline, giving users visibility into automated emails.
- When using `setSaveAsActivity(true)`, always also call `setWhatId()` with the related record Id because this links the activity to both the Contact AND the related record so the email appears in both activity timelines.
- Never use `setSaveAsActivity(true)` when `setTargetObjectId()` points to a User Id because Salesforce throws an error -- activity logging is only supported for Contacts and Leads, not internal Users.
- When querying sent email history, always query the `EmailMessage` object (not `Task`) in orgs with Enhanced Email because Enhanced Email stores emails as `EmailMessage` records with rich tracking fields (`IsOpened`, `IsBounced`, `FirstOpenedDate`, `LastOpenedDate`, `Status`).

### Governor Limits

- Never exceed 10 `Messaging.sendEmail()` invocations per Apex transaction because this is a hard governor limit -- even if each call sends only one message, the 11th call fails.
- Always be aware of the daily org-wide limit of 5,000 single emails per day (based on license count) because exceeding it causes all subsequent `sendEmail()` calls to fail silently for the rest of the rolling 24-hour period.
- Never exceed 1,000 mass emails per day (for orgs with under 50,000 external emails allocated) because `MassEmailMessage` has a separate daily cap and excess emails are silently dropped.
- Always check sandbox deliverability settings before debugging "emails not sending" because sandboxes default to "System email only" -- all Apex-sent emails to external addresses are silently dropped. Set to "All email" in Setup > Email > Deliverability immediately after every sandbox refresh.

## How It Works

### Outbound Email: SingleEmailMessage

`Messaging.SingleEmailMessage` is the workhorse class for sending email from Apex. It supports two recipient-targeting strategies that have fundamentally different capabilities:

**`setTargetObjectId(Id)`** -- accepts a Contact, Lead, or User Id. Enables:
- Merge field resolution in email templates (`{!Contact.FirstName}`)
- Activity logging via `setSaveAsActivity(true)`
- Respect for the recipient's `HasOptedOutOfEmail` flag
- Merge field resolution in programmatic bodies when combined with `setTreatBodiesAsTemplate(true)`

**`setToAddresses(List<String>)`** -- accepts up to 100 raw email address strings. Use this when:
- The recipient is not a Salesforce record
- You need to send to multiple addresses in one message (CC/BCC equivalent via `setCcAddresses()` / `setBccAddresses()`)
- You are sending system notifications to external endpoints

You can use both on the same message (different addresses), but never for the same recipient.

**Key methods on SingleEmailMessage:**

| Method | Purpose |
|--------|---------|
| `setTargetObjectId(Id)` | Contact/Lead/User recipient -- enables templates and activity |
| `setToAddresses(List<String>)` | Raw email addresses (up to 100) |
| `setCcAddresses(List<String>)` | CC recipients (up to 25) |
| `setBccAddresses(List<String>)` | BCC recipients (up to 25) |
| `setSubject(String)` | Email subject line |
| `setHtmlBody(String)` | HTML body -- use inline styles, not CSS classes |
| `setPlainTextBody(String)` | Plain text fallback body |
| `setTemplateId(Id)` | Use a stored EmailTemplate -- overrides subject and body |
| `setWhatId(Id)` | Related record for merge fields and activity linking |
| `setOrgWideEmailAddressId(Id)` | Controls the From address, display name, and reply-to |
| `setSaveAsActivity(Boolean)` | Log email as activity on the Contact/Lead (default: true) |
| `setFileAttachments(List<EmailFileAttachment>)` | In-memory blob attachments |
| `setEntityAttachments(List<String>)` | Attach existing ContentVersion/Document IDs |
| `setTreatBodiesAsTemplate(Boolean)` | Resolve merge fields in programmatic body |
| `setTreatTargetObjectAsRecipient(Boolean)` | When false, targetObjectId resolves merge fields but email goes to `toAddresses` |
| `setReplyTo(String)` | Reply-to address (overridden by org-wide address) |
| `setSenderDisplayName(String)` | From display name (overridden by org-wide address) |
| `setInReplyTo(String)` | Message-ID of email being replied to (for threading) |
| `setReferences(String)` | References header (for email threading) |

### Outbound Email: MassEmailMessage

`Messaging.MassEmailMessage` is for sending the same template to many recipients at once. It is more limited than `SingleEmailMessage`:

- Requires `setTargetObjectIds(List<Id>)` -- Contact or Lead IDs only (up to 250 per message)
- Requires `setTemplateId(Id)` -- you cannot use programmatic body
- Optionally `setWhatIds(List<Id>)` -- must be the same length as targetObjectIds, paired 1:1 for merge field resolution
- Daily limit: 1,000 mass emails per day (varies by org edition)
- Does not support `setToAddresses()`, `setHtmlBody()`, `setPlainTextBody()`, or `setFileAttachments()`

Use `MassEmailMessage` when you need to send a templated email to hundreds of Contacts/Leads. For anything requiring custom body logic or non-Contact/Lead recipients, use `SingleEmailMessage` in a Batch Apex job instead.

### Org-Wide Email Addresses

Org-Wide Email Addresses let you control the From address on outbound email. They are configured in Setup > Email > Organization-Wide Addresses and must be verified before use.

Query them by address -- never hardcode the Id:

```apex
List<OrgWideEmailAddress> owea = [
    SELECT Id, DisplayName, Address
    FROM OrgWideEmailAddress
    WHERE Address = 'support@yourcompany.com'
    LIMIT 1
];
```

When you set `setOrgWideEmailAddressId()`, it overrides `setSenderDisplayName()` and `setReplyTo()`. The display name and reply-to come from the org-wide address configuration.

Only verified org-wide addresses can be used. In sandboxes, you may need to re-verify after a refresh.

### Email Templates: Classic vs Lightning

**Classic Email Templates** (the original) -- stored as `EmailTemplate` records with `TemplateType` of `text`, `html`, `custom`, or `visualforce`. Merge field syntax: `{!Contact.FirstName}`, `{!Opportunity.Amount}`. The `whoId` (Contact/Lead) and `whatId` (related record) control which merge fields resolve.

**Lightning Email Templates** -- also stored as `EmailTemplate` records but with `UiType = 'SFX'` and `TemplateType = 'custom'`. Use the same merge field syntax but are built with the Lightning email template editor (drag-and-drop, enhanced rich text). They work with `setTemplateId()` the same way.

Both template types work with `setTemplateId()` in Apex. Query by `DeveloperName` to avoid hardcoding IDs:

```apex
EmailTemplate tmpl = [
    SELECT Id FROM EmailTemplate
    WHERE DeveloperName = 'Purchase_Order_Confirmation'
    LIMIT 1
];
```

### Rendering Templates Without Sending

`Messaging.renderStoredEmailTemplate()` renders a template into a `SingleEmailMessage` without sending it. This lets you inspect, modify, or log the rendered output.

**Signature:**
```apex
Messaging.SingleEmailMessage rendered = Messaging.renderStoredEmailTemplate(
    templateId,  // Id of the EmailTemplate
    whoId,       // Contact, Lead, or User Id for recipient merge fields
    whatId       // Related record Id for relatedTo merge fields
);
```

The returned `SingleEmailMessage` has its subject, HTML body, and plain text body populated with resolved merge fields. You can then modify the body, change the recipient, add attachments, and send it.

**Extended signature with attachment options:**
```apex
Messaging.SingleEmailMessage rendered = Messaging.renderStoredEmailTemplate(
    templateId,
    whoId,
    whatId,
    Messaging.AttachmentRetrievalOption.METADATA_WITH_BODY,  // or METADATA_ONLY, NONE
    false  // updateTemplateUsage -- set false to avoid incrementing usage count
);
```

### Attachments: In-Memory vs Stored

**`setFileAttachments(List<EmailFileAttachment>)`** -- for dynamically generated content (PDF blobs, CSV exports, generated files). The `EmailFileAttachment` class has these methods:

| Method | Purpose |
|--------|---------|
| `setFileName(String)` | File name with extension (e.g., `'Invoice.pdf'`) |
| `setBody(Blob)` | The file content as a Blob |
| `setContentType(String)` | MIME type (e.g., `'application/pdf'`, `'text/csv'`) |
| `setInline(Boolean)` | True = inline display hint, False = downloadable attachment |
| `setContentId(String)` | Content-ID header for inline references (limited support) |

**`setEntityAttachments(List<String>)`** -- for files already stored in Salesforce. Pass a list of ContentVersion IDs (18-character). These attachments also appear on the logged activity record when `setSaveAsActivity(true)` is set. This is the preferred method when the file already exists in Salesforce because it avoids reading the blob into memory.

Attachment limits: 3 MB per file, 25 MB total per email.

### Inline Images in HTML Email

Salesforce has limited support for true CID-based inline images (`<img src="cid:my-image">`). The `setContentId()` method on `EmailFileAttachment` exists but its behavior is inconsistent across email clients and Salesforce versions.

**Reliable alternatives:**

1. **Base64 data URIs** -- embed small images directly in the HTML: `<img src="data:image/png;base64,iVBOR...">`  Works for logos and icons under ~30KB. Some email clients (notably older Outlook) do not render data URIs.

2. **Publicly accessible URLs** -- host images on a CDN or Salesforce public site and reference them with `<img src="https://...">`. Most reliable approach for images the email client can fetch.

3. **ContentVersion reference URLs** -- for files stored in Salesforce, use the Salesforce content delivery URL. Requires the recipient to be able to access the file (authentication may be needed).

### Inbound Email: InboundEmailHandler

Inbound email processing uses three classes:

1. **`Messaging.InboundEmailHandler`** -- the interface your class implements. One method: `handleInboundEmail(Messaging.InboundEmail email, Messaging.InboundEnvelope envelope)`.

2. **`Messaging.InboundEmail`** -- the parsed email. Key properties:

| Property | Type | Description |
|----------|------|-------------|
| `fromAddress` | `String` | Sender's email address |
| `fromName` | `String` | Sender's display name |
| `toAddresses` | `String[]` | To recipients |
| `ccAddresses` | `String[]` | CC recipients |
| `subject` | `String` | Email subject |
| `plainTextBody` | `String` | Plain text body (null if none) |
| `htmlBody` | `String` | HTML body (null if none) |
| `binaryAttachments` | `InboundEmail.BinaryAttachment[]` | Binary file attachments (null if none) |
| `textAttachments` | `InboundEmail.TextAttachment[]` | Text file attachments (null if none) |
| `headers` | `InboundEmail.Header[]` | Email headers |
| `messageId` | `String` | Message-ID header value |
| `inReplyTo` | `String` | In-Reply-To header (for threading) |
| `references` | `String[]` | References header (for threading) |

3. **`Messaging.InboundEnvelope`** -- the SMTP envelope. Contains `toAddress` (String) and `fromAddress` (String) -- these are the SMTP-level addresses, which may differ from the email header addresses.

4. **`Messaging.InboundEmailResult`** -- the return value. Set `result.success = true` to accept the email, `false` to bounce it back to the sender. Set `result.message` to include a custom bounce message.

**Email Service setup:** After creating the Apex class, configure the Email Service in Setup > Email Services. Each service generates a unique email address (e.g., `handler@1a2b3c.in.salesforce.com`). You can set the running user context, whether to accept attachments, and allowed sender domains.

### Activity Tracking and Enhanced Email

When `setSaveAsActivity(true)` is set and `setTargetObjectId()` points to a Contact or Lead:

**With Enhanced Email (default since 2020):** An `EmailMessage` record is created. This record has rich tracking fields:
- `Status`: 0 (New), 1 (Read), 2 (Replied), 3 (Sent), 4 (Forwarded), 5 (Draft)
- `IsOpened`: True when the recipient opens the email (requires tracking pixel support)
- `IsBounced`: True when the email bounces
- `FirstOpenedDate` / `LastOpenedDate`: Timestamps of email opens
- `RelatedToId`: The record from `setWhatId()`
- `FromAddress`, `ToAddress`, `Subject`, `HtmlBody`, `TextBody`: Full email content

**Without Enhanced Email:** A `Task` record is created with `Subject = 'Email: <your subject>'` and minimal metadata.

To query sent emails:
```apex
// get all emails sent to a specific contact about a specific record
List<EmailMessage> sentEmails = [
    SELECT Id, Subject, Status, IsOpened, IsBounced,
           FirstOpenedDate, ToAddress, MessageDate
    FROM EmailMessage
    WHERE RelatedToId = :recordId
      AND ToAddress = :contactEmail
    ORDER BY MessageDate DESC
];
```

### Bounce Management

Bounce management is configured in Setup > Email > Deliverability > Activate Bounce Management. When enabled:

- Salesforce automatically populates `EmailBouncedReason` and `EmailBouncedDate` fields on Contact and Lead records when an email bounces
- An alert icon appears next to bounced email addresses in the UI
- The `IsBounced` field on `EmailMessage` is set to true
- Users are warned when attempting to email a bounced address

You can query bounce data programmatically:

```apex
// find contacts with bounced emails
List<Contact> bouncedContacts = [
    SELECT Id, Email, EmailBouncedReason, EmailBouncedDate
    FROM Contact
    WHERE EmailBouncedDate != null
];
```

### Deliverability Settings

**Setup path:** Setup > Email > Deliverability

Three access levels:
- **No access** -- all outbound email blocked
- **System email only** -- only password resets, user setup, and system alerts send (sandbox default)
- **All email** -- everything sends (production default)

Sandboxes default to "System email only" after every refresh. This silently drops all Apex-sent emails. Change to "All email" immediately after refresh when testing email functionality.

### Testing Email in Apex

**Emails do not actually send in test context.** Salesforce executes the email code, validates parameters, and counts against per-transaction limits, but no email leaves the org. This is by design -- you cannot test actual email delivery in Apex tests.

**What you CAN assert:**
- `SendEmailResult.isSuccess()` -- whether Salesforce accepted the email for queuing
- `SendEmailResult.getErrors()` -- error details on failure
- `Limits.getEmailInvocations()` -- how many `sendEmail()` calls were made in the current transaction
- Record state changes caused by your email logic (activity records, status field updates)

**What you CANNOT test:**
- Actual email delivery
- Email template rendering (merge fields resolve to empty strings in most test contexts unless you use `renderStoredEmailTemplate()`)
- `OrgWideEmailAddress` queries (these records cannot be created in test context and are not accessible via `@TestSetup`)

## Code Examples

### Complete Email Service with Template and Programmatic Fallback

```apex
public with sharing class NotificationEmailService {

    // send a notification email to a contact using a template if available,
    // falling back to a programmatic body if the template is not found
    // @param contactId: the Contact to receive the email
    // @param whatId: the related record Id for merge fields and activity linking
    // @param templateDevName: the DeveloperName of the email template to use
    // @param fallbackSubject: subject line if template not found
    // @param fallbackHtmlBody: HTML body if template not found
    // @param fallbackPlainBody: plain text body if template not found
    // @return Messaging.SendEmailResult: the result of the send operation
    public static Messaging.SendEmailResult sendNotification(
        Id contactId,
        Id whatId,
        String templateDevName,
        String fallbackSubject,
        String fallbackHtmlBody,
        String fallbackPlainBody
    ) {
        System.debug('Sending notification to Contact: ' + contactId + ', whatId: ' + whatId);

        Messaging.SingleEmailMessage email = new Messaging.SingleEmailMessage();

        // always set the target object for activity logging and merge fields
        email.setTargetObjectId(contactId);
        email.setWhatId(whatId);
        email.setSaveAsActivity(true);

        // set org-wide email address -- query by address, never hardcode Id
        setOrgWideAddress(email, 'notifications@yourcompany.com');

        // try to use the email template
        List<EmailTemplate> templates = [
            SELECT Id FROM EmailTemplate
            WHERE DeveloperName = :templateDevName
            LIMIT 1
        ];

        if (!templates.isEmpty()) {
            // template found -- let Salesforce handle subject and body via merge fields
            email.setTemplateId(templates[0].Id);
            // do NOT also set subject/body -- the template overrides them
            System.debug('Using email template: ' + templateDevName);
        } else {
            // template not found -- use the programmatic fallback
            // set both HTML and plain text for deliverability
            email.setSubject(fallbackSubject);
            email.setHtmlBody(fallbackHtmlBody);
            email.setPlainTextBody(fallbackPlainBody);
            System.debug('Template not found, using programmatic body');
        }

        // send in a single call -- never call sendEmail() in a loop
        List<Messaging.SendEmailResult> results = Messaging.sendEmail(
            new List<Messaging.SingleEmailMessage>{ email }
        );

        Messaging.SendEmailResult result = results[0];
        if (!result.isSuccess()) {
            for (Messaging.SendEmailError err : result.getErrors()) {
                System.debug(LoggingLevel.ERROR,
                    'Email send failed. Status: ' + err.getStatusCode() +
                    ', Message: ' + err.getMessage()
                );
            }
        }

        return result;
    }

    // helper to set the org-wide email address on a message
    // @param email: the SingleEmailMessage to configure
    // @param address: the verified org-wide email address string
    private static void setOrgWideAddress(Messaging.SingleEmailMessage email, String address) {
        List<OrgWideEmailAddress> owea = [
            SELECT Id FROM OrgWideEmailAddress
            WHERE Address = :address
            LIMIT 1
        ];
        if (!owea.isEmpty()) {
            email.setOrgWideEmailAddressId(owea[0].Id);
        } else {
            System.debug(LoggingLevel.WARN,
                'Org-wide email address not found: ' + address +
                '. Email will send from running user address.'
            );
        }
    }
}
```

### Rendering a Template, Modifying the Body, Then Sending

```apex
public with sharing class CustomRenderedEmailService {

    // render an email template, inject a custom section, and send
    // useful when you need template merge fields but also need to add
    // dynamic content that templates can't handle (aggregated data, tables, etc.)
    // @param contactId: the Contact to receive the email
    // @param whatId: the related record for merge fields
    // @param templateDevName: the DeveloperName of the template
    // @param customHtmlSection: HTML to inject before the closing tag
    // @return Boolean: true if email was queued successfully
    public static Boolean sendRenderedWithCustomSection(
        Id contactId,
        Id whatId,
        String templateDevName,
        String customHtmlSection
    ) {
        System.debug('Rendering template and injecting custom section');

        EmailTemplate tmpl = [
            SELECT Id FROM EmailTemplate
            WHERE DeveloperName = :templateDevName
            LIMIT 1
        ];

        // render the template -- this resolves all merge fields
        // without actually sending the email
        Messaging.SingleEmailMessage rendered = Messaging.renderStoredEmailTemplate(
            tmpl.Id,
            contactId,  // whoId -- resolves Contact merge fields
            whatId       // whatId -- resolves relatedTo merge fields
        );

        // extract the rendered body and inject our custom section
        String renderedHtml = rendered.getHtmlBody();
        String renderedPlain = rendered.getPlainBody();
        String renderedSubject = rendered.getSubject();

        // inject the custom section before the closing </body> or at the end
        if (renderedHtml != null && renderedHtml.containsIgnoreCase('</body>')) {
            renderedHtml = renderedHtml.replaceFirst(
                '(?i)</body>',
                customHtmlSection + '</body>'
            );
        } else if (renderedHtml != null) {
            renderedHtml += customHtmlSection;
        }

        // build a new email with the modified content
        // do NOT use setTemplateId -- we already rendered the template
        Messaging.SingleEmailMessage email = new Messaging.SingleEmailMessage();
        email.setTargetObjectId(contactId);
        email.setWhatId(whatId);
        email.setSaveAsActivity(true);
        email.setSubject(renderedSubject);
        email.setHtmlBody(renderedHtml);
        email.setPlainTextBody(renderedPlain);

        // set org-wide address
        List<OrgWideEmailAddress> owea = [
            SELECT Id FROM OrgWideEmailAddress
            WHERE Address = 'noreply@yourcompany.com'
            LIMIT 1
        ];
        if (!owea.isEmpty()) {
            email.setOrgWideEmailAddressId(owea[0].Id);
        }

        List<Messaging.SendEmailResult> results = Messaging.sendEmail(
            new List<Messaging.SingleEmailMessage>{ email }
        );

        return results[0].isSuccess();
    }
}
```

### Sending to Non-Contacts with Merge-Field-Like Behavior

```apex
public with sharing class AdHocEmailService {

    // send an email to an arbitrary address that is not a Contact or Lead
    // uses setTreatTargetObjectAsRecipient(false) so a Contact is used
    // for merge field resolution but the email goes to a different address
    // @param toAddress: the actual recipient email address
    // @param contactIdForMerge: a Contact Id used solely for merge field resolution
    // @param whatId: the related record for relatedTo merge fields
    // @param subject: email subject
    // @param htmlBody: HTML body with merge field syntax like {!Contact.FirstName}
    // @param plainBody: plain text body
    // @return Boolean: true if email was queued successfully
    public static Boolean sendToArbitraryAddress(
        String toAddress,
        Id contactIdForMerge,
        Id whatId,
        String subject,
        String htmlBody,
        String plainBody
    ) {
        System.debug('Sending ad-hoc email to: ' + toAddress);

        Messaging.SingleEmailMessage email = new Messaging.SingleEmailMessage();

        // the actual recipient is the arbitrary address
        email.setToAddresses(new List<String>{ toAddress });

        // use the contact for merge field resolution only -- email does NOT go to the contact
        email.setTargetObjectId(contactIdForMerge);
        email.setTreatTargetObjectAsRecipient(false);

        // enable merge field syntax in the programmatic body
        email.setTreatBodiesAsTemplate(true);

        email.setWhatId(whatId);
        email.setSubject(subject);
        email.setHtmlBody(htmlBody);
        email.setPlainTextBody(plainBody);

        // no activity logging -- setSaveAsActivity must be false when
        // setTreatTargetObjectAsRecipient is false
        email.setSaveAsActivity(false);

        List<Messaging.SendEmailResult> results = Messaging.sendEmail(
            new List<Messaging.SingleEmailMessage>{ email }
        );

        return results[0].isSuccess();
    }
}
```

### MassEmailMessage to a List of Contacts

```apex
public with sharing class MassNotificationService {

    // send a templated mass email to a list of contacts
    // @param contactIds: list of Contact Ids to receive the email (max 250)
    // @param relatedRecordIds: list of related record Ids, paired 1:1 with contactIds
    //     (pass null if no related record merge fields needed)
    // @param templateDevName: the DeveloperName of the email template
    // @return Boolean: true if the mass email was queued successfully
    public static Boolean sendMassNotification(
        List<Id> contactIds,
        List<Id> relatedRecordIds,
        String templateDevName
    ) {
        System.debug('Sending mass email to ' + contactIds.size() + ' contacts');

        // MassEmailMessage caps at 250 recipients per message
        if (contactIds.size() > 250) {
            throw new MassEmailException(
                'MassEmailMessage supports up to 250 recipients. ' +
                'Received: ' + contactIds.size() + '. Split into multiple batches.'
            );
        }

        EmailTemplate tmpl = [
            SELECT Id FROM EmailTemplate
            WHERE DeveloperName = :templateDevName
            LIMIT 1
        ];

        Messaging.MassEmailMessage massEmail = new Messaging.MassEmailMessage();
        massEmail.setTargetObjectIds(contactIds);
        massEmail.setTemplateId(tmpl.Id);

        // setWhatIds is optional -- if provided, must match the length
        // of targetObjectIds and entries are paired 1:1
        if (relatedRecordIds != null && !relatedRecordIds.isEmpty()) {
            if (relatedRecordIds.size() != contactIds.size()) {
                throw new MassEmailException(
                    'relatedRecordIds must be the same length as contactIds. ' +
                    'Contacts: ' + contactIds.size() +
                    ', Related: ' + relatedRecordIds.size()
                );
            }
            massEmail.setWhatIds(relatedRecordIds);
        }

        // setSaveAsActivity defaults to true for MassEmailMessage
        // each recipient gets an individual activity record

        List<Messaging.SendEmailResult> results = Messaging.sendEmail(
            new List<Messaging.MassEmailMessage>{ massEmail }
        );

        Boolean success = results[0].isSuccess();
        if (!success) {
            for (Messaging.SendEmailError err : results[0].getErrors()) {
                System.debug(LoggingLevel.ERROR,
                    'Mass email failed: ' + err.getMessage()
                );
            }
        }

        return success;
    }

    public class MassEmailException extends Exception {}
}
```

### Attaching Existing ContentVersion Files

```apex
public with sharing class EmailWithStoredAttachments {

    // send an email with files that are already stored in Salesforce as ContentVersions
    // @param contactId: the Contact to receive the email
    // @param whatId: the related record
    // @param subject: email subject
    // @param htmlBody: HTML email body
    // @param plainBody: plain text email body
    // @param contentVersionIds: list of ContentVersion Ids to attach
    // @return Boolean: true if email was queued successfully
    public static Boolean sendWithExistingFiles(
        Id contactId,
        Id whatId,
        String subject,
        String htmlBody,
        String plainBody,
        List<Id> contentVersionIds
    ) {
        System.debug('Sending email with ' + contentVersionIds.size() + ' stored attachments');

        Messaging.SingleEmailMessage email = new Messaging.SingleEmailMessage();
        email.setTargetObjectId(contactId);
        email.setWhatId(whatId);
        email.setSaveAsActivity(true);
        email.setSubject(subject);
        email.setHtmlBody(htmlBody);
        email.setPlainTextBody(plainBody);

        // setEntityAttachments takes a list of ContentVersion ID strings
        // these files are pulled from Salesforce storage, not loaded into memory
        // they also appear on the activity record when setSaveAsActivity is true
        List<String> cvIdStrings = new List<String>();
        for (Id cvId : contentVersionIds) {
            cvIdStrings.add(String.valueOf(cvId));
        }
        email.setEntityAttachments(cvIdStrings);

        List<OrgWideEmailAddress> owea = [
            SELECT Id FROM OrgWideEmailAddress
            WHERE Address = 'documents@yourcompany.com'
            LIMIT 1
        ];
        if (!owea.isEmpty()) {
            email.setOrgWideEmailAddressId(owea[0].Id);
        }

        List<Messaging.SendEmailResult> results = Messaging.sendEmail(
            new List<Messaging.SingleEmailMessage>{ email }
        );

        return results[0].isSuccess();
    }
}
```

### Complete Inbound Email Handler

```apex
public with sharing class CaseCreationEmailHandler implements Messaging.InboundEmailHandler {

    // handle inbound emails to create Cases
    // email subject becomes the Case Subject
    // email body becomes the Case Description
    // sender is matched to an existing Contact if possible
    // @param email: the parsed inbound email
    // @param envelope: the SMTP envelope with to/from addresses
    // @return Messaging.InboundEmailResult: success or failure
    public Messaging.InboundEmailResult handleInboundEmail(
        Messaging.InboundEmail email,
        Messaging.InboundEnvelope envelope
    ) {
        Messaging.InboundEmailResult result = new Messaging.InboundEmailResult();

        try {
            System.debug('Inbound email from: ' + email.fromAddress);
            System.debug('Subject: ' + email.subject);

            // try to match the sender to an existing Contact
            List<Contact> senderContacts = [
                SELECT Id, AccountId
                FROM Contact
                WHERE Email = :email.fromAddress
                LIMIT 1
            ];

            // create the Case
            Case newCase = new Case(
                Subject = email.subject,
                Description = buildDescription(email),
                Origin = 'Email',
                Status = 'New',
                Priority = determinePriority(email.subject),
                SuppliedEmail = email.fromAddress,
                SuppliedName = email.fromName
            );

            // link to Contact and Account if we found a match
            if (!senderContacts.isEmpty()) {
                newCase.ContactId = senderContacts[0].Id;
                newCase.AccountId = senderContacts[0].AccountId;
                System.debug('Matched sender to Contact: ' + senderContacts[0].Id);
            }

            insert newCase;
            System.debug('Created Case: ' + newCase.Id);

            // process attachments -- remember to check for null, not empty list
            if (email.binaryAttachments != null) {
                saveAttachments(newCase.Id, email.binaryAttachments);
            }
            if (email.textAttachments != null) {
                saveTextAttachments(newCase.Id, email.textAttachments);
            }

            // always return success = true unless you want to bounce the email
            // back to the sender with an error message
            result.success = true;

        } catch (Exception e) {
            System.debug(LoggingLevel.ERROR,
                'Error processing inbound email: ' + e.getMessage() +
                '\nStack trace: ' + e.getStackTraceString()
            );
            // return success = true even on error to avoid bouncing
            // the email back to the sender with internal error details
            // log the error for internal investigation instead
            result.success = true;
        }

        return result;
    }

    // build a description from the email body
    // prefer plain text over HTML for the Case description field
    // @param email: the inbound email
    // @return String: the description text
    private String buildDescription(Messaging.InboundEmail email) {
        if (String.isNotBlank(email.plainTextBody)) {
            return email.plainTextBody;
        } else if (String.isNotBlank(email.htmlBody)) {
            // strip HTML tags for the description field
            // a rough approach -- for production, consider a proper HTML-to-text utility
            return email.htmlBody.stripHtmlTags();
        }
        return '(No email body)';
    }

    // determine priority from subject line keywords
    // @param subject: the email subject
    // @return String: the Case priority
    private String determinePriority(String subject) {
        if (subject == null) {
            return 'Medium';
        }
        String lower = subject.toLowerCase();
        if (lower.contains('urgent') || lower.contains('critical') || lower.contains('down')) {
            return 'High';
        }
        return 'Medium';
    }

    // save binary attachments as ContentVersions linked to the Case
    // @param caseId: the Case record Id
    // @param attachments: the binary attachments from the inbound email
    private void saveAttachments(
        Id caseId,
        List<Messaging.InboundEmail.BinaryAttachment> attachments
    ) {
        List<ContentVersion> cvList = new List<ContentVersion>();
        for (Messaging.InboundEmail.BinaryAttachment att : attachments) {
            ContentVersion cv = new ContentVersion(
                Title = att.fileName,
                PathOnClient = att.fileName,
                VersionData = att.body,
                FirstPublishLocationId = caseId  // links to the Case automatically
            );
            cvList.add(cv);
        }
        if (!cvList.isEmpty()) {
            insert cvList;
            System.debug('Saved ' + cvList.size() + ' binary attachments to Case');
        }
    }

    // save text attachments as ContentVersions linked to the Case
    // @param caseId: the Case record Id
    // @param attachments: the text attachments from the inbound email
    private void saveTextAttachments(
        Id caseId,
        List<Messaging.InboundEmail.TextAttachment> attachments
    ) {
        List<ContentVersion> cvList = new List<ContentVersion>();
        for (Messaging.InboundEmail.TextAttachment att : attachments) {
            ContentVersion cv = new ContentVersion(
                Title = att.fileName,
                PathOnClient = att.fileName,
                VersionData = Blob.valueOf(att.body),
                FirstPublishLocationId = caseId
            );
            cvList.add(cv);
        }
        if (!cvList.isEmpty()) {
            insert cvList;
            System.debug('Saved ' + cvList.size() + ' text attachments to Case');
        }
    }
}
```

### Batch Email Sending Pattern

```apex
public with sharing class BatchEmailSender implements
    Database.Batchable<SObject>, Database.Stateful {

    // batch job to send individual emails to a large number of contacts
    // respects the 10-sendEmail-invocations-per-transaction limit by sending
    // all emails in each batch execute as a single sendEmail() call
    // uses Database.Stateful to track success/failure counts across batches

    private String templateDevName;
    private Id relatedRecordId;
    private Integer successCount = 0;
    private Integer failureCount = 0;

    // @param templateDevName: the DeveloperName of the email template
    // @param relatedRecordId: the record Id for whatId merge fields
    public BatchEmailSender(String templateDevName, Id relatedRecordId) {
        this.templateDevName = templateDevName;
        this.relatedRecordId = relatedRecordId;
    }

    // query contacts that need to receive the email
    public Database.QueryLocator start(Database.BatchableContext bc) {
        return Database.getQueryLocator([
            SELECT Id, Email
            FROM Contact
            WHERE Email != null
              AND HasOptedOutOfEmail = false
              AND AccountId != null
        ]);
    }

    // send emails in each batch -- use a batch size of 10 or fewer
    // because each SingleEmailMessage with a template counts against the
    // 10-message limit when using setTemplateId
    // @param bc: the batch context
    // @param contacts: the contacts in this batch chunk
    public void execute(Database.BatchableContext bc, List<Contact> contacts) {
        System.debug('Processing batch of ' + contacts.size() + ' contacts');

        EmailTemplate tmpl = [
            SELECT Id FROM EmailTemplate
            WHERE DeveloperName = :templateDevName
            LIMIT 1
        ];

        List<OrgWideEmailAddress> owea = [
            SELECT Id FROM OrgWideEmailAddress
            WHERE Address = 'notifications@yourcompany.com'
            LIMIT 1
        ];
        Id oweaId = owea.isEmpty() ? null : owea[0].Id;

        List<Messaging.SingleEmailMessage> emails = new List<Messaging.SingleEmailMessage>();

        for (Contact con : contacts) {
            Messaging.SingleEmailMessage msg = new Messaging.SingleEmailMessage();
            msg.setTargetObjectId(con.Id);
            msg.setWhatId(relatedRecordId);
            msg.setTemplateId(tmpl.Id);
            msg.setSaveAsActivity(true);
            if (oweaId != null) {
                msg.setOrgWideEmailAddressId(oweaId);
            }
            emails.add(msg);
        }

        // ONE sendEmail call per execute -- uses 1 of 10 invocations
        if (!emails.isEmpty()) {
            List<Messaging.SendEmailResult> results = Messaging.sendEmail(emails, false);
            for (Messaging.SendEmailResult res : results) {
                if (res.isSuccess()) {
                    successCount++;
                } else {
                    failureCount++;
                    for (Messaging.SendEmailError err : res.getErrors()) {
                        System.debug(LoggingLevel.ERROR, 'Email failed: ' + err.getMessage());
                    }
                }
            }
        }
    }

    // log final results
    public void finish(Database.BatchableContext bc) {
        System.debug('Batch email complete. Success: ' + successCount + ', Failed: ' + failureCount);

        // optionally send a summary email to the admin
        // this uses 1 sendEmail invocation in the finish context
        if (failureCount > 0) {
            Messaging.SingleEmailMessage summary = new Messaging.SingleEmailMessage();
            summary.setToAddresses(new List<String>{ 'admin@yourcompany.com' });
            summary.setSubject('Batch Email Job Complete');
            summary.setPlainTextBody(
                'Batch email sending complete.\n' +
                'Successful: ' + successCount + '\n' +
                'Failed: ' + failureCount
            );
            Messaging.sendEmail(new List<Messaging.SingleEmailMessage>{ summary });
        }
    }
}
```

Execute the batch with a scope size that respects limits:

```apex
// use a batch size of 10 -- each contact becomes one SingleEmailMessage
// all 10 send in one sendEmail() call, using 1 of 10 invocations per execute
Database.executeBatch(new BatchEmailSender('Welcome_Email', someRecordId), 10);
```

### Email Queue Pattern with Custom Object

```apex
// custom object: Email_Queue__c
// fields: To_Address__c (Email), Contact__c (Lookup to Contact),
//         Related_Record_Id__c (Text), Subject__c (Text),
//         Html_Body__c (LongTextArea), Plain_Body__c (LongTextArea),
//         Status__c (Picklist: Pending, Sent, Failed), Error_Message__c (LongTextArea),
//         Template_Dev_Name__c (Text), Retry_Count__c (Number)

public with sharing class EmailQueueProcessor implements Schedulable {

    // scheduled job that processes pending email queue records
    // run every 5 minutes to drain the queue
    // @param sc: the schedulable context
    public void execute(SchedulableContext sc) {
        // query up to 10 pending emails (respects the 10-message limit)
        List<Email_Queue__c> pendingEmails = [
            SELECT Id, To_Address__c, Contact__c, Related_Record_Id__c,
                   Subject__c, Html_Body__c, Plain_Body__c,
                   Template_Dev_Name__c, Retry_Count__c
            FROM Email_Queue__c
            WHERE Status__c = 'Pending'
              AND Retry_Count__c < 3
            ORDER BY CreatedDate ASC
            LIMIT 10
        ];

        if (pendingEmails.isEmpty()) {
            return;
        }

        System.debug('Processing ' + pendingEmails.size() + ' queued emails');

        List<Messaging.SingleEmailMessage> emails = new List<Messaging.SingleEmailMessage>();
        // map to correlate results back to queue records
        List<Email_Queue__c> queueRecords = new List<Email_Queue__c>();

        // query org-wide address once
        List<OrgWideEmailAddress> owea = [
            SELECT Id FROM OrgWideEmailAddress
            WHERE Address = 'noreply@yourcompany.com'
            LIMIT 1
        ];
        Id oweaId = owea.isEmpty() ? null : owea[0].Id;

        for (Email_Queue__c eq : pendingEmails) {
            Messaging.SingleEmailMessage msg = new Messaging.SingleEmailMessage();

            if (eq.Contact__c != null) {
                // send to a Contact -- enables activity logging and templates
                msg.setTargetObjectId(eq.Contact__c);
                msg.setSaveAsActivity(true);
            } else if (String.isNotBlank(eq.To_Address__c)) {
                // send to a raw address -- no activity logging
                msg.setToAddresses(new List<String>{ eq.To_Address__c });
                msg.setSaveAsActivity(false);
            } else {
                // no recipient -- mark as failed and skip
                eq.Status__c = 'Failed';
                eq.Error_Message__c = 'No recipient specified';
                queueRecords.add(eq);
                continue;
            }

            if (String.isNotBlank(eq.Related_Record_Id__c)) {
                msg.setWhatId(Id.valueOf(eq.Related_Record_Id__c));
            }

            // use template if specified, otherwise use programmatic body
            if (String.isNotBlank(eq.Template_Dev_Name__c)) {
                List<EmailTemplate> templates = [
                    SELECT Id FROM EmailTemplate
                    WHERE DeveloperName = :eq.Template_Dev_Name__c
                    LIMIT 1
                ];
                if (!templates.isEmpty()) {
                    msg.setTemplateId(templates[0].Id);
                }
            } else {
                msg.setSubject(eq.Subject__c);
                msg.setHtmlBody(eq.Html_Body__c);
                msg.setPlainTextBody(eq.Plain_Body__c);
            }

            if (oweaId != null) {
                msg.setOrgWideEmailAddressId(oweaId);
            }

            emails.add(msg);
            queueRecords.add(eq);
        }

        // send all emails in one call
        if (!emails.isEmpty()) {
            List<Messaging.SendEmailResult> results = Messaging.sendEmail(emails, false);

            // correlate results back to queue records
            // note: queueRecords may contain records that were marked Failed above
            // and skipped in the emails list -- so we track a separate index
            Integer emailIndex = 0;
            for (Email_Queue__c eq : queueRecords) {
                if (eq.Status__c == 'Failed') {
                    // already marked as failed above, skip
                    continue;
                }
                if (emailIndex < results.size()) {
                    if (results[emailIndex].isSuccess()) {
                        eq.Status__c = 'Sent';
                    } else {
                        eq.Status__c = 'Failed';
                        eq.Retry_Count__c = (eq.Retry_Count__c == null ? 0 : eq.Retry_Count__c) + 1;
                        // reset to Pending if under retry limit so it gets picked up again
                        if (eq.Retry_Count__c < 3) {
                            eq.Status__c = 'Pending';
                        }
                        String errorMsg = '';
                        for (Messaging.SendEmailError err : results[emailIndex].getErrors()) {
                            errorMsg += err.getStatusCode() + ': ' + err.getMessage() + '\n';
                        }
                        eq.Error_Message__c = errorMsg;
                    }
                    emailIndex++;
                }
            }
        }

        // update all queue records with their new status
        update queueRecords;
    }
}
```

Schedule the processor:

```apex
// run every 5 minutes
String cronExp = '0 0/5 * * * ?';
System.schedule('Email Queue Processor', cronExp, new EmailQueueProcessor());
```

### CustomNotification as a Lightweight Alternative to Email

```apex
public with sharing class InAppNotificationService {

    // send an in-app notification (bell icon) instead of email
    // useful for internal user alerts where email is overkill
    // @param recipientUserIds: the User Ids to notify
    // @param title: notification title (short)
    // @param body: notification body (can be longer)
    // @param targetRecordId: the record the notification links to
    // @param notificationTypeName: the DeveloperName of the CustomNotificationType
    public static void sendBellNotification(
        Set<Id> recipientUserIds,
        String title,
        String body,
        Id targetRecordId,
        String notificationTypeName
    ) {
        System.debug('Sending bell notification to ' + recipientUserIds.size() + ' users');

        // query the custom notification type by developer name
        CustomNotificationType notifType = [
            SELECT Id FROM CustomNotificationType
            WHERE DeveloperName = :notificationTypeName
            LIMIT 1
        ];

        Messaging.CustomNotification notification = new Messaging.CustomNotification();
        notification.setTitle(title);
        notification.setBody(body);
        notification.setTargetId(targetRecordId);
        notification.setNotificationTypeId(notifType.Id);

        try {
            notification.send(recipientUserIds);
            System.debug('Bell notification sent successfully');
        } catch (Exception e) {
            System.debug(LoggingLevel.ERROR,
                'Failed to send bell notification: ' + e.getMessage()
            );
        }
    }
}
```

### Complete Test Class for Email Services

```apex
@isTest
private class EmailServicesTest {

    @TestSetup
    static void makeData() {
        Account acct = new Account(Name = 'Test Account');
        insert acct;

        Contact con = new Contact(
            FirstName = 'Jane',
            LastName = 'Tester',
            Email = 'jane.tester@example.com',
            AccountId = acct.Id
        );
        insert con;
    }

    // test sending a notification email with programmatic body
    @isTest
    static void testSendNotification_programmaticBody() {
        Contact con = [SELECT Id FROM Contact LIMIT 1];
        Account acct = [SELECT Id FROM Account LIMIT 1];

        Test.startTest();
        // emails do NOT actually send in test context
        // but the code runs and counts against per-transaction limits
        Messaging.SendEmailResult result = NotificationEmailService.sendNotification(
            con.Id,
            acct.Id,
            'Nonexistent_Template',  // forces the programmatic fallback
            'Test Subject',
            '<p>Test HTML Body</p>',
            'Test Plain Body'
        );
        Test.stopTest();

        // assert the email was accepted for queuing
        System.assertEquals(true, result.isSuccess(),
            'Email should be queued successfully');

        // verify that exactly one sendEmail invocation occurred
        // Limits.getEmailInvocations() returns the count within the
        // Test.startTest() / Test.stopTest() block
        System.assertEquals(1, Limits.getEmailInvocations(),
            'Expected exactly one sendEmail invocation');
    }

    // test that the rendered email service works with template rendering
    // note: in test context, merge fields in templates resolve to empty strings
    // unless the template is a simple text template
    @isTest
    static void testSendRenderedEmail() {
        Contact con = [SELECT Id FROM Contact LIMIT 1];
        Account acct = [SELECT Id FROM Account LIMIT 1];

        // query a system template that exists in all orgs
        // in real tests, create the template in @TestSetup or use a known DeveloperName
        List<EmailTemplate> templates = [
            SELECT Id, DeveloperName FROM EmailTemplate
            WHERE IsActive = true
            LIMIT 1
        ];

        // skip if no templates exist in the test org
        if (templates.isEmpty()) {
            return;
        }

        Test.startTest();
        Boolean success = CustomRenderedEmailService.sendRenderedWithCustomSection(
            con.Id,
            acct.Id,
            templates[0].DeveloperName,
            '<div style="color: red;">Custom section</div>'
        );
        Test.stopTest();

        System.assertEquals(true, success, 'Rendered email should succeed');
    }

    // test the MassEmailMessage service
    @isTest
    static void testMassNotification() {
        Contact con = [SELECT Id FROM Contact LIMIT 1];

        List<EmailTemplate> templates = [
            SELECT Id, DeveloperName FROM EmailTemplate
            WHERE IsActive = true
            LIMIT 1
        ];

        if (templates.isEmpty()) {
            return;
        }

        Test.startTest();
        Boolean success = MassNotificationService.sendMassNotification(
            new List<Id>{ con.Id },
            null,  // no related record Ids
            templates[0].DeveloperName
        );
        Test.stopTest();

        System.assertEquals(true, success, 'Mass email should succeed');
    }

    // test the MassEmailMessage list size validation
    @isTest
    static void testMassNotification_exceedsLimit() {
        // create a list of 251 fake Ids to exceed the 250 limit
        List<Id> fakeIds = new List<Id>();
        for (Integer i = 0; i < 251; i++) {
            // generate fake Contact Ids for the size check
            // the exception should fire before any SOQL
            fakeIds.add(
                Id.valueOf('003' + String.valueOf(i).leftPad(15, '0'))
            );
        }

        Test.startTest();
        try {
            MassNotificationService.sendMassNotification(
                fakeIds,
                null,
                'Some_Template'
            );
            System.assert(false, 'Should have thrown MassEmailException');
        } catch (MassNotificationService.MassEmailException e) {
            System.assert(
                e.getMessage().contains('250'),
                'Error should mention the 250-recipient limit'
            );
        }
        Test.stopTest();
    }

    // test the inbound email handler
    @isTest
    static void testInboundEmailHandler_withContact() {
        // create the inbound email object -- this is how you test InboundEmailHandler
        // no actual email is sent or received
        Messaging.InboundEmail email = new Messaging.InboundEmail();
        email.fromAddress = 'jane.tester@example.com';  // matches our test Contact
        email.fromName = 'Jane Tester';
        email.subject = 'Urgent: System is down';
        email.plainTextBody = 'The production system is not responding.';

        // add a binary attachment
        Messaging.InboundEmail.BinaryAttachment att =
            new Messaging.InboundEmail.BinaryAttachment();
        att.fileName = 'screenshot.png';
        att.body = Blob.valueOf('fake image data');
        att.mimeTypeSubType = 'image/png';
        email.binaryAttachments = new List<Messaging.InboundEmail.BinaryAttachment>{ att };

        Messaging.InboundEnvelope envelope = new Messaging.InboundEnvelope();
        envelope.fromAddress = 'jane.tester@example.com';
        envelope.toAddress = 'support@company.in.salesforce.com';

        Test.startTest();
        CaseCreationEmailHandler handler = new CaseCreationEmailHandler();
        Messaging.InboundEmailResult result = handler.handleInboundEmail(email, envelope);
        Test.stopTest();

        // verify the handler returned success
        System.assertEquals(true, result.success, 'Handler should return success');

        // verify the Case was created
        List<Case> cases = [
            SELECT Id, Subject, Priority, ContactId, Origin
            FROM Case
            WHERE Subject = 'Urgent: System is down'
        ];
        System.assertEquals(1, cases.size(), 'One Case should be created');
        System.assertEquals('High', cases[0].Priority,
            'Subject contains "urgent" so priority should be High');
        System.assertEquals('Email', cases[0].Origin, 'Origin should be Email');

        // verify the Contact was linked
        Contact con = [SELECT Id FROM Contact WHERE Email = 'jane.tester@example.com'];
        System.assertEquals(con.Id, cases[0].ContactId,
            'Case should be linked to the matching Contact');

        // verify the attachment was saved
        List<ContentDocumentLink> cdls = [
            SELECT ContentDocumentId
            FROM ContentDocumentLink
            WHERE LinkedEntityId = :cases[0].Id
        ];
        System.assertEquals(1, cdls.size(), 'One attachment should be linked to the Case');
    }

    // test inbound email with no matching contact
    @isTest
    static void testInboundEmailHandler_unknownSender() {
        Messaging.InboundEmail email = new Messaging.InboundEmail();
        email.fromAddress = 'stranger@unknown.com';
        email.fromName = 'Unknown Person';
        email.subject = 'General inquiry';
        email.plainTextBody = 'I have a question about your services.';
        // no attachments -- leave binaryAttachments and textAttachments as null

        Messaging.InboundEnvelope envelope = new Messaging.InboundEnvelope();
        envelope.fromAddress = 'stranger@unknown.com';
        envelope.toAddress = 'support@company.in.salesforce.com';

        Test.startTest();
        CaseCreationEmailHandler handler = new CaseCreationEmailHandler();
        Messaging.InboundEmailResult result = handler.handleInboundEmail(email, envelope);
        Test.stopTest();

        System.assertEquals(true, result.success, 'Handler should succeed for unknown sender');

        List<Case> cases = [
            SELECT Id, ContactId, SuppliedEmail, SuppliedName
            FROM Case
            WHERE Subject = 'General inquiry'
        ];
        System.assertEquals(1, cases.size(), 'Case should still be created');
        System.assertEquals(null, cases[0].ContactId,
            'ContactId should be null for unknown sender');
        System.assertEquals('stranger@unknown.com', cases[0].SuppliedEmail,
            'SuppliedEmail should capture the sender address');
    }

    // test the email queue processor
    @isTest
    static void testEmailQueueProcessor() {
        Contact con = [SELECT Id FROM Contact LIMIT 1];

        // create a queue record
        Email_Queue__c eq = new Email_Queue__c(
            Contact__c = con.Id,
            Subject__c = 'Test Queued Email',
            Html_Body__c = '<p>Hello from the queue</p>',
            Plain_Body__c = 'Hello from the queue',
            Status__c = 'Pending',
            Retry_Count__c = 0
        );
        insert eq;

        Test.startTest();
        EmailQueueProcessor processor = new EmailQueueProcessor();
        processor.execute(null);  // pass null for SchedulableContext in tests
        Test.stopTest();

        // verify the queue record was updated
        eq = [SELECT Status__c FROM Email_Queue__c WHERE Id = :eq.Id];
        System.assertEquals('Sent', eq.Status__c,
            'Queue record should be marked as Sent');
    }
}
```

### Querying EmailMessage Records for Reporting

```apex
public with sharing class EmailActivityReporter {

    // query email activity for a specific record
    // requires Enhanced Email to be enabled
    // @param relatedRecordId: the record to query email activity for
    // @return List<EmailMessage>: sent emails related to the record
    public static List<EmailMessage> getEmailHistory(Id relatedRecordId) {
        return [
            SELECT Id, Subject, ToAddress, FromAddress,
                   MessageDate, Status, IsOpened, IsBounced,
                   FirstOpenedDate, LastOpenedDate, HtmlBody
            FROM EmailMessage
            WHERE RelatedToId = :relatedRecordId
            ORDER BY MessageDate DESC
            LIMIT 100
        ];
    }

    // find contacts with bounced emails for cleanup
    // requires Bounce Management to be enabled in Setup > Email > Deliverability
    // @return List<Contact>: contacts with bounced email addresses
    public static List<Contact> getBouncedContacts() {
        return [
            SELECT Id, Name, Email, EmailBouncedReason, EmailBouncedDate
            FROM Contact
            WHERE EmailBouncedDate != null
            ORDER BY EmailBouncedDate DESC
        ];
    }

    // count emails sent today against the daily limit
    // useful for monitoring before a batch email job
    // note: this queries EmailMessage records, not the actual limit counter
    // the actual remaining limit is available via OrgLimits in some contexts
    // @return Integer: approximate count of emails sent today
    public static Integer getApproxEmailsSentToday() {
        return [
            SELECT COUNT()
            FROM EmailMessage
            WHERE MessageDate = TODAY
              AND Incoming = false
        ];
    }
}
```

## Common Mistakes

1. **Calling `Messaging.sendEmail()` inside a loop** -- Each `sendEmail()` invocation counts against the 10-per-transaction limit, even if you pass only one message per call. With a list of 15 contacts, you hit the limit at contact 11. Fix: collect all `SingleEmailMessage` objects into a `List<Messaging.SingleEmailMessage>` and make ONE `Messaging.sendEmail(emailList)` call after the loop.

2. **Using `MassEmailMessage` when you need custom body logic** -- `MassEmailMessage` requires a template and does not support `setHtmlBody()` or `setPlainTextBody()`. Developers try to set a programmatic body and wonder why it's ignored. Fix: use `SingleEmailMessage` in a Batch Apex job with a batch size of 10 when you need programmatic email bodies for many recipients.

3. **Setting `setSaveAsActivity(true)` with a User Id as `targetObjectId`** -- Salesforce throws an error: "INVALID_ID_FIELD: saveAsActivity is true but targetObjectId is a User." Fix: set `setSaveAsActivity(false)` when sending to Users. Activity logging only works for Contacts and Leads.

4. **Not handling null on `InboundEmail.binaryAttachments` and `textAttachments`** -- Salesforce returns `null` (not an empty list) when there are no attachments. Iterating over a null list throws a `NullPointerException`. Fix: always null-check before iterating: `if (email.binaryAttachments != null) { for (... att : email.binaryAttachments) { ... } }`.

5. **Returning `result.success = false` in `InboundEmailHandler` for business logic errors** -- When you return `false`, Salesforce sends a bounce/failure reply to the sender, potentially exposing your custom error message and internal details. Fix: return `success = true` for any email that was validly received, even if your processing logic failed. Log errors internally and handle them through your own notification system.

6. **Expecting email templates to render merge fields in test context** -- In test methods, merge fields in email templates generally resolve to empty strings because the test transaction does not fully simulate the template rendering engine. Fix: test that `SendEmailResult.isSuccess()` returns true and that `Limits.getEmailInvocations()` shows the expected count. Do not assert on the rendered body content in test methods. If you need to test rendering, use `Messaging.renderStoredEmailTemplate()` which does resolve merge fields in test context (with proper whoId/whatId).

7. **Hardcoding OrgWideEmailAddress Ids** -- Ids differ across sandboxes and production. The code works in the dev sandbox and breaks everywhere else. Fix: always query `OrgWideEmailAddress` by `Address` or `DisplayName`, handle the empty-result case gracefully (fall back to running user email), and log a warning so you know the address isn't configured.

8. **Using a batch size larger than 10 in Batch Apex email jobs** -- If your `execute()` method creates one `SingleEmailMessage` per record and sends them all in one `Messaging.sendEmail()` call, a batch size of 200 means 200 messages in one call. While Salesforce will send them, you risk hitting the 5,000 daily email limit much faster and may encounter heap size issues with large attachments. Fix: use a batch size of 10. Each execute processes 10 contacts, builds 10 messages, and sends them in one call.

9. **Forgetting to change sandbox deliverability settings after a refresh** -- Sandboxes default to "System email only" on every refresh. Apex emails are silently dropped with no error. `SendEmailResult.isSuccess()` returns `true` because the email was accepted for queuing, but it never leaves Salesforce. Fix: immediately after every sandbox refresh, go to Setup > Email > Deliverability and set the access level to "All email."

10. **Setting both `setTemplateId()` and `setHtmlBody()` on the same message** -- When `setTemplateId()` is provided, the template's subject and body silently override anything from `setSubject()`, `setHtmlBody()`, and `setPlainTextBody()`. Your programmatic body is discarded without warning. Fix: choose one approach. If you need to modify template output, use `Messaging.renderStoredEmailTemplate()` to get the rendered body, modify it in Apex, then use `setHtmlBody()` WITHOUT `setTemplateId()`.

## Governor Limits Quick Reference

| Limit | Value | Notes |
|-------|-------|-------|
| `Messaging.sendEmail()` invocations per transaction | 10 | Each call counts as 1, regardless of how many messages are in the list |
| `SingleEmailMessage` recipients per message (toAddresses) | 100 | Across to + cc + bcc |
| `MassEmailMessage` recipients per message | 250 | Contact or Lead IDs only |
| Single emails per day (org-wide) | 5,000 | Based on license count; includes Apex, workflow, and approval emails |
| Mass emails per day | 1,000 | For orgs with fewer than 50,000 external email allocations |
| Attachment size per file | 3 MB | Applies to both `EmailFileAttachment` and `setEntityAttachments` |
| Total attachment size per email | 25 MB | Combined size of all attachments |
| Email body size (HTML) | 6 MB | Practical limit; varies by email client rendering |

## See Also

- [Email with Attachments](./email-with-attachments.md) -- detailed patterns for PDF attachment workflows with Purchase Orders
- [Apex Fundamentals](../apex-fundamentals/apex-fundamentals.md) -- governor limits, sharing keywords, and testing best practices
