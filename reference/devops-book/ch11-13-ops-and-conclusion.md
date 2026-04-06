# Chapters 11-13: Keeping the Lights On, Making It Better, and Conclusion

> **Source:** Mastering Salesforce DevOps by Andrew Davis (Apress, 2019)
> **When to read this:** When setting up operational processes for a Salesforce org, onboarding admins into a DevOps workflow, deciding what is safe to change directly in production, or establishing monitoring and governance practices.

## Key Concepts

1. **Salesforce does 80% of "Ops" for you** -- infrastructure, patching, scaling, and uptime are Salesforce's responsibility. Admin ops focus on user management, security, monitoring, and incremental improvements.
2. **DevOps is about cooperation, not tools.** The term originates from breaking down silos between Dev and Ops teams. On Salesforce, "Ops" looks different (admins instead of sysadmins), but the same silo problems apply.
3. **Use Permission Sets instead of Profiles** for managing permissions. Profiles are notorious for causing deployment errors because their metadata definition varies depending on what other metadata is present in the org.
4. **Lock people out of production** -- even Salesforce locked all their own sysadmins out of making direct changes to their production org (Org62). Start by restricting "Author Apex" and "Customize Application" permissions.
5. **Distinguish safe vs. unsafe production changes.** Most reports, dashboards, documents, email templates, list views, queues, and groups can safely be changed in production. Metadata that could break things or should exist in all environments must go through the delivery pipeline.
6. **Monitor what matters.** Set up Governor Limit Email Warnings, consider Salesforce Shield for event monitoring, use the Limits class in Apex for programmatic monitoring, and always monitor newly released features.
7. **Track issues and feature requests** separately from support cases. Support cases get immediate admin attention; issues and features go into the delivery pipeline for development, testing, and deployment.

## Detailed Notes

### Chapter 11: Keeping the Lights On

#### Salesforce Does the Hard Work for You

Salesforce handles the operations that would require entire teams on other platforms: database administration, security engineering, network analysis, load balancing, cache invalidation, and performance optimization. From the admin's point of view, Salesforce basically just works.

- Check `https://trust.salesforce.com` for outages or performance impairments (rare, short-lived, followed by root cause analysis)
- File support tickets for issues; pay for higher support tiers up to 24x7 Mission-Critical Applications (MCA) phone support

#### What Dev and Ops Cooperation Means on Salesforce

The DevOps concept comes from the 2009 Velocity conference talk "10 deploys a day: Dev and ops cooperation at Flickr." Key tensions:

- **Developers** want to innovate continuously; **operations** wants to keep everything running smoothly. Innovation implies risk; stability implies resistance to change.
- **Separation of duties** (a compliance requirement) has historically been used to prevent developers from releasing code into production. DevOps replaces manual separation with automated pipelines that provide the same controls with traceability.
- **Conway's Law**: Organizations produce systems that mirror their communication structures. If Dev and Ops teams do not communicate well, the overall system suffers.
- **Inverse Conway Maneuver**: Organize your teams (and their communication) in a way that is most conducive to your desired architecture.

On Salesforce, cooperation challenges include:
- Silos between developers, admins (app builders), business analysts, QA testers, and security specialists
- DevOps is not just "Dev vs. Ops" -- it is about cooperation across all teams participating in the final system
- Terms like TestOps, DevTestOps, and DevSecOps all address the same underlying issue: silos between teams increase inefficiency and risk

**Common failure modes:**

- Not providing developers access to a Salesforce DX Dev Hub (use the "Free Limited Access" production license)
- Long-running parallel development projects not developing in an integrated environment
- Developers having no access to test in a production-like environment
- Developers unable to submit Salesforce support tickets
- Infrequent release windows
- Developers neglecting to establish production logging
- Inefficient release paths or handoffs from development through to production
- Parallel consulting partner-led projects from different vendors introducing silos

When maintaining multiple production orgs, establish a Center of Excellence for knowledge sharing across teams.

#### Salesforce Admin Activities

##### User Management

- Use **single sign-on (SSO)** as the single source of truth for user access. Prevent password access to Salesforce except for admins when there is an SSO problem.
- **Two extremes to avoid**: too many system administrators in production (risk of untracked, conflicting changes) and not allowing developers any kind of access to production (they need access to debug logs, which requires "View All Data").
- The **Salesforce DX Dev Hub** is necessarily a production org. Developers need a user account on it. Use the **"Free Limited Access License"** which allows Dev Hub usage without the ability to view data or change metadata.
- Establish **integration users** for each integrated system (including your CI system). Ideally each integration has its own user for security and logging, but since integration user licenses cost as much as regular licenses, most companies combine multiple integrations under fewer users.

##### Security

Salesforce security is a layered, additive model:

- Every user is assigned a single **Profile** that establishes baseline security privileges
- **Permission Sets** add permissions on top of the Profile (permissions never subtract)
- Standard profiles: "Standard User," "System Administrator," "Integration User" -- can be cloned and customized
- Limit System Administrator privileges to a very small number of people

**Always use Permission Sets instead of Profiles for managing permissions** because:

- Profiles are the single biggest source of deployment pain. When you retrieve a Profile via the Metadata API, the definition varies depending on what other metadata is in the org. This makes CI/CD management extremely difficult.
- Permission Sets (since API version 40.0) are always retrieved and deployed consistently.
- Permission Sets can be used for everything except: page layout assignments, login hours, login IP ranges, session settings, password policies, delegated authentication, two-factor authentication with SSO, desktop client access, and organization-wide email From addresses.
- Minimize the number of Profiles. For example, if you need to lock call center users to a specific IP range, that justifies a "Call Center" profile. But do not create a separate profile for every category of user.

##### Managing Scheduled Jobs

- Scheduled jobs handle batch processing for activities too slow or computationally expensive to run on-the-fly
- Scheduled job definitions should be stored in version control and deployed through an automated process (not created manually in each environment)
- This becomes critical when promoting jobs across environments -- they can be promoted and tested gradually before deploying to production

#### Monitoring and Observability

**Built-in signals:**

- **Governor Limit Email Warnings**: Set up in Setup; sends email when Apex code uses more than 50% of governor limits. Combine with email filters to ignore certain types automatically.
- **Login and logout information**: Free in every org.
- **Salesforce Limits API**: Query organizational limits programmatically (`Limits` class in Apex for heap size, CPU time, etc.).
- **Developer Console**: Best tool for ad hoc page load time analysis and performance profiling, despite being somewhat buggy.

**Add-on tools:**

- **Salesforce Shield**: Platform encryption, field audit trail, and event monitoring. Event monitoring provides real-time access to performance, security, and usage data, which can be ingested into third-party tools like Splunk.
- **Proactive Monitoring**: Available to mission-critical support customers. Provides monitoring for page load times and transient errors.
- **ThousandEyes**: Enterprise-wide availability and page load time monitoring; details on which Salesforce data center is being used and network performance.
- **AppNeta**: Network traffic analysis and categorization with an eye toward prioritizing higher-value traffic.
- **Opsview**: Open-core system performance monitoring across on-premise, private, and public cloud. Uses the Salesforce REST API to monitor organizational limits.
- **Google Analytics**: For Salesforce Communities (Experience Cloud), remains one of the best tools for page views, click-through rates, and user behavior.
- Store performance metrics (heap size, CPU execution time) in a custom object to create a Salesforce-native application performance monitoring system with reports and dashboards.

**What to monitor:**

- Focus monitoring on newly released features -- release day is the **first day** of an application's life, not the last day of a project
- Monitor whether the application is working (error rates, page load times) and whether it is helping users (adoption, usage patterns)
- Remove or quiet monitoring once an application is proven stable. As the Site Reliability Engineering book states: monitoring can become fragile and complicated. Design it with simplicity in mind, and regularly remove data collection that is rarely exercised.

### Chapter 12: Making It Better

#### An Admin's Guide to Doing DevOps

DevOps tools matter far less than the culture shift involved. The key points for admins:

1. **Do not make changes directly in production.** Get access to a development environment (shared Dev sandbox, short-lived sandbox, or scratch org). Make changes there, capture them, and let the delivery pipeline promote them.
2. **Learn to see your configuration as XML in version control.** Even if you never write code, tracking your config changes alongside developer code gives you shared visibility, rollback capability, and confidence.
3. **Code literacy is valuable.** Code editors are just text editors. The gap between coders and non-coders is the gap between being literate and not literate in a particular language. Even basic Git and command-line experience unlocks new capabilities.
4. **Participate in the delivery pipeline.** You will face deployment errors and Git merge conflicts (especially ugly for Profiles -- another reason to use Permission Sets). But participating gives you shared responsibility, change tracking, and rollback capability.
5. **Help improve automated tests.** UI testing tools like Provar, Selenium, and Tosca are designed to be more admin-friendly than code-based tests. The logic in Flows and Processes is every bit as complex as code logic and deserves testing.
6. **Experiment in a development org, not production.** Every experiment in production puts your data, your users, and your reputation at risk.

#### Locking Everybody Out of Production

The story of Salesforce's own Org62 is instructive:

- Before 2013, Salesforce had hundreds of people with System Administrator privileges in their production org, all making legitimate customizations
- A small percentage of ad hoc changes caused unexpected issues -- fields removed from layouts, validation rules breaking integrations, etc.
- Each incident cost tens of thousands of dollars in developer debugging time, often outside work hours
- After years of debate, the IT team compiled cost statistics showing uncontrolled innovation was clearly more costly than restricting it
- Since roughly 2013, Salesforce has locked all sysadmins out of production. Every change goes through their DevOps workflow. Changes are tracked, propagated to every environment, and incidents are far less common.

**How to lock people out -- the recommended sequence:**

1. Clearly understand and articulate the business and financial justification
2. Check your motivation -- this is about increasing efficiency, not distrust
3. Establish a Delivery Pipeline using Integration User accounts for deployments
4. Ensure all developers and App Builder admins are comfortable making changes in scratch orgs or dev sandboxes and deploying through the pipeline
5. Identify which profiles and permission sets grant "Author Apex," "Customize Application," "Modify All Data," and "View All Data"
6. Consider separating sensitive permissions into a new permission set for fewer users
7. Identify a strictly limited number of trusted people to retain real sysadmin access
8. Communicate the change with executive backing, a timeline, and a clear explanation to affected users
9. Roll out in waves over several weeks to minimize business impact
10. Anticipate complaints, stand your ground, and reiterate the reasoning
11. Wait for the grumbling to die down. Short-term villain, long-term hero.
12. By the end, everyone except integration users and a few trusted sysadmins should be locked out

**Minimum restricted permissions:** Start by restricting just "Author Apex" and "Customize Application." Be careful with "Modify All Data" and "View All Data" -- review the full list of admin permissions.

**Automated enforcement:** Add a dynamic step to your build process that reviews all profiles and permission sets (other than the elevated-privilege ones) and fails the build if restricted permissions are detected. Setting permissions to `false` in the XML ensures they are removed even if someone adds them manually in production. Copado's Compliance Hub addresses this use case.

#### What's Safe to Change Directly in Production?

Two types of changes must go through the delivery pipeline:

1. Changes that could break things
2. Changes that should immediately be available in all development and testing environments

If neither applies, the change can safely be made directly in production.

**Metadata that can generally be changed safely in production:**

- Most Reports and Dashboards
- Most Documents
- Most Email Templates
- Most List Views
- Most Queues and Groups

**Exceptions:** Reports whose values are used in Visualforce pages, or any metadata that other metadata depends on.

**Data changes that are risky even in production:**

- Configuration data used as part of business logic
- Complex configuration data (Products in CPQ systems) that could break integrations or business logic

**Configuration data migration** through the delivery pipeline requires different tools from metadata. Commercial tools like Copado, AutoRABIT, and Gearset include hierarchical data migration as a native capability. Dedicated tools like Prodly Moover also specialize in this.

#### Tracking Issues and Feature Requests

- Support cases come from users and get immediate admin attention (password resets, how-to questions)
- Issues and feature requests are different: they imply changes that might affect org behavior, have side effects, and need to be tracked and tested
- Having a continuous delivery pipeline opens up the possibility of expediting low-risk, high-priority changes
- Tracking a request as an issue or feature should not mean indefinite lead time -- being able to release simple, low-risk changes quickly is a key benefit of an automated deployment pipeline
- Key DevOps success metrics to track and improve: **lead time, deployment frequency, change failure rate, recovery time, and uptime**

### Chapter 13: Conclusion

The DevOps movement is a gathering place for teams across all technologies to exchange ideas and improve tooling for managing complex configuration at scale. The Salesforce community is increasingly looking to DevOps to manage the sophistication that has outpaced traditional development lifecycle methods.

**Core message:** DevOps is fundamentally about building trust while delivering innovation. Innovation creates value; trust reduces risk. Doing both together, at high velocity and large scale, is the essence of DevOps.

The book emphasizes that every suggestion is optional and should be adapted to your team's needs. The goal is to make life easier and better for your coworkers and the people who depend on your work -- to spend less time on deployments and cleanup, and more time on creative, high-value work.

## Key Takeaways

Salesforce handles 80% of traditional operations work, so admin "ops" focuses on user management, security, monitoring, and continuous improvement rather than infrastructure. Always use Permission Sets instead of Profiles for managing permissions because Profiles are the single biggest source of deployment pain -- their metadata API representation varies depending on the org's contents, making CI/CD management extremely difficult. Lock everyone out of making direct production changes by restricting "Author Apex" and "Customize Application" permissions, following the model Salesforce itself adopted for their own Org62 -- the short-term pain of resistance is far outweighed by the long-term gains in stability and traceability. Distinguish between changes that are safe to make directly in production (reports, dashboards, list views, email templates) and changes that must go through the delivery pipeline (anything that could break things or should exist across all environments).
