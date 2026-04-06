# Chapter 10: Releasing to Users

> **Source:** Mastering Salesforce DevOps by Andrew Davis (Apress, 2019)
> **When to read this:** When you need to deploy features without immediately exposing them to users, implement feature flags, or design a release strategy that separates deployment from user-facing availability.

## Key Concepts

1. **Deploying and releasing are different.** Deploying moves code/config between environments. Releasing makes functionality available to users. You can deploy without releasing.
2. **"Dark deploying"** (decoupling deployments from releases) reduces stress, enables canary deployments, allows in-situ testing in production, and lets you deploy unfinished work safely.
3. **Permission Sets are the simplest release gate** -- deploy functionality but do not assign the Permission Set until the feature is ready for users.
4. **Layouts** can hide new functionality (Quick Actions, Related Lists, fields) from users until release.
5. **Dynamic Lightning Pages** use filter-based visibility rules to show or hide components based on record data or user attributes, providing a powerful UI-level release mechanism.
6. **Feature Flags** (Feature Toggles) enable or disable functionality at the business logic level using Custom Metadata, Custom Settings, or Custom Permissions.
7. **Branching by abstraction** introduces an abstraction layer between callers and a component being replaced, allowing gradual transition without branching in version control.

## Detailed Notes

### Why Separate Deployments from Releases?

Deploying is like buying and hiding Christmas presents; releasing is like giving them on Christmas morning. The key benefits:

1. **Reduces stress and risk** -- deployments can happen during business hours since users are not impacted. No weekend/evening deployment windows needed.
2. **Simplifies releasing** -- if releasing is just changing a flag or assigning a Permission Set, it can be done by an Admin at any time, coordinated with announcements and training.
3. **Enables in-situ testing** -- you can test features in production with real data before releasing to all users. No staging environment perfectly replicates production.
4. **Allows deploying unfinished work** -- this is a requirement for continuous deployment. Teams must be able to check incomplete features into trunk regularly without exposing them. Rather than long-lived feature branches, hide unfinished work behind flags.
5. **Enables canary deployments** -- release to a subset of users first (sometimes without their knowledge) to reduce the risk of a major change affecting everyone simultaneously.
6. **Makes releases reversible** -- if something goes wrong, disable the feature flag or remove the Permission Set instead of rolling back a deployment.

**When to just release by deploying (skip the separation):**

- Bug fixes -- once tested, deploy and release immediately
- Minor UI tweaks and new database fields
- Changes that would traditionally be made by admins directly in production (new report types, adding fields to layouts)
- When the risk of the change is low and no user coordination is needed

**Important:** Even Salesforce admins ("App Builders") should use the delivery pipeline rather than making changes directly in production. Direct production changes cause version control to drift out of sync, and those changes get overwritten on the next deployment.

### Permissions as a Release Mechanism

The simplest way to separate deployment from release: do not assign users the Permission Set that grants access to the new functionality.

- Every package should have at least one Permission Set associated with it
- When you install a package, the Permission Set is included but not automatically assigned to users -- you must assign it manually
- To release: assign the Permission Set to the appropriate users
- To do a canary deployment: assign the Permission Set to a subset of users first

**When the Permission Set is already assigned** (you are adding new capabilities to an existing package):

- If the new capabilities should never be available to some existing users, create a new Permission Set specifically for the new feature (e.g., "Chat User Permission Set" for Live Agent features added to a customer support package)
- Be careful about Permission Set proliferation -- complex permission landscapes create security and maintenance burdens. Only create new Permission Sets when there is a legitimate long-term use case.
- Avoid the temptation to skip the Permission Set and just add permissions manually in each environment. This reverts to a manual workflow that is untracked and error-prone.

### Layouts as a Release Mechanism

Salesforce page layouts control not just which fields are shown, but which Quick Actions, Related Lists, and embedded functionality are visible.

- A new Quick Action can be hidden by not adding it to the Layout assigned to users
- When ready to release, update the Layout to include the new functionality
- If rolling out capabilities for a specific team, create a Layout tailored to that team's needs

**Caveats:**

- Be extremely careful about creating layouts for short-term purposes. Everything created in Salesforce tends to persist far longer than intended.
- Every layout adds complexity and maintenance burden.
- Even the need for different page layout assignments is becoming smaller -- you can restrict access to fields by hiding them from certain users without separate layouts.

### Dynamic Lightning Pages

Dynamic Lightning Pages in the Lightning App Builder allow components to be shown or hidden based on filter criteria:

- Filters can reference record information: "show this component when the value is greater than 100"
- Filters can reference user attributes: "show this component when the User has the BetaUser__c flag enabled"
- This provides a powerful release mechanism at the UI level without needing separate layouts or Permission Sets
- Salesforce's roadmap includes adding the ability to show/hide specific fields and layout sections dynamically (not just components)

### Feature Flags (Feature Toggles)

Feature Flags are Boolean on/off settings checked at some point in your logic to enable or disable a feature. They have existed since the Unix universe began in 1970 in the form of "settings."

**Where to store Feature Flags:**

| Storage Mechanism | Use When | Available In |
|---|---|---|
| **Custom Metadata Types** | Feature should be deployable between orgs as metadata. General-purpose default. | Apex, Formulas, Validation Rules, Approval Processes, Flows, Process Builder |
| **Custom Settings (Hierarchical)** | Feature needs to vary by org, Profile, or individual user. Ideal for per-user feature testing. | Apex, Formulas, Validation Rules, Approval Processes |
| **Custom Permissions** | Feature is controlled purely at the permission level (enabled/disabled in a Formula or Apex). | Formulas, Validation Rules, Apex |

**Key properties of Custom Metadata for feature flags:**

- Values can be deployed between orgs just like other metadata -- developers define values and push them as part of a package
- Values can be overridden in a target org (unless marked as "Protected" and deployed as part of a managed package)
- A Custom Metadata record can be used to enable a feature but can be turned off by default, then selectively enabled in specific orgs to release
- Stored in Salesforce's Platform Cache, so accessed quickly without governor limit risk
- Available in Formula Fields, Validation Rules, Approval Process conditions, and elsewhere

**Key properties of Hierarchical Custom Settings for feature flags:**

- Dynamically change their value based on the environment, Profile, or user
- A setting can be turned off at the org level but enabled for all users with a particular Profile
- Can be overridden at the level of an individual user to allow that user to test a feature
- Ideal for canary deployments and gradual rollouts

**Feature Flags checked by code** (Apex, Flows, Invocable Methods, REST External Services) can use any conceivable mechanism to determine whether a feature is enabled, including calling external REST services to create cross-system feature flags (enabling capabilities in both Salesforce and SAP or Oracle simultaneously).

**Important:** Always remove Feature Flags once the feature is stable. Forgotten flags become useless if-statements and technical debt. "Good programmers write good code. Great programmers write no code. Zen programmers delete code."

### Branching by Abstraction

When transitioning from an old version of a component to a new version, branching by abstraction provides a way to do it gradually without long-lived feature branches or delayed deployments:

1. Create an abstraction layer that is called instead of the component being replaced
2. Initially, the abstraction layer simply passes all requests through to the old component (trivial implementation, safe to deploy)
3. As you build the new version, add decision criteria to the abstraction layer to delegate processing to either the old or new component
4. In Salesforce, this works in Apex code, Flows, and Processes

**Example in Apex -- a `ConfigService` class:**

```apex
// before: calling code queries configuration directly
Boolean enabled = [SELECT feature_enabled__c
    FROM Configuration_Object__c
    WHERE product__c = :product][0].feature_enabled__c;

// after: calling code uses an abstraction layer
public with sharing class CallingCode {
    public CallingCode() {
        String product = 'myProduct';
        Boolean enabled = ConfigService.isFeatureEnabled(product);
    }
}
```

The `ConfigService` can internally switch between querying a Custom Object (old way) and querying Custom Metadata (new way), allowing a gradual transition without impacting any callers.

Branching by abstraction should generally be temporary. Remove the abstraction layer once the transition is complete and fully tested -- unless the abstraction layer itself provides ongoing value (like `ConfigService` avoiding repetitive SOQL queries).

## Key Takeaways

Always think about deploying and releasing as separate activities -- even when they happen simultaneously, being aware of the distinction forces you to design reversible, low-risk changes. Permission Sets are the simplest and most Salesforce-native mechanism for gating releases: deploy the functionality, then assign the Permission Set when ready. Feature Flags using Custom Metadata Types or Hierarchical Custom Settings give you fine-grained control over business logic at the org, profile, or individual user level, and should always be cleaned up once the feature is stable. The ability to deploy without releasing is a prerequisite for continuous deployment, because it allows incomplete work to be merged to trunk regularly without impacting users. When choosing a release mechanism, match the technique to the need: Permission Sets for access control, Layouts for UI visibility, Dynamic Lightning Pages for component visibility, Feature Flags for business logic, and Branching by Abstraction for gradual component replacement.
