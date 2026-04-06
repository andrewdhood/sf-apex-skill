# Chapter 2: Salesforce

> **Source:** Mastering Salesforce DevOps by Andrew Davis (Apress, 2019)
> **When to read this:** When you need to understand how the Salesforce platform's unique architecture affects development workflow choices, or when deciding between org development model vs. package development model.

## Key Concepts

- **"Salesforce" in this book means Salesforce Core** -- Sales Cloud, Service Cloud, Community Cloud, and the Lightning Platform. Not Marketing Cloud, Commerce Cloud, Heroku, or other acquired products, which have entirely different development and deployment methods.
- **Salesforce is both SaaS and PaaS** -- it handles everything from networking to runtime (SaaS for CRM), while also providing a platform for custom code (PaaS via Lightning Platform).
- **Almost no traditional DevOps tools work directly for Salesforce** -- tools designed for AWS/Azure/IaaS infrastructure management solve different problems than Salesforce configuration management.
- **Salesforce DX has two core principles**: version control as the source of truth (via scratch orgs), and modular architecture via second-generation packages.
- **Dev Hub** is the gateway to scratch orgs and unlocked packages -- enable it in a production org with zero risk.
- **Scratch orgs** are temporary, disposable environments created from version control (default 7 days, max 30 days).
- **Unlocked packages** solve the problem of tens of thousands of unpackaged metadata items existing as an undifferentiated collection in enterprise orgs.
- **SFDX source format** decomposes large monolithic metadata files into granular, version-control-friendly files.
- **The Salesforce CLI** is the primary automation interface, built on OCLIF, supporting JSON output for scripting.

## Detailed Notes

### "Salesforce" vs. Salesforce (Scope Boundaries)

When applying DevOps practices from this book, only apply them to Salesforce Core (Sales Cloud, Service Cloud, Community Cloud, Lightning Platform). Marketing Cloud, Commerce Cloud, and other acquired products are developed and deployed in fundamentally different ways. Sharing data between these products and Core requires integration work.

### How Salesforce Differs from Traditional Platforms

When comparing Salesforce development to traditional development, understand the key distinction: on traditional platforms (IaaS), every aspect of infrastructure can be represented as code (JSON for AWS, Terraform configs, etc.). On Salesforce, you are working at a much higher abstraction level -- application configuration rather than infrastructure provisioning. This means:

- You never manage servers, databases, middleware, or networking directly.
- The challenge is not "infrastructure as code" but "configuration as code" -- ensuring your custom objects, fields, Apex, Flows, layouts, and other metadata are consistently represented and deployable.
- Almost none of the existing DevOps tooling ecosystem (Docker, Kubernetes, Terraform, Ansible) is relevant to Salesforce. The Salesforce-specific tools (Salesforce CLI, Metadata API) are what you work with.

### The Dev Hub

When setting up Salesforce DX for a team, enable Dev Hub in your production org first. This carries zero risk and zero side effects -- scratch orgs created from it carry no data or metadata from the production org. For security, use the "Free Limited Access License" to give developers or consultants Dev Hub access without exposing any production data or metadata. Each developer also needs appropriate permissions on the Dev Hub (detailed in Chapter 6).

Developer Edition orgs (including Trailhead orgs) can serve as Dev Hubs for training purposes, but their limits are too restrictive for production use.

### Scratch Orgs

When choosing development environments, prefer scratch orgs over sandboxes for source-driven development because they are populated entirely from version control. Key characteristics:

- Default lifespan of 7 days (configurable up to 30 days).
- Used for development, testing, and CI.
- Sandboxes remain useful as long-running test environments.
- Packages and metadata developed in scratch orgs can be deployed to sandboxes and production using CI/CD.

### Second-Generation Packaging (Unlocked Packages)

When organizing a large Salesforce org, use unlocked packages to break the undifferentiated mass of metadata into discrete, independently deployable modules. Key distinctions:

- **Unmanaged packages**: Act like cardboard shipping containers -- once unpacked, the package structure is gone and the metadata is loose.
- **Managed packages**: Hide IP, prevent modification, used by ISVs. Creating and updating them is complex (separate Dev Edition org, namespaces, etc.), so enterprises rarely use them for internal work.
- **Unlocked packages** (new with Salesforce DX): Do not hide contents, allow modification after installation, designed for enterprise use. The most practical path for organizing internal customizations.
- **Second-generation managed packages**: More flexible successor to managed packages for ISVs. Methods for creating/deploying are nearly identical to unlocked packages.

When adopting unlocked packages, do not attempt to convert an entire org into a single massive package at once. Start with one small, simple package and ensure you can deploy it across all your orgs before expanding.

### Metadata API vs. SFDX Source Format

When choosing a project format, use SFDX source format because it decomposes large monolithic XML files into granular files that work well with version control:

- Traditional Metadata API format stores entire objects (with all fields, layouts, list views) in single `.object` files that can be tens of thousands of lines long, causing constant merge conflicts.
- SFDX source format breaks these into individual files per field (`Email__c.field-meta.xml`), per layout, per list view, etc., with `-meta.xml` suffixes.
- Changes can be retrieved from sandboxes using either `source:retrieve` (SFDX format) or `mdapi:retrieve` (Metadata API format).

### Salesforce CLI

When automating Salesforce development tasks, use the Salesforce CLI (`sf` / `sfdx`) as the primary interface. Key capabilities:

- Call common Salesforce APIs from the command line.
- Synchronize source with scratch orgs via `source:push` and `source:pull`.
- Output in JSON format for scripting and CI integration.
- Can be used directly on the command line, in CI scripts, and is also the engine behind the VS Code extensions.

## Key Takeaways

Salesforce is unique among development platforms because it operates at a much higher abstraction level than traditional software -- you configure applications rather than provision infrastructure. This means traditional DevOps tooling is irrelevant, but the principles still apply. Salesforce DX bridges this gap by introducing scratch orgs (making version control the source of truth) and unlocked packages (enabling modular architecture). When adopting these capabilities, start small -- enable Dev Hub (zero risk), begin with one unlocked package, and use SFDX source format to make your metadata version-control-friendly.
