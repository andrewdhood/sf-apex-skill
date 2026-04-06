# Chapter 4: Developing on Salesforce

> **Source:** Mastering Salesforce DevOps by Andrew Davis (Apress, 2019)
> **When to read this:** When setting up a Salesforce development workflow, deciding what metadata to track in version control, choosing between declarative and code-based development, or understanding the constraints that shape how you build and deploy on the platform.

## Key Concepts

1. **Metadata is everything.** All Salesforce customizations -- objects, fields, code, layouts, flows -- are represented as metadata components (mostly XML, some JSON) that can be retrieved, stored in version control, and deployed between orgs. This is the foundation of any CI/CD pipeline.

2. **Two metadata formats exist.** The native Metadata API format uses a flat `src/` folder with `package.xml`; the Salesforce DX source format decomposes large files (like `.object`) into per-component subfolders. Always prefer SFDX source format for version control because it makes Git diffs and collaborative merges tractable.

3. **Not all metadata belongs in your repo.** Metadata "controlled by developers and admins" goes in version control. Metadata "controlled by end users" (reports, dashboards, documents) generally does not, unless it is a dependency for your code or config.

4. **Never change DeveloperNames.** Renaming an object or field API name creates a brand-new metadata item on deployment -- the old one stays behind, all data is orphaned, and every reference (code, formulas, integrations) breaks. Change the *label* instead.

5. **Profiles are a nightmare for version control.** A single Profile retrieve overwrites the entire file. Use Permission Sets as your primary access control mechanism -- they are granular, composable, and far easier to track and deploy.

6. **Schema must be in version control alongside code.** Code references objects and fields. If you deploy code to an environment where the schema does not exist, the deployment fails. Always track custom objects, fields, validation rules, and record types.

7. **Governor limits are a design constraint, not a bug.** Apex runs in a multitenant environment. Salesforce enforces limits on SOQL queries, DML statements, CPU time, and heap size per transaction. Write code that respects these limits from day one.

8. **Apex classes and triggers carry an API version.** The version is stored in the `-meta.xml` sidecar file. Periodically review and upgrade to the latest API version, but run your unit tests afterward because version changes can surface compilation errors or behavior changes.

9. **LWC over Aura over Visualforce.** Lightning Web Components are the current standard for client-side UI. They use native Web Components, run faster than Aura, and are the direction Salesforce is investing in. Aura is still supported but adds overhead. Visualforce is server-rendered and slow.

10. **Declarative changes are still metadata.** Flows, Process Builder automations, validation rules, and App Builder pages all produce metadata files. They must go through the same version control and deployment pipeline as code.

## Detailed Notes

### Development Environment and Tools

- **Use VS Code as your primary IDE.** Salesforce has deprecated the Force.com IDE and exclusively supports VS Code with the Salesforce Extensions. VS Code has built-in Git support, terminal, and the Apex Debugger / Replay Debugger.
- **Be aware of VS Code's sandbox limitation:** as of the book's writing, VS Code does not warn you when deploying changes that will overwrite others' work in a shared sandbox. Use scratch orgs for isolated development, or coordinate carefully.
- **Commercial IDEs fill real gaps.** The Welkin Suite and Illuminated Cloud provide features like advanced code completion, replay debugging, and task management that the free Salesforce extensions lack. Evaluate them if your team's productivity justifies the cost.
- **Developer Console is useful for quick debugging and anonymous Apex** but is not a substitute for a local IDE with version control integration.

### Metadata: What It Is and How It Works

- **What is metadata:** Files (mostly XML, some JSON) representing every customization in a Salesforce org -- objects, fields, classes, triggers, layouts, flows, applications, permission sets, and roughly 250 other types.
- **Metadata API format vs. SFDX source format:**
  - Metadata API: flat `src/` directory, one file per metadata item, grouped by type. A single `.object` file can contain hundreds of child items (fields, validation rules, record types). Git merges on these files are extremely painful.
  - SFDX source format: decomposes large files into folders with one file per child component. This makes Git diffs and merges practical. SFDX format does not use `package.xml` -- it is auto-generated when communicating with the Metadata API.
- **Conversion commands:**
  - `sfdx force:mdapi:convert` -- converts Metadata API format to SFDX source format
  - `sfdx force:source:convert` -- converts SFDX source format back to Metadata API format (lossy: subfolder groupings are lost)
- **Sidecar files** (e.g., `MyClass.cls-meta.xml`) store companion metadata like API version. They are small XML files that travel alongside the main metadata file.

### What Metadata to Exclude from Version Control

When deciding what to track, ask: "Is this controlled by developers/admins, or by end users?" Add excluded types to `.forceignore` to prevent them from sneaking back in.

**Exclude these types (unless they are code/config dependencies):**

| Type | Reason |
|------|--------|
| Certificate | Environment-specific, should be kept secure |
| ConnectedApp | Environment-specific; requires dynamic parameter substitution for automation |
| Dashboard | User-controlled, changes frequently |
| Document | User-controlled |
| EmailTemplate | User-controlled (exception: VisualForce email templates with embedded code) |
| InstalledPackage | Cannot control installation order; use `sfdx package` commands instead |
| Layout | Problematic in package model; manage at org level for objects that straddle packages |
| NamedCredential | Environment-specific; requires parameter substitution |
| PlatformCachePartition | Environment-specific |
| Profile | Extremely difficult to manage; prefer Permission Sets |
| Report | User-controlled |
| SamlSsoConfig | Environment-specific |
| Site | Binary file that changes unpredictably |

### Retrieving and Deploying Changes

- **Retrieving is tricky in Metadata API format.** Retrieving a single field from an object will overwrite the entire `.object` file locally, wiping out other field definitions. This is why specialized tools and the SFDX source format exist.
- **Profiles are the worst offender.** Retrieving a profile returns only "user permissions." To get field-level security, you must also retrieve the field and the profile together -- which then overwrites other permissions in the profile file. Retrieving page layout assignments requires retrieving the profile, layout, and record type simultaneously.
- **Use specialized tools.** The Metadata API was not designed for version-control-friendly workflows. Salesforce IDEs, release management tools, and the SFDX source synchronization process all solve these retrieval/merge problems.
- **SFDX source sync with scratch orgs** uses timestamps and hashes to push only changed files, making deployments fast and eliminating the need to manually specify which files to deploy.
- **Always verify your local file deploys successfully before committing.** It is possible to commit invalid metadata to version control, causing downstream deployment failures.

### Manual Changes and Untrackable Config

- **Some changes cannot be represented in metadata.** When manual configuration is required, document the exact steps in your project tracking tool. Make the instructions detailed enough for anyone to follow.
- **The SFDX source synchronization process** (available for scratch orgs, and expanding to sandboxes) automatically detects what was changed through the Setup UI and downloads the corresponding metadata. This dramatically reduces the need to manually track changes.

### Click-Based (Declarative) Development

- **There are roughly 250 types of declarative metadata** configurable through the Salesforce UI -- from simple settings to complex Builders (App Builder, Flow Builder, Schema Builder, Community Builder).
- **Even code-focused developers must understand declarative tools** because the platform is built around them. Configuration changes produce metadata that must flow through the same CI/CD pipeline as code.
- **Make configuration changes in the UI, make code changes in the IDE.** This keeps metadata coherent (the UI enforces validation) while code benefits from local tooling and version control.

### Declarative Builders and Their Metadata

- **Lightning App Builder** produces `FlexiPage` metadata -- a single XML file per page containing layout, components, and properties. Easy to version and deploy.
- **Community Builder (Experience Builder)** creates entire multipage sites. Until recently, Community metadata was not human-readable. The `ExperienceBundle` metadata type (stored as JSON) is changing this, but was in Developer Preview at time of writing. Until `ExperienceBundle` is GA, teams must deploy the entire `Site` (a binary file) or manually migrate configuration between orgs.
- **Process Builder and Flow Builder** both produce `Flow` metadata:
  - Since Winter '19, only the active Flow version is retrieved -- `FlowDefinition` files are no longer needed.
  - Flow versions provide declarative version control: you can roll back to a previous version from the Setup UI regardless of when it was deployed.
  - Flows and Processes are asynchronous (they store state in a `FlowInterview` object), so deploying a new version does not interrupt users mid-flow.
  - When deploying Flows/Processes as Active, Salesforce enforces code coverage requirements -- you must have Apex tests that exercise them, similar to (but softer than) the 75% line coverage required for Apex classes.
  - Enable active deployment at **Setup > Process Automation Settings > Deploy processes and flows as active**.

### Data Management

#### Schema in Version Control
- **Always track your schema** (custom objects, fields, validation rules, record types) in version control. Code depends on schema. Deploying code without its schema dependencies will fail.
- **Do not allow manual schema changes in test or production environments** unless you have a high tolerance for confusion. Modify objects and fields in a dev environment and deploy through your pipeline.
- **SFDX source format decomposes `.object` files** into a folder with one file per field, validation rule, record type, etc. This makes collaborative schema management practical with Git.

#### Changing the Schema Safely
- **Increasing a field's length** is usually safe, but decreasing it can truncate data. Also check external integrations that may reject longer values.
- **Adding validation rules** is a business logic change -- test carefully because it can break Apex tests and integrations that create/update records.
- **Disabling field history tracking** deletes historical data (unless you use Salesforce Shield).
- **Making a field required** can break Apex tests and integrations, and is not a deployable change unless the field is populated for every record in the target org.

#### Never Change DeveloperNames
- When you deploy a renamed field/object, Salesforce treats it as an entirely new item. The old one remains, data is not migrated, and all code/formulas/integrations referencing the old API name break.
- **Instead:** change the field's *label* (display name) in the UI. Leave the API name (`DeveloperName`) as-is. Add a description noting the rename if needed.
- Salesforce follows this pattern themselves -- Einstein Analytics is still `Wave` in the metadata, Communities are still `Picasso` in some places.

#### Changing Field Types
1. Experiment in your dev environment first to see if the change is even permitted.
2. Assess data loss risk and get business stakeholder sign-off.
3. Check external integrations that connect to the field.
4. Try deploying the field type change via Metadata API. If it succeeds, you are done.
5. If deployment fails, you have two options: (a) manually change the field type in each environment, or (b) create a new field of the correct type and migrate data.
6. For heavily-used fields in production, consider creating a data synchronization process (Apex trigger, Process, or Workflow Rule) that keeps the old and new fields in sync during the transition, then retire the old field once migration is complete.

### Bulk Database Operations
- **Anonymous Apex** can process up to 10,000 records per execution (DML limit). Store your scripts in version control for audit trail. Run them through Developer Console or the Salesforce CLI.
- **Batch Apex** uses `Database.Batchable` interface with `start`, `execute`, and `finish` methods, processing 200 records per batch. Use this for anything exceeding 10,000 records.
- **Data backup matters.** Use the free monthly export at **Setup > Data > Data Export**, or invest in a backup service (OwnBackup, Odaseva, AutoRABIT Vault). Bulk operations, bad triggers, and malicious actions can all cause data loss.

### Configuration Data Management
- **Configuration data (records that drive application behavior) is different from metadata** and is often overlooked in CI/CD pipelines. Managed packages like Rootstock ERP, Vlocity, nCino, and Salesforce CPQ rely heavily on configuration data.
- When migrating configuration data between orgs:
  - Use external IDs (not Salesforce record IDs, which vary by org) to establish uniqueness and relationships.
  - Load parent records before child records.
  - Make loads idempotent by using upsert on external IDs -- this ensures re-running the load never creates duplicates.
  - The Salesforce CLI's `data:tree:export` and `data:tree:import` commands can handle structured data, but are limited to 200 records per call.

### The Security Model

#### Infrastructure Security
- Salesforce handles infrastructure security (servers, networks, data centers). Check `trust.salesforce.com` for status, certifications, and security protocols.

#### Login and Identity
- **Track IAM metadata in version control:** SecuritySettings, ConnectedApp, AuthProvider, SamlSsoConfig. These may need environment-specific parameter substitution during deployment.
- Changes to login and identity config should be tested in a sandbox first -- an errant admin change could break integrations or lock out users.

#### Admin Access
- **Be very selective with System Administrator profile.** Most users should get the Standard User profile. Admin access = "what you can do in Setup."
- **Prefer Permission Sets over Profiles** for managing access. Keep profile permissions thin and use permission sets to layer on additional access. This follows the DRY principle and makes access easier to version, deploy, and audit.
- Permission Set Groups allow you to bundle multiple permission sets and assign them to user groups (reduces manual assignment overhead).
- Muting Permission Sets (added to Permission Set Groups) are the first example of **negative permissions** on the platform -- they inhibit specific permissions that other sets in the group would grant.

#### User Access
- Permissions in Salesforce are **additive** -- users start with nothing and gain permissions from their profile, permission sets, and user settings.
- **Developers often forget to deploy permissions with their features.** A developer with System Administrator access can build and test without explicit permissions, then deploy to production where standard users cannot access the new feature.
- Use **"Login As"** functionality to test as target users in dev/test environments to catch permission gaps early.
- When deploying a new Permission Set, you must separately assign it to users -- assignment data cannot be deployed via metadata. Write anonymous Apex to query User IDs and create `PermissionSetAssignment` records.

### Server-Side Programming

#### Apex
- **Strongly typed, Java-like language** that compiles to Java behind the scenes. Cannot use third-party Java libraries. Runs only on the Salesforce platform.
- **Two kinds of Apex metadata:** triggers (fire on database events: insert, update, delete, undelete) and classes (more flexible, support methods, interfaces, global variables).
- **Governor limits enforce multitenant fairness:** heap size, CPU time, SOQL query count, DML statement count, and more. These are per-transaction limits. Design for them from the start.
- **API version matters.** Each class/trigger specifies an API version in its `-meta.xml` sidecar file. Periodically upgrade to the latest version for optimal performance, but run unit tests afterward -- version changes can introduce compilation errors or behavior changes.
- **Never hardcode record IDs** in Apex. IDs differ between environments. Use dynamic queries or Custom Metadata Types to identify records.

#### Visualforce
- Server-rendered pages using HTML-like markup. Good for overriding standard record pages using Standard Controllers or Controller Extensions.
- **Falling out of favor:** slower than Lightning, state transferred back and forth on every action, Viewstate size limits make it unsuitable for large data editing. Still useful for specific scenarios (PDF generation, email templates with embedded logic).

#### Anonymous Apex
- Not stored in the org -- runs on demand via Developer Console or CLI.
- Excellent for **pre/post-deployment automation:** creating scheduled jobs, assigning permission sets, modifying User Role assignments, triggering batch jobs, and one-time data fixes.
- **Always save anonymous Apex scripts in your code repository** for reuse and audit trail, even though they are not persisted in Salesforce.

### Client-Side Programming

#### Lightning Web Components (LWC)
- Based on the open Web Components standard -- native browser execution, no framework overhead.
- Salesforce rewrote their own Lightning UI in LWC before announcing it publicly, proving the technology at scale.
- LWC and Aura components can coexist on the same page.
- **Preferred for all new client-side development.**

#### Lightning Aura Components
- Based on the open-source Aura framework (inspired by AngularJS). Requires browser compilation into native JavaScript, adding execution overhead.
- Aura runs in a separate domain from Visualforce for security -- prevents cross-domain data scraping.
- Still supported but being superseded by LWC.

#### Other Client-Side Options
- JavaScript Remoting, Visualforce Remote Objects, and S-controls are legacy options. S-controls are deprecated.
- **JSForce** is the most significant external JavaScript library for Salesforce -- it powers the Salesforce CLI itself and many other Node.js/TypeScript tools.
- Third-party libraries (Restforce for Ruby, Simple Salesforce for Python) provide REST API wrappers for external integrations.

## Key Takeaways

1. **Track schema alongside code in version control** -- code depends on schema, and deploying one without the other will fail. Use SFDX source format to decompose large `.object` files into Git-friendly individual component files.

2. **Never rename DeveloperNames (API names) for fields or objects.** Salesforce treats a renamed API name as an entirely new item on deployment, orphaning data and breaking all references. Change the label instead.

3. **Use Permission Sets instead of Profiles** for access control. Profiles are extremely difficult to manage in version control due to Metadata API retrieval behavior, and they do not provide the right granularity. Permission Sets are composable, deployable, and auditable.

4. **Exclude user-controlled metadata (reports, dashboards, documents) from your CI/CD pipeline** unless they are dependencies for your code. Add them to `.forceignore`. Focus your pipeline on metadata that is "controlled by developers and admins."

5. **Design for governor limits from day one** when writing Apex. These limits (SOQL queries, DML statements, CPU time, heap size) are per-transaction constraints in a multitenant environment -- they are not negotiable and must shape your architecture, not be bolted on after the fact.
