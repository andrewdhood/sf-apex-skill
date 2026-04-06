# Chapter 9: Deploying

> **Source:** Mastering Salesforce DevOps by Andrew Davis (Apress, 2019)
> **When to read this:** When you need to choose deployment technologies, build deployment automation, resolve deployment errors, or evaluate commercial release management tools.

## Key Concepts

1. **Deployment is logistics, not development** -- the process of moving code and configuration from one environment to another. It should be fast, predictable, and automated.
2. **Salesforce CLI is the future** -- the Ant Migration Tool is deprecated; all new capabilities go through the Salesforce CLI (SFDX). Start there if building from scratch.
3. **Unlocked packages are the preferred deployment mechanism** for enterprise teams. They bundle metadata into versioned, installable units that eliminate hand-picking metadata.
4. **Deployment errors are the #1 time sink** in release management. Errors cascade, so always resolve from the top of the list down.
5. **Continuous delivery means small, frequent deployments** distributed across time and the team, not big-bang releases.
6. **Commercial tools (Copado, Gearset, Flosum, etc.) reduce pain** but should accelerate your shift to source-driven development, not replace it.
7. **Differential deployments** (deploying only changed files since the last successful deploy) keep deployments small and fast. Use Git tags to track successful deploy points.
8. **Configuration data** stored in Custom Settings or Custom Objects must be extracted, version-controlled, and loaded separately from metadata. Prefer Custom Metadata Types because they deploy via the Metadata API.
9. **Org differences always increase** unless you actively apply energy to keep environments in sync. Unlocked packages are the best tool for maintaining consistency across orgs.
10. **Dependency and risk analysis** tools (Panaya, Strongpoint) can assess the risk of proposed metadata changes before deployment. Research shows change approval processes do not improve stability and definitely decrease velocity.

## Detailed Notes

### Deployment Technologies Overview

When building a deployment process, understand the landscape of available tools:

- **Manual changes** (Setup UI): Only appropriate for very small teams or low-risk changes. Never scale.
- **Change Sets**: Salesforce's built-in deployment tool. Point-and-click, but limited -- can only deploy between related orgs (same production lineage), no deletions, no version history, dependencies are optional to include.
- **Metadata API**: The underlying API that all tools use. Direct usage requires building XML `package.xml` manifests. Powerful but verbose.
- **IDEs (VS Code)**: Use the Salesforce Extensions Pack. Good for individual developer deploys, not for CI/CD pipelines.
- **Salesforce CLI (SFDX)**: The flagship tool. Wraps the Metadata API with a usable interface. Supports JSON output for scripting, credential management, scratch org creation, source tracking, and package management.
- **Packages (1GP, 2GP, Unlocked)**: The most reliable deployment mechanism. Bundles metadata into versioned, installable units.

### Scripting Best Practices

When writing deployment scripts, always prefer Unix-style commands (Bash) over PowerShell for cross-platform compatibility. Key techniques:

- **Store command snippets in `package.json`** using the `scripts` section. This keeps commonly-used commands in version control, runnable via `npm run <scriptname>`, and always executing from the project root.
- **Use `--json` flag** on all SFDX commands and parse output with `jq`. This makes command output programmatically consumable.
- **Chain commands in Bash** using variables to pass output between steps. For example, query the default Dev Hub alias and use it as a `--targetusername` parameter for subsequent commands.
- **Use Node.js for sophisticated scripting** -- it allows importing NPM modules, using the Salesforce Core API (`@salesforce/core`), and building more complex logic than shell scripts allow.

### Salesforce CLI Key Capabilities

- Securely manages credentials for all orgs you access
- Creates and manages scratch orgs, packages, and projects
- Converts metadata between Metadata API format and Source format automatically
- Tracks metadata in target orgs for quick synchronization
- Provides anonymous Apex execution, SOQL queries, data loads
- Supports JSON output for parsing and chaining commands
- Built on OCLIF (Open CLI Framework) with a plugin architecture
- Plugins are developed in JavaScript/TypeScript using NPM libraries like `@salesforce/core` and `@salesforce/command`

### Free Salesforce Tools

- **CumulusCI**: The most sophisticated free tool. Built by the Salesforce.org team in Python. Automates release management, includes Robot Framework for Selenium testing.
- **SFDX-Falcon**: Full project templates for Salesforce DX. Optimized for ISV managed package builds.
- **Force-dev-tool**: In reduced maintenance mode. Appirio DX uses it internally for parsing Salesforce XML metadata.
- **Salesforce Toolkit** (by Ben Edwards): Tools for comparing org permissions and other common challenges. Runs on Heroku, source is available to fork.

### Commercial Salesforce Tools

When evaluating commercial tools, always ensure you give licenses to all developers and admins -- not just a few release managers. Bottlenecks defeat the purpose of DevOps.

**Copado** (Released 2013)
- Architecture: Salesforce frontend + Heroku processing engine
- Includes its own ALM tool built on Salesforce, Selenium test recorder, compliance tool
- Native integration with Jira, Azure DevOps, VersionOne, Rally
- Priced per user per month with developer and release manager tiers
- Benefits: Nice metadata picker, rich tool suite including data migrations and Selenium testing
- Drawbacks: Logs stored as Salesforce attachments (hard to read), UI looks slightly awkward, Heroku jobs have nontrivial startup time

**Gearset** (Released 2016)
- Architecture: .NET and C# on AWS (SaaS)
- Founded by Redgate Software (enterprise DB tooling background)
- Intelligent comparison engine that auto-fixes common deployment issues (missing dependencies, obsolete flows) before pushing changes
- Per-user pricing with unlimited comparisons and deployments
- Benefits: Fast UI, metadata comparisons (org-to-org, org-to-git, git-to-org, branch-to-branch), auto-fix of common errors, scratch org support, hierarchical data migration
- Drawbacks: No Selenium testing, UI not customizable, cannot mix in third-party DevOps tools

**Flosum** (Released 2014)
- Architecture: Built entirely on the Salesforce platform (native Apex)
- Highest number of positive AppExchange reviews
- No additional security reviews needed (it lives on Salesforce)
- Has built basic version control and CI/CD capabilities, plus Git integration
- Benefits: Business logic customizable using Salesforce mechanisms, no additional platforms to whitelist
- Drawbacks: Large operations are very slow (native Apex constraints), expensive, cannot integrate standard DevOps tools

**AutoRABIT** (Released 2014)
- Architecture: Built on OpenStack using Java; SaaS or on-premise
- 40+ Fortune 500 customers, strong in regulated industries (finance, healthcare)
- Connects multiple orgs, captures metadata differences, supports Salesforce DX and scratch orgs
- Includes DataLoader Pro for hierarchical data migration, Selenium integration, Vault backup product
- Benefits: On-premise option for security-sensitive industries, prebuilt integrations (Jira, CheckMarx, test automation)
- Drawbacks: Long implementation time (typically a month), clumsy metadata picker, UI not customizable

**ClickDeploy** (Released 2017)
- Architecture: AWS (SaaS)
- Easiest to get started with; free tier allows 15 deploys/month
- OAuth-based authentication to any number of Salesforce orgs
- Supports Salesforce DX source format, scheduled Git backups, automated deployments from Git on push
- Benefits: Easy to start, free tier, nice metadata selection, metadata comparisons, Git integration with admin-friendly UI
- Drawbacks: No scratch org or package automation, no data migration, no Selenium, limited team access controls

**Blue Canvas** (Released 2016)
- Architecture: AWS, Auth0, Go, Git, Salesforce DX
- Takes regular metadata snapshots of connected orgs and records changes in Git with attribution
- Supports both "defensive version control" (passively tracking admin changes) and "offensive version control" (auto-deploying tracked changes)
- Benefits: Git built in, fast metadata comparisons, near-real-time change tracking
- Drawbacks: Doesn't track profiles/permission sets in main tool, no data migration or Selenium

**Metazoa Snapshot** (Released 2006)
- Architecture: Desktop app written in Visual C++
- Visual workspace concept with graphical pipeline views from dev to production
- Automatically batches metadata retrieval/deployment, bypassing the 10,000 item Metadata API limit
- Over 40 reports including "generate a data dictionary"
- Supports data extraction/loading with relationship preservation and data scrambling
- Benefits: Quick to install, many reports not found elsewhere, supports SFDX metadata format
- Drawbacks: Old-looking UI, no scratch org or package automation, desktop-only (not cloud-based)

### Packaging

#### Classic Packaging (1GP)

- **Unmanaged packages**: Same technology as change sets. Metadata is no longer associated with the package after installation -- like a shipping container discarded after opening. Useful for deployment but not for modularizing architecture.
- **Managed packages**: Require a namespace prefix (e.g., `myMgdPkg__packageContents__c`). Must be developed and published from a Developer Edition org. Code is hidden from the installation org. Upgradeable. Package metadata has its own governor limits above and beyond the installation org.
- Unlike change sets, packages automatically include dependencies because they must be self-contained and installable in any org.
- Commercial AppExchange apps are almost always managed packages; free apps are almost always unmanaged.

#### Second-Generation Packaging (2GP)

- Designed for source-driven development (unlike 1GP which is org-based)
- **Unlocked packages** are for enterprise use (building and migrating functionality within a single enterprise)
- **2GP managed packages** are the successor to classic managed packages for ISVs
- Defined using configuration files, published via Salesforce CLI, can express dependencies on other packages and org-level features/settings
- A single Dev Hub can be associated with multiple namespaces; scratch orgs can use any of those namespaces
- 2GP managed packages can be published to the AppExchange, unifying the ISV and enterprise workflows

#### Unlocked Packages -- The Preferred Approach

Unlocked packages solve the core problems of deploying unpackaged metadata:

1. Eliminate the burden of hand-selecting metadata for each deployment
2. Avoid error-prone Salesforce-specific XML merging across developers
3. Ensure consistent deployment across all environments
4. Metadata remains associated with the package (unlike unmanaged packages)
5. Package deployments cannot overwrite metadata in another package

**To build and publish unlocked packages:**

1. Enable packaging in the Dev Hub
2. Configure `sfdx-project.json` with `packageDirectories` pointing to your metadata folders
3. Create the package on your Dev Hub: `sfdx force:package:create` (generates a 0Ho ID)
4. The `packageAliases` and `packageDirectories` sections in `sfdx-project.json` are updated automatically
5. Create package versions: `sfdx force:package:version:create` (generates a 04t ID used to install)

Script version publishing as part of your CI process. Use the `--branch` flag to auto-set based on the Git branch. Add Git tags when new versions are published.

**Adding and removing metadata from packages:**

- Unlocked packages can take ownership of existing metadata already in an org. When you install a package containing `MyObject__c`, it takes ownership of that object.
- Do NOT add a namespace to unlocked packages if you want them to take ownership of existing metadata (namespace would change the API name).
- Automate checks to ensure no overlap between metadata in different packages and unpackaged metadata.
- To remove metadata: it gets deprecated (not deleted) if it contains data (custom fields, objects). This prevents data loss. Use `sfdx force:package:version:install --upgradetype DeprecateOnly` to safely deprecate.
- To migrate metadata between packages: move files from one package directory to another, publish new versions of both, install the source package with `DeprecateOnly`, then install the target package (which assumes ownership).

### Resolving Deployment Errors

Deployment errors are extremely common in legacy Salesforce projects. Even with Salesforce DX, they remain a fact of life.

**General approach to debugging deployment errors:**

1. **Do not panic.** Large deployments can produce hundreds of errors. Many are related -- resolving one root cause can fix dozens of dependent errors.
2. **Copy errors into a spreadsheet** if your tool does not organize them for you.
3. **Errors cascade.** A field deployment failure can cause a class failure, which causes a Visualforce page failure. Work from the top of the list down.
4. **Delete duplicate errors** first -- they all resolve the same way.
5. **Delete dependent errors** -- errors caused by earlier errors in the list.
6. **Work top-down.** Note file names, metadata names, and line numbers in error messages.
7. **Comment out problematic metadata lines** temporarily to get the main deployment through.
8. **For large deployments, temporarily remove persistent error items**, deploy the main body, then iterate quickly on the isolated problematic items.
9. **"An unexpected error occurred" (gack) errors** reflect internal Salesforce exceptions. File a case with Salesforce to look up the error number in Splunk. Meanwhile, isolate the problem by deploying metadata in subgroups.

**Tips for reducing deployment errors:**

- Deploy small batches frequently
- Use feature branches with validations against the next higher org (e.g., QA)
- If a feature branch validates successfully, it will likely deploy successfully when merged to the main branch
- Use Salesforce DX scratch orgs to eliminate hand-picking metadata entirely -- push and pull from scratch orgs in their entirety

**Getting help:** The Salesforce Stack Exchange (`salesforce.stackexchange.com/questions/tagged/deployment`) is the best resource for obscure deployment errors.

### Continuous Delivery

Continuous delivery is the ability to get changes of all types into production safely, quickly, and sustainably. It is a maturation from ad hoc deployments to deployments happening on an ongoing basis.

**Key principle:** Separate deployments from releases. You can practice continuous delivery even if features should not be immediately released to users.

**Why continuous delivery matters:**

- Deploying to QA on every trunk commit ensures testers always have the latest code, and immediately know if something is broken
- Validating (check-only deploy) against UAT on every trunk commit ensures code is deployable to UAT at all times
- Batching deployments into infrequent releases leads to massive, error-prone deployments with one person losing half a day resolving errors under time pressure

**Automating deployments:**

- Most commercial release management tools offer automated deployments from version control
- For custom CI/CD, use Git tags to track successful deployments per environment
- Tag format: `[orgname]-[timestamp]` (e.g., `uat-20180626172404`)
- Use `git describe --tags --match "uat-*" HEAD` to find the last successful UAT deploy
- Use `git diff --name-only --ignore-all-space [tag]` to find changed files since then
- Copy changed files to a new directory for a differential deployment
- Using Source format (not Metadata API format) allows deploying individual fields rather than entire `.object` files

**Deploying configuration data:**

- Use Custom Metadata Types whenever possible -- they deploy via the Metadata API like any other metadata
- Custom Settings and Custom Object data require extraction from one org and loading into another (data migration)
- Store configuration data and the extraction/loading scripts in version control
- Transform data if it includes IDs or org-specific values
- Commercial tools like AutoRABIT, Copado, Gearset, and Metazoa have built-in data migration capabilities
- For massive configuration data (like CPQ products), specialized tools like Vlocity Build exist

**Continuous delivery rituals:**

- Code is developed on a single trunk with feature branches not persisting more than a day
- Every commit to trunk triggers a set of automated tests
- If the build breaks, the team's highest priority is to fix it within 10 minutes (fix forward or revert)
- Treat a green build as sacred -- never commit on top of a broken build
- Everyone should refrain from pushing to trunk while the build is broken; swarm to help resolve it

### Deploying Across Multiple Production Orgs

- Sandboxes are all related to a single production org; deploying across them is about making orgs more consistent
- Multiple production orgs are intentionally different -- they serve different business units
- **Unlocked packages are the only recommended approach** for syndicating functionality across multiple production orgs. They prevent configuration drift.
- Prior to unlocked packages, there was no reliable way to keep multiple production orgs in sync

**Managing org differences:**

- Differences between Salesforce orgs always increase unless you actively invest energy to keep them in sync (second law of thermodynamics corollary)
- Significant differences are either intentional (temporary or long-term) or unintentional
- Temporary differences = features being promoted through environments; long-term differences = org-specific configuration (endpoints, email addresses, etc.)
- Use Custom Metadata records with org-specific lookups via `UserInfo.getOrganizationId()` in Apex or `{!$Organization.Id}` in formulas to handle per-org configuration
- When org-level metadata must differ, use XSLT or higher-level language scripts (Node, Python, Perl) to search-and-replace values during deployment (e.g., replacing sandbox usernames with production usernames in approval process XML)
- Salesforce auto-translates sandbox usernames for most metadata types (e.g., `myuser@myOrg.com.dev` becomes `myuser@myOrg.com.qa`), but this does not work for org-wide email addresses or reports shared to particular users

### Dependency and Risk Analysis

- Tools like Panaya and Strongpoint assess metadata dependencies and rate proposed changes based on their potential risk to the org
- For example, adding a validation rule on a heavily-used field could interfere with other logic if not well-tested
- The 2018 State of DevOps Report found that change approval processes have not been shown to increase org stability and definitely decrease deployment velocity -- even for selective approval processes that only apply to high-risk changes
- The most useful step to limit deployment risk: track each change in version control and make frequent small deployments so the impact of any single deployment is minimized and problems can be easily diagnosed

## Key Takeaways

The Salesforce CLI is the future of Salesforce deployment tooling -- invest in learning it and building scripts around it rather than the deprecated Ant Migration Tool. Unlocked packages are the preferred deployment mechanism for enterprise teams because they eliminate hand-picking metadata, ensure consistent deployments, and prevent cross-package overwrites. When resolving deployment errors, always work from the top of the error list down because errors cascade -- fixing one root cause often resolves dozens of dependent errors. Continuous delivery (small, frequent, automated deployments) is vastly superior to batched releases because it distributes risk and gives developers immediate feedback on the quality of their metadata. When choosing commercial tools, ensure all team members (not just a few release managers) have licenses, and use the tools to accelerate your transition to source-driven development rather than as a crutch to stay in org-based workflows.
