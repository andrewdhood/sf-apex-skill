# Chapter 6: Environment Management

> **Source:** Mastering Salesforce DevOps by Andrew Davis (Apress, 2019)
> **When to read this:** When setting up or rationalizing Salesforce development environments, deciding between scratch orgs and sandboxes, configuring Dev Hub and API authentication, planning org refresh cycles, or dealing with multiple production orgs.

## Key Concepts

- Salesforce orgs come in several types: production orgs (Enterprise, Unlimited, Professional, Group, Developer Edition), sandboxes (Developer, Developer Pro, Partial Copy, Full Copy), and scratch orgs. Each has different limits, data capacities, and refresh intervals.
- The fundamental environment decision is whether to develop in sandboxes (org-based development) or scratch orgs (source-based development). Scratch orgs are strongly preferred for new development because they enforce version control discipline, but sandboxes remain necessary for integration testing, training, staging, and complex orgs.
- Every environment adds complexity. Always use the minimum number of long-lived orgs possible and keep connections between them simple.
- Branching strategy and environment strategy must harmonize -- every long-running branch implies a corresponding environment.

## Detailed Notes

### Org Types and Editions

When choosing a Salesforce edition, understand that the `edition` in a scratch org definition file corresponds to the production org edition. Match the scratch org edition to your target org because features and limits differ between editions. For example, if you build features relying on Work.com capabilities available in a Developer Edition org, never expect that to work in an Enterprise org that does not have Work.com installed.

The `features` option in a scratch org definition generally corresponds to paid features not available in every org (e.g., `SalesWave`, `Communities`, `ServiceCloud`). The `settings` option corresponds to org configuration that can be defined through features and settings but may need to be enabled (e.g., Omni-Channel, encrypted storage preferences).

### Developing in Sandboxes (Org-Based Development)

When developing in sandboxes, understand that the sandbox itself is the source of truth. If all development is done in a single shared Developer sandbox, conflicting changes are recognized quickly, which is a form of continuous integration. However, this also means your work can be overwritten by others.

Never refresh the development org unless the team is disciplined about saving all work to version control first, because refreshing destroys everything not already saved or deployed. Most teams take the path of least resistance and never refresh, which causes the sandbox to drift further from production over time and accumulate cruft -- duplicate fields, half-finished customizations, abandoned experiments.

When using independent developer sandboxes or isolating teams into separate sandboxes, always integrate changes frequently. Developing in isolation for weeks or months causes conflicts to accumulate while developers gradually forget the detailed reasons behind each change, making integration painful and risky. Delaying conflict resolution until close to go-live is far riskier than resolving conflicts daily.

Even if a team uses version control to deploy changes to test and production orgs, many changes in the development org may never be tracked in version control. Some changes that are critical for functionality cannot be tracked in version control at all. This makes it difficult to debug problems when deploying features that have been under development for a long time.

### Developing in Scratch Orgs (Source-Based Development)

Always prefer scratch orgs for new development because they enforce version control as the source of truth. The short lifespan of a scratch org (up to 30 days, but typically recreated much more frequently) is a feature, not a bug -- it forces developers to persist their work in version control rather than relying on the org.

Scratch orgs do not inherit any data, metadata, features, or preferences from the Dev Hub that creates them. They are created entirely from configuration stored in version control (the scratch org definition file). This means there is no ambiguity about what features, settings, packages, or metadata are required to create an application.

Scratch orgs support "source synchronization" -- use `sfdx force:source:pull` and `sfdx force:source:push` to synchronize metadata between version control and your org. With org-based development, developers must use metadata retrieve/deploy commands, which requires deep expertise in the different types of Salesforce metadata. Source synchronization eliminates this complexity.

When developing integrations with external systems in scratch orgs, never try to connect the scratch org directly to external test systems. Instead, practice "design by contract" by writing Apex Mocks to simulate requests and responses from external systems. Then use a CI/CD process to migrate the service to an integrated sandbox for integration testing.

### Scratch Orgs vs. Sandboxes -- When to Use Each

Scratch orgs complement sandboxes but do not replace them. Use each for its strengths:

| Use Case | Recommended Environment |
|----------|------------------------|
| New feature development | Scratch orgs |
| Integration testing with external systems | Sandboxes with stable integrations |
| Testing production hotfixes | Sandboxes (refresh as needed) |
| Training with real data | Partial Copy sandbox |
| Staging / performance testing | Full Copy sandbox (requires Salesforce authorization for perf testing) |
| Code review / demos | Scratch orgs as "review apps" |
| Large data volume testing | Partial or Full Copy sandbox |

For teams working on complex orgs where scratch org setup is burdensome, consider a "sandbox cloning workflow" as a middle ground: establish an integration sandbox that receives deployments from all developers, then regularly clone it to recreate individual developer sandboxes. Track changes in version control and deploy to the integration environment on an ongoing basis.

### Org Access and Security

When working with sandbox user accounts, remember that sandbox usernames are autogenerated by appending the sandbox name to the source username. For example, `user1@production.org` becomes `user1@production.org.UAT` in a sandbox called UAT. Users can log in at `https://test.salesforce.com` using their production password with the sandbox-appended username.

Sandbox user accounts inherit the same profile, licenses, permission sets, and password as the source org. Users who log in via SSO may not know their production passwords, so SSO has to be set up separately for each sandbox.

Scratch orgs behave differently from sandboxes regarding security. The user who creates a scratch org automatically gets sysadmin access. Scratch orgs contain no metadata or data from a source org, and they contain only the single admin user account by default. You can create additional user accounts using `sfdx force:user:create` to test with different permission sets.

For full and partial copy sandboxes, always consider data security alongside production data security, because user permissions are copied from the source org and users will not be able to access data they could not access in production.

### The Dev Hub

Always designate a production org as your Dev Hub for business work, rather than using a personal Developer Edition org which only allows a small number of scratch orgs and packages. Enable Dev Hub by going to Setup > Dev Hub.

Never give developers Admin rights in the production org just to use the Dev Hub. Instead, add the necessary permissions (Create and delete scratch orgs, Create and update second-generation packages) to a permission set or profile.

When a package version is created, it starts in "beta" status and cannot be installed in production. To allow marking a package version as "Released," enable the system permission `Promote a package version to released`.

If developers should not have any access to the production org, use "Free Limited Access Licenses" which allow access to the Dev Hub but do not allow any data, login, or API access.

### API-Based Authentication

Always prefer OAuth 2.0 over username/password authentication for API access because OAuth permissions can be tuned, monitored, and revoked through a Connected App. Sharing your Salesforce username and password with a third-party app is fundamentally less secure.

For long-running integrations between Salesforce and another system, always create and use a dedicated integration user and perform authentications as that user. This ensures the connection is not disabled if the original user leaves the organization, and it enables monitoring of the integration user's activity.

The OAuth 2.0 flow for Salesforce works as follows:

1. A tool or service requests access to Salesforce on the user's behalf.
2. The user is redirected to Salesforce to log in.
3. Salesforce presents an authorization screen specifying which permissions are being requested.
4. If approved, Salesforce sends an Access Token to the application, subject to the permissions granted.
5. If "Perform requests on your behalf at any time" is approved, a Refresh Token is also issued, allowing the application to reconnect without re-authorization.

To view your authorized applications: Settings > My Personal Information > Connections. Admins can see all OAuth connections at Setup > Apps > Connected Apps > Connected Apps OAuth Usage.

### Salesforce DX Org Authorizations

The Salesforce CLI provides several authentication methods. Choose the right one for your context:

**`force:auth:web:login`** -- Use for manual developer work. Opens a browser tab to input credentials. Not used directly in automated processes, but by authorizing locally first, you can obtain an Auth URL for use in CI systems.

**`force:auth:sfdxurl:store`** -- Use for CI/CD automation. The Auth URL contains everything needed to access an org on a user's behalf (endpoint URL, OAuth Client ID, refresh token). The format is `force://clientId:clientSecret:refreshToken@instanceUrl`. To get the Auth URL:

1. Authorize the org locally with `sfdx force:auth:web:login -a OrgAlias`
2. Run `sfdx force:org:display -u <OrgAlias> --verbose` to display the Auth URL
3. Store the Auth URL as a secret in your CI system (never hardcode or commit it)
4. In the CI job, write the secret to a temporary file, run `sfdx force:auth:sfdxurl:store -f <file> -s -a MyDefaultOrg`, then delete the file

**`auth:jwt:grant`** -- Use for orgs where security is a particular concern. Recommended by Salesforce in official documentation. Credentials are stored in three pieces (key, clientID, user) instead of a single token. Requires creating a custom Connected App per org. Disadvantages: more complex setup, and the username can be spoofed by supplying any user assigned to that Connected App. Assign only a single integration user to the Connected App to mitigate spoofing risk.

**Postcreation Sandbox Login** -- Use for automating sandbox cloning workflows. The Salesforce CLI can initiate a sandbox clone with `sfdx force:org:clone`, track progress with `sfdx force:org:status`, and automatically authenticate once the sandbox is ready. The authentication piggybacks on the production org's OAuth connection.

### Environment Strategy Guidelines

Always use the minimum number of orgs possible and keep connections simple. Resist the urge to add more orgs. Every additional environment increases the complexity of your delivery process and creates more opportunities for confusion, extra work, and delays.

This rule does not apply to scratch orgs since they are temporary and created from version control. It does apply to code repositories and long-running branches, which add overhead to keep in sync. Aim for one code repository, one main branch, and a small number of long-lived sandboxes.

The reasons for needing additional environments fall into five categories:

1. **Development** -- Prefer scratch orgs. Maintain developer sandboxes during the transition if scratch orgs are not yet feasible.
2. **Testing** -- "Shift left" by performing tests earlier in the development lifecycle. Static analysis and code reviews need no org. Apex unit tests run in development and test orgs. Automated acceptance tests can run in scratch orgs. UAT should be in fully integrated orgs with full sample data. Performance testing requires a partial or full sandbox and Salesforce authorization.
3. **Integration/Data Migration Testing** -- Use sandboxes integrated with test instances of external systems. Data import testing requires Partial Copy (5 GB) or Full Copy sandboxes because scratch orgs and Developer sandboxes are limited to 200 MB / 1 GB respectively.
4. **Training** -- Can sometimes share a UAT sandbox if training data can be clearly demarcated. Otherwise, maintain a dedicated training environment in your delivery pipeline so its metadata stays in sync automatically.
5. **Hotfixes** -- Handle critical production fixes through your normal delivery pipeline. If you practice continuous delivery, the master branch is always releasable. Use Feature Flags to safely deploy half-baked functionality. Never bypass your CI/CD pipeline in an emergency -- making untracked changes while panicked makes problems worse.

For hotfix debugging, consider maintaining a separate partial sandbox as a hotfix environment. Refresh it immediately after each production deployment so its data stays in sync. Connect it to a "hotfix" branch in your CI/CD system.

### Multiple Production Orgs

There are generally three reasons an organization has multiple production orgs: deliberate architectural decision, mergers/acquisitions, or different departments purchasing Salesforce independently.

When deciding whether to have multiple production orgs, study the "Enterprise Architecture: Single-org versus Multi-org Strategy" article on the Salesforce Developer Blog. Use the 2x2 matrix from *Enterprise Architecture as Strategy* (Ross, Weill, Robertson): high "Business Process Integration" implies shared data needs; high "Business Process Standardization" implies shared metadata needs. Only if both are low (the "Diversification" model) do truly independent orgs make sense.

When coordinating multiple production orgs:

- For shared data reporting, use Einstein Analytics or a third-party BI tool to aggregate across systems.
- For real-time data sharing, use Salesforce Connect with the Cross-Org Adapter.
- For shared customizations across orgs, use unlocked packages to build, version, and syndicate code and configuration. This is the strongest tool against "configuration drift."
- Second-generation managed packages are another option. They prevent the receiving org from modifying metadata (including Apex classes), which makes them less customizable but more consistent.

Never merge production orgs without engaging qualified consulting help and committing significant time and resources. Org merges are among the most complex Salesforce projects. Splitting orgs is simpler but still requires careful planning -- once split, reversing requires an org merge.

### Identifying and Mapping Existing Orgs

When joining a project and rationalizing existing environments:

- Log into each org and check Setup > Identity > Login History to determine if the org is actively used. No recent logins means it is a candidate for deletion.
- Check the "Environments" section in Setup of each production org to see all associated sandboxes, their types, and last refresh dates.
- Be suspicious of Developer Edition orgs used in enterprise development -- enterprise admins have no centralized control over them, and they may constitute a security concern.
- If the team uses a third-party release management tool (Gearset, Copado, etc.), check those systems for a list of authorized orgs and their connections.
- If using change sets, check Setup > Environments > Deploy > Deployment Settings to see org-to-org deployment relationships. Note that mapping these relationships requires logging in to each org individually.

### Creating Scratch Orgs

Always match the scratch org definition file to the characteristics of the target production org. The key properties are:

- **`edition`** -- Match to the target org (Developer, Enterprise, Group, Professional). Unlimited edition orgs are just Enterprise with higher limits.
- **`features`** -- Paid features not available in every org (e.g., `Communities`, `ServiceCloud`, `Chatbot`).
- **`settings`** -- Configuration that needs to be enabled (Metadata API settings). See the Metadata Coverage Report at `https://developer.salesforce.com/docs/metadata-coverage/` for what is available.

When migrating from org-based to source-based development, expect to spend time adjusting features and settings in the scratch org definition file until metadata deploys successfully. This debugging is comparable to resolving obscure deployment errors in traditional release management, but with the benefit that once resolved, the dependencies are explicit and need not be rediscovered.

**Org Shape** creates a template from a production org that mimics its features, limits, and settings. Use a reference to the shape ID in place of `edition` in the scratch org definition. Note that org shapes are not automatically updated when the production org's settings change -- you must recreate the shape.

**Scratch Org Snapshots** create templates from a configured scratch org. They can include managed packages and other metadata, which speeds up scratch org creation because large managed packages (e.g., Vlocity) can take up to 30 minutes to install. Be careful about what to include in a snapshot: only include underlying dependencies (managed packages, foundation metadata), not your team's active customizations. Push active metadata from version control to keep version control as the source of truth.

### Initializing Scratch Orgs

When a scratch org is first created, it has an edition, features, settings, and one admin user. To proceed with development, the typical setup steps are:

1. **Install dependent packages** -- Defined in `sfdx-project.json` using `packageAliases`. Salesforce does not auto-install dependencies during scratch org setup; use scripts or third-party tools (Texei plugin, Rootstock dependency utility) to automate this. Alternatively, use a scratch org snapshot with managed packages pre-installed.
2. **Push metadata** -- Use `sfdx force:source:push`. Push order matters because metadata depends on packages and data depends on user permissions.
3. **Create additional user accounts** -- Use `sfdx force:user:create` with `--setalias` for convenience. Use user definition files to define standard personas with specific profiles and permission sets. Refine these over time to allow developers and testers to view the org from end users' perspectives.
4. **Load sample or testing data** -- Use `sfdx force:data:tree` commands as the first approach. These handle hierarchical data (related Accounts, Contacts, Cases) in a single step without external IDs. Limited to 200 records per import. For larger volumes, use `sfdx force:data:bulk`. For individual record modifications, use `sfdx force:data:record`. Regularly update sample data alongside the code to capture the diversity of objects and data you work with.
5. **Run additional setup scripts** -- Use anonymous Apex (`sfdx force:apex:execute`) to populate configuration data or perform setup that cannot be handled by metadata. Use Selenium UI automation scripts for settings not yet supported by the Metadata API. Store all setup scripts in the code repository.

Use `.forceignore` files (similar syntax to `.gitignore`) to exclude metadata files or folders from being pushed/pulled to/from a scratch org.

### Developing on Scratch Orgs

Scratch orgs are always in a "known" state because they are created entirely from version control configuration. This eliminates the unpredictability of long-lived sandboxes that accumulate modifications from other users.

When deciding how frequently to recreate scratch orgs, consider:

- Whether you need different scratch org types for different applications (more types = more frequent recreation as you switch between work items)
- How long setup takes (push to automate fully; if setup exceeds 10 minutes, look for selective initialization mechanisms)
- Whether you use feature branches (each branch may need its own scratch org)

Scratch orgs can also serve as **review apps** -- self-service environments created dynamically from version control for testing, demos, or code review. Build a CI job that runs the same setup script developers use, then generates a one-click login URL with `sfdx force:org:open --urlonly`.

### Cloning and Refreshing Sandboxes

When creating, cloning, or refreshing sandboxes, understand that all three operations replace the sandbox with a new copy based on a source org. The only thing preserved during a refresh is the sandbox name and type. Refreshing is equivalent to destroying and recreating the sandbox.

Sandbox refresh intervals and data behavior:

| Sandbox Type | Data from Production | Refresh Interval | Storage |
|-------------|---------------------|-------------------|---------|
| Full Copy | Complete copy | Every 29 days | Same as production |
| Partial Copy | Subset via sandbox template | Every 5 days | 5 GB |
| Developer Pro | No data | Once per day | 1 GB |
| Developer | No data | Once per day | 200 MB |

When cloning a sandbox from another sandbox (not production), both must be the same type, and a complete copy of the source sandbox's data is made into the target.

After sandbox creation/refresh, Salesforce can trigger execution of an Apex class to perform setup actions. Common uses include deleting or obfuscating sensitive data, creating or updating users, and loading configuration data.

When planning org refreshes, rely on version control and continuous delivery to keep metadata in sync across environments. Use sandbox refreshes primarily for syncing data, not metadata. The old pattern of refreshing sandbox metadata from production implies a lack of control over the deployment process.

Your final UAT or staging environment should closely resemble production, especially regarding data volumes, because functionality that works with small data sets can fail under production volumes. Complex CPQ applications in particular can behave entirely differently depending on how configuration data is set up.

Always communicate sandbox refresh schedules clearly. Establish a shared calendar. Check Login History in Setup to identify active users before deleting or refreshing an org.

### Working with Lightning Dev Pro Sandboxes

Lightning Dev Pro sandboxes (LDPS) are designed to be created and destroyed using the Salesforce CLI or API, like scratch orgs, but with sandbox characteristics (1 GB storage, cloned from an existing org). They enable individual developers to have their own clean development environments that are up to date and free from accumulated cruft.

### The Salesforce Upgrade Cycle

Salesforce performs three major releases per year: Winter (autumn, timed with Dreamforce), Spring (early in the new year), and Summer (around June). Minor patches are released between major versions. The schedule is posted at `https://trust.salesforce.com`.

Each major release has a new API version (e.g., Winter '20 = API v47.0, Spring '20 = API v48.0). API versioning allows Salesforce to change functionality without breaking existing integrations and customizations.

To get early access to a release:

- **Prerelease Developer Edition orgs** -- Available 4-6 weeks before release. Sign up at `www.salesforce.com/form/signup/prerelease-<season><year>/`. These orgs provide earliest access but features may not make it to the final release. During this prerelease window, Salesforce runs "Apex Hammer" regression tests across all customer instances.
- **Preview sandbox instances** -- Available 2-3 weeks after prereleases. The instance number of your sandbox determines whether it is on a preview instance (e.g., CS21 = preview, CS22 = non-preview). The date you refresh your sandbox during the preview window determines which instance it lands on.
- **Preview scratch orgs** -- Add `"release": "Preview"` to your scratch org definition file to create a scratch org on the preview release, or `"release": "Previous"` if the Dev Hub has already been upgraded.

When deploying between environments on different releases, watch for new UserPermissions appearing in profiles downloaded from preview/prerelease instances. These will cause deployment errors to non-preview instances because the permissions do not exist there yet. Have an automated method to strip out problematic tags arising from different org versions.

### Behind-the-Scenes Architecture

Salesforce maintains its own data centers plus uses AWS and Google Cloud infrastructure. Within these data centers, over 100 "Pods" (instances like NA42, AP28) each represent a self-contained set of resources serving a group of customers.

Salesforce is a multitenant system: all customers share the same underlying database and application servers, with data segregated by Org ID. Each Pod has application servers and database server clusters, with backup instances for failover and cross-data-center replication.

All metadata including custom code is stored in the database. Apex code is stored in the database and compiled on demand. The database uses a generic structure: one table for standard object records, and a separate shared table for all custom fields and custom objects. Custom field records store a GUID, Org ID, object reference, and standard audit fields alongside the field values. This architecture explains the strict limits on custom fields per object (finite columns in the underlying table) and why Salesforce performs table joins for every query, leading to optimizations like skinny tables for large data volumes.

## Key Takeaways

- Always prefer scratch orgs for new development to enforce version control discipline and maintain a clean, reproducible development environment. Use sandboxes for integration testing, staging, training, and scenarios requiring production data.
- Never let developer sandboxes run indefinitely without refreshing -- they accumulate cruft and diverge from production. If using sandboxes for development, integrate changes frequently (daily, not monthly) to catch conflicts early.
- Always use the minimum number of long-lived environments needed. Every extra org adds maintenance burden and sync overhead. Scratch orgs are exempt from this rule because they are temporary.
- Always use OAuth 2.0 (not username/password) for API authentication, and always use a dedicated integration user for long-running integrations.
- Never store Auth URLs, client secrets, or credentials in your code repository. Use your CI system's secret management.
- Always match your scratch org definition file's edition, features, and settings to your target production org. Mismatches cause obscure deployment errors.
- Always automate scratch org setup (package installation, metadata push, user creation, data loading, setup scripts) to make creating new orgs fast and trivial.
- Never bypass your CI/CD pipeline for emergency fixes. Use Feature Flags to keep the master branch always releasable, and route hotfixes through the same pipeline.
- When managing multiple production orgs, use unlocked packages to syndicate shared customizations and combat configuration drift.
- Always plan and communicate environment changes (refreshes, deletions) well in advance. Check Login History to verify no one is actively using an org before destroying it.
