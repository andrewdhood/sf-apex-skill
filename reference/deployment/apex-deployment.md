# Apex Deployment & CI/CD

> **When to read this:** Before deploying Apex to any Salesforce org, setting up a CI/CD pipeline, or troubleshooting a failed deployment.

## Rules

- When deploying to production, always validate first with `sf project deploy validate` and then use quick deploy, because a validation-then-quick-deploy workflow minimizes the production deployment window.
- Always deploy dependent metadata before the code that references it (objects and fields before Apex classes, Apex classes before triggers that reference them) because Salesforce validates dependencies at deploy time and rejects broken references.
- Never deploy to production with `--test-level NoTestRun` because Salesforce requires tests for any deployment that includes Apex classes or triggers.
- When deploying to a sandbox for development or testing, use `--test-level NoTestRun` to skip tests and speed up iteration, because sandbox deployments don't require test coverage.
- Always run `RunLocalTests` (not `RunAllTestsInOrg`) for production deployments unless you have a specific reason, because `RunAllTestsInOrg` includes managed package tests that you don't control and that may fail for unrelated reasons.
- When using `RunSpecifiedTests`, always include every test class that covers the deployed Apex, because Salesforce calculates coverage only from the tests you specify -- if you miss a test class, your coverage may appear lower than it actually is.
- Never commit or deploy `.cls-meta.xml` files without their corresponding `.cls` source files (or vice versa) because Salesforce requires both files as a pair.
- When deleting metadata from a target org, always use `destructiveChangesPost.xml` (not `destructiveChangesPre.xml`) unless the deletion must happen before the deployment, because post-destructive changes are safer -- they only delete after the deployment succeeds.
- Always version-control your `package.xml` and destructive changes manifests because they are deployment artifacts that must be reproducible.
- When setting up CI/CD, always use a dedicated service user (not a personal admin account) for authentication because personal accounts change passwords, leave the org, and have permissions that are too broad.

## How It Works

### Source Format File Structure

Salesforce DX projects use "source format" -- a file structure optimized for version control where each metadata component is a separate file or directory.

```
my-project/
├── sfdx-project.json          # project config: source paths, API version, package info
├── config/
│   └── project-scratch-def.json  # scratch org definition
├── force-app/
│   └── main/
│       └── default/
│           ├── classes/
│           │   ├── AccountService.cls            # Apex source
│           │   └── AccountService.cls-meta.xml   # metadata (API version, status)
│           ├── triggers/
│           │   ├── AccountTrigger.trigger
│           │   └── AccountTrigger.trigger-meta.xml
│           ├── pages/
│           │   ├── PurchaseOrderPdf.page
│           │   └── PurchaseOrderPdf.page-meta.xml
│           ├── lwc/
│           │   └── purchaseOrderCard/
│           │       ├── purchaseOrderCard.html
│           │       ├── purchaseOrderCard.js
│           │       ├── purchaseOrderCard.js-meta.xml
│           │       └── purchaseOrderCard.css
│           ├── objects/
│           │   └── Purchase_Order__c/
│           │       ├── Purchase_Order__c.object-meta.xml
│           │       └── fields/
│           │           ├── Status__c.field-meta.xml
│           │           └── Total_Amount__c.field-meta.xml
│           ├── permissionsets/
│           │   └── PO_Admin.permissionset-meta.xml
│           └── layouts/
│               └── Purchase_Order__c-Purchase Order Layout.layout-meta.xml
└── manifest/
    └── package.xml               # optional: for targeted deployments
```

**Metadata file pairs:** Every Apex component has a source file and a `-meta.xml` companion:

| Component | Source file | Metadata file | Example meta content |
|-----------|------------|---------------|---------------------|
| Apex class | `MyClass.cls` | `MyClass.cls-meta.xml` | API version, status |
| Trigger | `MyTrigger.trigger` | `MyTrigger.trigger-meta.xml` | API version, status |
| VF page | `MyPage.page` | `MyPage.page-meta.xml` | API version, label, controller |
| VF component | `MyComp.component` | `MyComp.component-meta.xml` | API version |
| LWC | `myLwc/myLwc.js` | `myLwc/myLwc.js-meta.xml` | API version, targets, visibility |

**Example `.cls-meta.xml`:**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<ApexClass xmlns="http://soap.sforce.com/2006/04/metadata">
    <apiVersion>62.0</apiVersion>
    <status>Active</status>
</ApexClass>
```

**Example `.trigger-meta.xml`:**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<ApexTrigger xmlns="http://soap.sforce.com/2006/04/metadata">
    <apiVersion>62.0</apiVersion>
    <status>Active</status>
</ApexTrigger>
```

### sfdx-project.json Configuration

This is the project root config file. Every Salesforce DX project needs one.

```json
{
    "packageDirectories": [
        {
            "path": "force-app",
            "default": true,
            "package": "MyApp",
            "versionName": "ver 1.0",
            "versionNumber": "1.0.0.NEXT"
        }
    ],
    "name": "my-project",
    "namespace": "",
    "sfdcLoginUrl": "https://login.salesforce.com",
    "sourceApiVersion": "62.0"
}
```

| Field | Purpose |
|-------|---------|
| `packageDirectories` | Array of source paths. `force-app` is the default. Add multiple for modular projects. |
| `default` | Which directory gets new metadata by default. |
| `package` / `versionName` / `versionNumber` | Only needed for unlocked/managed packages. Omit for unpackaged metadata. |
| `namespace` | Empty string for most projects. Set only if developing a managed package. |
| `sourceApiVersion` | The default API version for new metadata files. Use the latest stable version. |

### SF CLI Deployment Commands

**Deploy source to an org:**

```bash
# deploy everything in force-app to the default org
sf project deploy start --source-dir force-app

# deploy to a specific org
sf project deploy start --source-dir force-app --target-org my-sandbox

# deploy specific metadata types
sf project deploy start --metadata ApexClass --target-org my-sandbox

# deploy specific files
sf project deploy start --source-dir force-app/main/default/classes/AccountService.cls --target-org my-sandbox

# deploy using a manifest (package.xml)
sf project deploy start --manifest manifest/package.xml --target-org my-sandbox

# deploy with tests
sf project deploy start --source-dir force-app --target-org production --test-level RunLocalTests

# deploy with specific tests
sf project deploy start --source-dir force-app --target-org production \
    --test-level RunSpecifiedTests \
    --tests AccountServiceTest \
    --tests PurchaseOrderControllerTest
```

**Retrieve source from an org:**

```bash
# retrieve everything defined in package.xml
sf project retrieve start --manifest manifest/package.xml --target-org my-sandbox

# retrieve specific metadata types
sf project retrieve start --metadata ApexClass --target-org my-sandbox

# retrieve specific components
sf project retrieve start --metadata "ApexClass:AccountService" --target-org my-sandbox

# retrieve from source tracking (scratch orgs and sandboxes with tracking enabled)
sf project retrieve start --target-org my-scratch-org
```

**Monitor deployment status:**

```bash
# check the status of a deployment
sf project deploy report --job-id 0Af...

# resume a deployment that timed out locally (the deployment continues server-side)
sf project deploy resume --job-id 0Af...

# cancel a running deployment
sf project deploy cancel --job-id 0Af...
```

### Test Coverage Requirements

**Production deployment rules:**
- 75% overall org-wide Apex code coverage is required.
- Every trigger must have at least 1% coverage (effectively: it must be covered by some test).
- Coverage is calculated from the tests that run during deployment. If you use `RunSpecifiedTests`, only those tests count toward coverage.
- Test methods, `@TestSetup` methods, and test classes do not count toward the code total.

**Test level flags:**

| Flag | Behavior | When to use |
|------|----------|-------------|
| `NoTestRun` | Skips all tests | Sandbox deployments without Apex changes |
| `RunSpecifiedTests` | Runs only named tests | When you know exactly which tests cover your code |
| `RunLocalTests` | Runs all non-managed-package tests | Standard production deployment (recommended default) |
| `RunAllTestsInOrg` | Runs every test including managed packages | Rarely needed; managed package tests can fail for unrelated reasons |

**Critical detail:** When deploying to production, if your deployment includes Apex classes or triggers, you must run tests. `NoTestRun` is rejected for production deployments that contain Apex.

### Validate-Then-Quick-Deploy Workflow

This is the recommended production deployment pattern. It separates validation (which runs tests and takes time) from the actual deployment (which is fast).

```bash
# step 1: validate the deployment (runs tests but doesn't commit changes)
sf project deploy validate \
    --source-dir force-app \
    --target-org production \
    --test-level RunLocalTests \
    --wait 60

# the output includes a job ID like: 0Af8Z00000LbR7zSAF
# note this ID — you need it for the quick deploy

# step 2: quick deploy using the validation job ID
# this is fast because tests already passed — it just commits the changes
sf project deploy quick \
    --job-id 0Af8Z00000LbR7zSAF \
    --target-org production
```

**Constraints on quick deploy:**
- The validation must have succeeded (all tests passed, 75%+ coverage).
- The validation expires after 10 days. After that you must re-validate.
- Any deployment to the same org between validation and quick deploy invalidates it.
- Quick deploy only works for production orgs, not sandboxes.
- `NoTestRun` validations cannot be quick-deployed.

### package.xml for Targeted Deployments

A `package.xml` manifest specifies exactly which components to deploy or retrieve.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Package xmlns="http://soap.sforce.com/2006/04/metadata">
    <version>62.0</version>

    <!-- Apex classes -->
    <types>
        <members>AccountService</members>
        <members>AccountServiceTest</members>
        <members>PurchaseOrderController</members>
        <name>ApexClass</name>
    </types>

    <!-- Apex triggers -->
    <types>
        <members>AccountTrigger</members>
        <name>ApexTrigger</name>
    </types>

    <!-- Visualforce pages -->
    <types>
        <members>PurchaseOrderPdf</members>
        <name>ApexPage</name>
    </types>

    <!-- Custom objects (deploys the object + all fields, etc.) -->
    <types>
        <members>Purchase_Order__c</members>
        <name>CustomObject</name>
    </types>

    <!-- Individual custom fields (when you don't want the whole object) -->
    <types>
        <members>Purchase_Order__c.Status__c</members>
        <members>Purchase_Order__c.Total_Amount__c</members>
        <name>CustomField</name>
    </types>

    <!-- Lightning Web Components -->
    <types>
        <members>purchaseOrderCard</members>
        <name>LightningComponentBundle</name>
    </types>

    <!-- Wildcard: retrieve ALL components of a type -->
    <types>
        <members>*</members>
        <name>ApexClass</name>
    </types>
</Package>
```

### Destructive Changes (Deleting Metadata)

To remove metadata from a target org, you deploy a destructive changes manifest.

**`destructiveChangesPost.xml`** (deletes after deployment succeeds -- the safe default):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Package xmlns="http://soap.sforce.com/2006/04/metadata">
    <types>
        <members>OldAccountHelper</members>
        <members>DeprecatedService</members>
        <name>ApexClass</name>
    </types>
    <types>
        <members>OldAccountTrigger</members>
        <name>ApexTrigger</name>
    </types>
</Package>
```

**`destructiveChangesPre.xml`** (deletes before deployment -- use when new code would conflict with the old):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Package xmlns="http://soap.sforce.com/2006/04/metadata">
    <types>
        <members>ConflictingClass</members>
        <name>ApexClass</name>
    </types>
</Package>
```

**Deploying destructive changes:**

```bash
# post-destructive: delete components after deploy succeeds
sf project deploy start \
    --source-dir force-app \
    --post-destructive-changes manifest/destructiveChangesPost.xml \
    --target-org my-sandbox

# pre-destructive: delete components before deploy starts
sf project deploy start \
    --source-dir force-app \
    --pre-destructive-changes manifest/destructiveChangesPre.xml \
    --target-org my-sandbox

# destructive-only (no new code to deploy): use an empty source dir or empty package.xml
# create an empty package.xml:
# <?xml version="1.0" encoding="UTF-8"?>
# <Package xmlns="http://soap.sforce.com/2006/04/metadata">
#     <version>62.0</version>
# </Package>

sf project deploy start \
    --manifest manifest/empty-package.xml \
    --post-destructive-changes manifest/destructiveChangesPost.xml \
    --target-org my-sandbox
```

**What you cannot delete via destructive changes:**
- Components referenced by active Lightning pages, flows, or processes
- Standard objects and standard fields
- Components in managed packages you don't own

### Change Sets

Change sets are the point-and-click deployment tool built into Salesforce Setup.

**When to use change sets:**
- Small, one-time deployments between connected sandbox and production
- Admin-led deployments by non-developers
- Environments without CLI access or version control

**Limitations of change sets:**
- Only work between orgs connected by a production-sandbox relationship
- Cannot be version-controlled or reproduced
- Cannot delete metadata (no destructive changes)
- Limited metadata type support (some types aren't available)
- No rollback capability -- you must manually undo changes
- Cannot be automated (no API for change sets)
- Deployment order is not guaranteed, causing dependency failures
- Upload + deploy is slow compared to CLI

**Bottom line:** Use change sets only when the CLI is not an option. For any repeatable deployment or team-based development, use `sf` CLI with version control.

### Deployment Dependencies

Salesforce validates all references at deploy time. If you deploy Apex that references a custom field that doesn't exist in the target org, the deployment fails.

**Deploy in this order:**
1. Custom objects and custom fields
2. Custom settings and custom metadata types
3. Permission sets and profiles (if they reference the above)
4. Apex classes (services, utilities, controllers)
5. Apex triggers (reference the classes above)
6. Visualforce pages and components (reference controllers)
7. Lightning Web Components (reference Apex controllers)
8. Flows and process builders (reference Apex invocable methods)
9. Layouts, record types, page layouts
10. Destructive changes (post-deploy)

**Tip:** When deploying everything at once via `--source-dir force-app`, Salesforce resolves most dependency ordering automatically. Dependency ordering matters most when deploying incrementally or via `package.xml`.

### Scratch Org Development Workflow

Scratch orgs are temporary, disposable Salesforce environments created from source. They last up to 30 days and are ideal for feature-branch development.

**Scratch org definition file (`config/project-scratch-def.json`):**

```json
{
    "orgName": "My Project Scratch Org",
    "edition": "Developer",
    "features": ["EnableSetPasswordInApi", "Communities"],
    "settings": {
        "lightningExperienceSettings": {
            "enableS1DesktopEnabled": true
        },
        "mobileSettings": {
            "enableS1EncryptedStoragePref2": false
        }
    }
}
```

**Workflow:**

```bash
# 1. create a scratch org from the definition file
sf org create scratch \
    --definition-file config/project-scratch-def.json \
    --alias my-feature \
    --duration-days 14 \
    --set-default

# 2. push your local source to the scratch org
sf project deploy start --target-org my-feature

# 3. assign permission sets
sf org assign permset --name PO_Admin --target-org my-feature

# 4. import sample data (if you have it)
sf data import tree --plan data/sample-data-plan.json --target-org my-feature

# 5. open the scratch org in a browser
sf org open --target-org my-feature

# 6. do your development in the scratch org or locally
# ... make changes ...

# 7. pull changes made in the scratch org back to local source
sf project retrieve start --target-org my-feature

# 8. commit to version control
git add -A && git commit -m "feat: add purchase order PDF generation"

# 9. delete the scratch org when done
sf org delete scratch --target-org my-feature --no-prompt
```

### Unlocked Packages vs. Unpackaged Metadata

| Aspect | Unpackaged Metadata | Unlocked Packages |
|--------|-------------------|-------------------|
| Deployment | Deploy source directly | Create a versioned package, then install |
| Tracking | Git only | Git + Salesforce package version history |
| Dependencies | Manual ordering | Declared in `sfdx-project.json` |
| Rollback | Manual | Install a previous package version |
| Multi-team | Coordinate deployments carefully | Teams deploy their own packages independently |
| Setup complexity | Low | Medium (requires Dev Hub, namespace planning) |
| Best for | Small teams, single-app orgs | Large teams, modular architectures, ISVs |

**When to use unlocked packages:** When you have multiple teams working on the same org and need clear boundaries between metadata groups, when you want versioned rollback capability, or when building internal apps that will be installed across multiple orgs.

**When to stick with unpackaged metadata:** Small teams, single-app projects, or when the overhead of package versioning isn't justified. This is the simpler path and works fine for most projects.

### CI/CD with GitHub Actions

A complete CI/CD pipeline for Salesforce has these stages:

1. **Validate on PR** -- Run static analysis and deploy-validate to a sandbox
2. **Deploy to QA** -- On merge to develop branch, deploy and run tests
3. **Deploy to Staging/UAT** -- On merge to release branch, deploy and run full tests
4. **Deploy to Production** -- On merge to main, validate then quick-deploy

**GitHub Actions workflow example:**

```yaml
# .github/workflows/sf-deploy.yml
name: Salesforce CI/CD

on:
    pull_request:
        branches: [main, develop]
        paths: ['force-app/**']
    push:
        branches: [main, develop]

jobs:
    # run on every PR to validate the deployment
    validate:
        if: github.event_name == 'pull_request'
        runs-on: ubuntu-latest
        steps:
            - name: Checkout source
              uses: actions/checkout@v4

            - name: Install Salesforce CLI
              run: npm install -g @salesforce/cli

            - name: Authenticate to sandbox
              run: |
                  echo "${{ secrets.SF_AUTH_URL_SANDBOX }}" > auth.txt
                  sf org login sfdx-url --sfdx-url-file auth.txt --alias ci-sandbox
                  rm auth.txt

            - name: Validate deployment (no commit)
              run: |
                  sf project deploy validate \
                      --source-dir force-app \
                      --target-org ci-sandbox \
                      --test-level RunLocalTests \
                      --wait 30

    # deploy to QA on merge to develop
    deploy-qa:
        if: github.event_name == 'push' && github.ref == 'refs/heads/develop'
        runs-on: ubuntu-latest
        steps:
            - name: Checkout source
              uses: actions/checkout@v4

            - name: Install Salesforce CLI
              run: npm install -g @salesforce/cli

            - name: Authenticate to QA sandbox
              run: |
                  echo "${{ secrets.SF_AUTH_URL_QA }}" > auth.txt
                  sf org login sfdx-url --sfdx-url-file auth.txt --alias qa-sandbox
                  rm auth.txt

            - name: Deploy to QA with tests
              run: |
                  sf project deploy start \
                      --source-dir force-app \
                      --target-org qa-sandbox \
                      --test-level RunLocalTests \
                      --wait 30

    # deploy to production on merge to main
    deploy-prod:
        if: github.event_name == 'push' && github.ref == 'refs/heads/main'
        runs-on: ubuntu-latest
        steps:
            - name: Checkout source
              uses: actions/checkout@v4

            - name: Install Salesforce CLI
              run: npm install -g @salesforce/cli

            - name: Authenticate to production
              run: |
                  echo "${{ secrets.SF_AUTH_URL_PROD }}" > auth.txt
                  sf org login sfdx-url --sfdx-url-file auth.txt --alias production
                  rm auth.txt

            - name: Validate deployment in production
              id: validate
              run: |
                  RESULT=$(sf project deploy validate \
                      --source-dir force-app \
                      --target-org production \
                      --test-level RunLocalTests \
                      --wait 60 \
                      --json)
                  JOB_ID=$(echo $RESULT | jq -r '.result.id')
                  echo "job_id=$JOB_ID" >> $GITHUB_OUTPUT

            - name: Quick deploy to production
              run: |
                  sf project deploy quick \
                      --job-id ${{ steps.validate.outputs.job_id }} \
                      --target-org production
```

**Authentication setup:** Store the SFDX auth URL as a GitHub secret. Generate it with:

```bash
# authenticate interactively first
sf org login web --alias my-org

# export the auth URL
sf org display --target-org my-org --verbose --json | jq -r '.result.sfdxAuthUrl'
# copy this value into your GitHub secrets as SF_AUTH_URL_PROD (or similar)
```

**Static analysis with PMD:**

```yaml
    # add as a step in the validate job
    - name: Run PMD static analysis
      run: |
          # install PMD
          wget https://github.com/pmd/pmd/releases/download/pmd_releases%2F7.0.0/pmd-dist-7.0.0-bin.zip
          unzip pmd-dist-7.0.0-bin.zip
          # run against Apex source
          ./pmd-bin-7.0.0/bin/pmd check \
              --dir force-app/main/default/classes \
              --rulesets category/apex/bestpractices.xml,category/apex/security.xml \
              --format text \
              --fail-on-violation
```

## Code Examples

### Minimal sfdx-project.json for a new project

```json
{
    "packageDirectories": [
        {
            "path": "force-app",
            "default": true
        }
    ],
    "name": "purchase-order-app",
    "namespace": "",
    "sfdcLoginUrl": "https://login.salesforce.com",
    "sourceApiVersion": "62.0"
}
```

### Full deployment script (bash)

```bash
#!/bin/bash
# deploy.sh: deploy force-app to a target org with tests
# usage: ./deploy.sh <org-alias> [test-level]

set -euo pipefail

ORG_ALIAS="${1:?Usage: ./deploy.sh <org-alias> [test-level]}"
TEST_LEVEL="${2:-RunLocalTests}"

echo "// deploying to ${ORG_ALIAS} with test level ${TEST_LEVEL}"

# validate first
echo "// step 1: validating deployment..."
VALIDATE_RESULT=$(sf project deploy validate \
    --source-dir force-app \
    --target-org "${ORG_ALIAS}" \
    --test-level "${TEST_LEVEL}" \
    --wait 60 \
    --json 2>&1)

JOB_ID=$(echo "${VALIDATE_RESULT}" | jq -r '.result.id // empty')

if [ -z "${JOB_ID}" ]; then
    echo "// validation failed"
    echo "${VALIDATE_RESULT}" | jq '.result.details.runTestResult.failures // .message'
    exit 1
fi

echo "// validation succeeded. job ID: ${JOB_ID}"

# quick deploy
echo "// step 2: quick deploying..."
sf project deploy quick \
    --job-id "${JOB_ID}" \
    --target-org "${ORG_ALIAS}" \
    --wait 10

echo "// deployment complete"
```

### package.xml for retrieving all Apex from an org

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Package xmlns="http://soap.sforce.com/2006/04/metadata">
    <version>62.0</version>
    <types>
        <members>*</members>
        <name>ApexClass</name>
    </types>
    <types>
        <members>*</members>
        <name>ApexTrigger</name>
    </types>
    <types>
        <members>*</members>
        <name>ApexPage</name>
    </types>
    <types>
        <members>*</members>
        <name>ApexComponent</name>
    </types>
</Package>
```

## Common Mistakes

1. **Deploying to production without running tests** -- Happens when using `--test-level NoTestRun` out of habit from sandbox work. Fix: Always use `RunLocalTests` or `RunSpecifiedTests` for production. The CLI will reject `NoTestRun` if your deployment contains Apex, but it's better to never form the habit.

2. **Deploying code before its dependent metadata** -- Happens when an Apex class references a custom field that hasn't been deployed yet. Fix: Deploy custom objects and fields first, then Apex. When using `--source-dir force-app`, Salesforce usually resolves this automatically, but `package.xml` deployments and incremental deploys require manual ordering.

3. **Using `RunAllTestsInOrg` for production deployments** -- Causes managed package tests to run, which can fail for reasons unrelated to your code, blocking your deployment. Fix: Use `RunLocalTests` which runs only tests you authored (excludes managed package tests).

4. **Forgetting that validation expires after 10 days** -- The team validates on Friday, plans to quick-deploy the following Monday, but something else deploys to production over the weekend, invalidating the job. Fix: Quick-deploy as soon as the validation succeeds. If the deployment window is scheduled, validate at the start of that window.

5. **Deploying a `.cls` file without its `.cls-meta.xml`** -- Happens when files are committed to git separately. Fix: Always commit and deploy both files as a pair. Use `sf project deploy start --source-dir force-app/main/default/classes/MyClass.cls` and the CLI will include the meta file automatically.

6. **Using change sets for repeatable deployments** -- Change sets can't be version-controlled, reproduced, or automated. Fix: Switch to `sf` CLI with source control. Reserve change sets for one-time admin changes where CLI access isn't available.

7. **Not using a destructive manifest when renaming/removing metadata** -- Deploying new code without deleting the old components leaves orphaned metadata in the target org. Fix: Include a `destructiveChangesPost.xml` that lists the old components to delete alongside your deployment of the new ones.

8. **Authenticating CI/CD pipelines with a personal admin account** -- When the person leaves or changes their password, all pipelines break. Fix: Create a dedicated integration user, authenticate with JWT flow (Connected App + certificate), and store the auth URL in GitHub secrets.

9. **Not testing in a sandbox that mirrors production** -- Code works in a developer sandbox but fails in production because of data volume, custom settings, permission differences, or org-specific configuration. Fix: Validate against a full or partial sandbox that is a recent refresh of production.

10. **Skipping static analysis in the CI pipeline** -- Code smells, security vulnerabilities, and anti-patterns ship to production unchecked. Fix: Add PMD (or a similar scanner) as a required check on pull requests. Use the `apex/bestpractices.xml` and `apex/security.xml` rulesets at minimum.

## See Also

- [Apex Fundamentals](../apex-fundamentals/apex-fundamentals.md)
- [Quick Actions & Invocable Methods](../quick-actions/)
- [Visualforce PDF Generation](../visualforce-pdf/)
- [Solution Architecture Patterns](../solution-architecture/)
