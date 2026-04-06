# Chapter 7: The Delivery Pipeline

> **Source:** Mastering Salesforce DevOps by Andrew Davis (Apress, 2019), pp. 203-267
> **When to read this:** When setting up or refining version control, branching strategy, or CI/CD automation for a Salesforce project -- particularly when choosing between trunk-based development and feature branches, configuring CI pipelines, or managing org-level metadata across environments.

## Key Concepts

1. **The delivery pipeline** is the mechanism to move features and fixes safely from development to production. Its foundation is version control, its shape is determined by branching strategy, and its engine is CI/CD automation.

2. **Branching strategy balances freedom, control, and ease.** Freedom is the ability for developers to innovate (branches, forks, repos). Control is the ability to govern what gets deployed. Ease is how painlessly changes flow through the pipeline. These three are in tension -- more freedom means more merging overhead and less control.

3. **Trunk-based development is the recommended default.** Research from the 2017 State of DevOps Report shows that high-performing teams merge into trunk daily, keep branches alive less than a day, and maintain fewer than three active branches. Teams keeping branches alive longer than a day should treat it as a warning sign.

4. **CI/CD tools have three parts:** the CI engine (orchestrates jobs), configuration (defines what jobs run and when), and runners (isolated environments that execute jobs). Understanding this separation is key to choosing and configuring tools.

5. **Two distinct pipeline workflows exist for Salesforce:** package publishing (for unlocked/managed packages) and org-level management (for org-specific configuration metadata). They require different branching strategies and CI configurations.

---

## Detailed Notes

### Version Control Foundations

#### Git Tools
- **Command line (CLI):** The most powerful and universal interface. Every Git operation is available. Essential for scripting and CI/CD automation.
- **GUI clients:** Tools like SourceTree, GitKraken, and VS Code's built-in Git support provide visual representations. SourceTree recognizes slashes in branch names (e.g., `feature/`) and groups them like folders.
- **Git host web interface:** GitHub, GitLab, Bitbucket web UIs are useful for creating pull/merge requests, reviewing code, commenting on changes, and monitoring CI jobs. Also useful for soliciting contributions from less-technical team members who do not need to install Git locally.

#### Naming Conventions

**Commit messages** -- always include the ticket number from your tracking system to establish requirements traceability. Place the ticket number at the beginning of the message because CI systems may truncate long messages:

```
git commit -m "S-12345 I-67890 Added region to Account trigger"
```

Git hosts like GitHub and GitLab have integrated issue tracking. GitHub/GitLab can close issues automatically when commit messages contain syntax like `fixes #113`.

**Feature branch naming** -- use a consistent pattern that communicates purpose at a glance:

```
feature/[developer]-[ticketID]-[brief-description]
```

Example: `feature/sdeep-S-523567-Oppty-mgmt`

The components serve specific purposes:
- `feature/` prefix groups branches visually in GUI tools
- Developer name (e.g., `sdeep`) helps large teams identify who owns a branch
- Ticket ID (e.g., `S-523566`) ties the branch to work tracking and enables automation
- Brief description (e.g., `Oppty-mgmt`) tells humans what the work is about

**Squash commits** -- when merging feature branches, squash multiple small in-progress commits into a single meaningful commit. GitHub and GitLab both support squash-on-merge.

**Conventional Commits** -- a rigorous commit message convention used by Semantic Release. Each message begins with a type keyword:
- `fix` -- a bug fix (increments patch version)
- `feat` -- a new feature (increments minor version)
- `BREAKING CHANGE` -- not backward compatible (increments major version)
- Other accepted types: `chore`, `docs`, `style`, `refactor`, `test`

Format: `type(scope): description`

```
feat(lang): added Polish language
docs: correkt speling of CHANGELOG
BREAKING CHANGE: extendskey in config file is now used for extending other config files
```

Tools like Commitizen provide interactive prompts to enforce these conventions. Semantic Release builds on this to auto-increment version numbers based on commit messages.

### Preserving Git History When Converting to SFDX

When converting from Metadata API format to Salesforce DX format, follow these steps to preserve Git history:

1. **Delete original `src/` files and commit new SFDX files at the same time.** Git recognizes paired deletion + creation as a rename operation if file contents are relatively unchanged.
2. **Stage rename changes before making any other modifications.** If Git cannot correctly identify the rename, commit files in smaller batches by metadata type (e.g., stage deletion of `src/classes/` and creation of `force-app/main/default/classes/` together).
3. **"Decomposed" metadata (like custom objects) will not preserve history** because one large file is broken into many smaller files. But the original file history remains accessible even after deletion.
4. **When splitting into separate repositories**, clone or fork the existing repo rather than creating a new one. This preserves history in both the parent and child repositories. Then delete files from each repo that do not belong there.
5. **Before moving files to different folders**, first save and commit any pending changes to those files, then move them and commit the move separately.

### Branching Strategy

#### Trunk, Branches, and Forks

- **Trunk (mainline/master):** The single main branch. In Git, even the trunk is technically a branch. A short-lived branch lasts less than a day; a long-running branch lasts more than a day.
- **Branches:** Alternate versions of the codebase within the same repository. Useful for experimentation and isolation, but they must eventually be merged back, which carries risk proportional to how long they have been alive.
- **Forks:** Complete copies of a repository that maintain a connection to the original. Like a super-branch encompassing all branches. Two reasons to fork: (1) a team wants independent responsibility for their copy, or (2) allowing contributions from untrusted external developers. Forking should generally be reserved for open source contributions -- for internal Salesforce teams it creates enormous merge overhead.

#### Well-Known Branching Strategies

| Strategy | Best For | Author's Assessment |
|---|---|---|
| **Centralized / Trunk-based** | Most teams | Simplest, most efficient, proven workflow. Good place to start. |
| **GitHub Flow** | Open source, code reviews | Feature branches merged into trunk via PRs. Well suited for unknown contributors. |
| **Feature Branch Workflow** | Similar to GitHub Flow | Like GitHub Flow but includes rebasing commits onto master. |
| **GitLab Flow** | Versioned software | Generally more complex than needed for most Salesforce projects. |
| **Gitflow** | Complex release management | Sophisticated but tends to leave branches unmerged too long. Antithesis of CI. |
| **Forking Workflow** | Open source, multi-repo | Useful for open source but compounds risks of long-lived branches for internal teams. |

#### Research on Branching (2017 State of DevOps Report)

The DevOps Research and Assessment team found that high-performing teams:
- **Merge code into trunk on a daily basis**
- **Keep branches or forks alive less than a day**
- **Maintain fewer than three active branches**
- Work without code lock periods (which correlates with higher software delivery performance)

Key finding: high performers have integration times lasting hours; low performers have integration times lasting days. These differences are statistically significant.

**Guidance: If it takes more than a day to merge and integrate branches, re-examine your practices and architecture.**

#### Freedom, Control, and Ease

**Freedom** -- each degree of freedom (files/folders, commits, branches, forks, repos) adds a variation that must eventually be reconciled:
- Files and folders represent application structure
- Commits represent changes over time
- CI jobs manage stages of automation
- Tags mark specific commits (useful for triggering deployments)
- Branches represent alternate versions of code
- Merge requests provide formal approval for merging
- Forked repositories represent independent copies

**Control** -- version control is the non-negotiable foundation. Without it, you have no real control over environments. The fundamental principle: lock everybody out of changing metadata manually; use automated processes for deploying changes.

**Ease** -- three factors improve ease:
1. Automating builds and deployments for all configuration
2. Automating tests to identify bugs before production
3. Tuning branching approach to prevent unnecessary variations

**The KISS principle applies.** Keep files, folders, commits, and CI jobs as simple as possible. Since Salesforce only supports "the latest version" (no old versions to maintain), trunk-based development with a single branch is usually sufficient.

**Warning about merge requests:** While useful for open source, requiring them for internal developers often adds bureaucracy without value. In practice, most tech leads do not carefully review every MR, so it just delays merging. Real-time peer programming or informal code reviews generally yield higher performance.

### Branching for Package Publishing

For unlocked packages, a simple trunk-based strategy works well:
- Use a single branch (`master`) for package metadata
- Package versions correspond to commits on `master` (or tagged commits)
- A simple, linear commit history = a simple, linear progression of package versions
- Use a separate branch only for experiments you do not want to accidentally publish
- If you have multiple packages in one repo, it is generally simpler to split them into separate repositories to get independent security, automation, and build processes

**Evolving package publishing automation:**
1. Publish all packages on every commit to `master` (use CI job number as version)
2. Only publish if underlying metadata has changed
3. Publish only when a Git tag is added
4. Use Semantic Release to auto-increment versions based on commit message keywords (Bryan Leboff's `semantic-release-sfdx` plugin)

### Feature Branch Guidelines

Reasons to use feature branches:
1. Enables formal code reviews via merge/pull requests
2. Allows CI jobs to validate work-in-progress before merging to `master`
3. Allows in-progress features to be previewed in review apps

**Critical rules when using feature branches:**
- **Avoid long-running feature branches.** Merge at least daily if not within a few hours.
- **Delete feature branches after merging** to reduce clutter (only the name is deleted, not the history).
- **Before merging your branch into master, merge master into your feature branch first** (not the other way around). This forces the developer -- who knows their code best -- to resolve conflicts, rather than putting that burden on the MR reviewer.

### Branching for Org-Level Configuration

Managing org-level (non-package) metadata requires a different approach because:
1. Some configuration (integration endpoints, Named Credentials) varies between orgs
2. Orgs are populated with metadata at different stages of development
3. Some metadata (reports) does not need to be tracked

**Folder strategy for org differences:**
- One folder for metadata common to all orgs
- Separate folders for metadata unique to each org (NamedCredentials, RemoteSiteSettings, org-specific Custom Metadata)
- Deployments combine common + org-specific metadata into a temporary folder
- Configuration data can similarly be combined, with org-specific values taking precedence
- **Never store secrets (OAuth credentials) in the repo.** Use CI environment variables and inject them dynamically before deployment.

**Branch-per-environment strategy:**
- `master` branch = production metadata (also deploys to staging/UAT)
- `SIT` branch = system integration testing environment
- Feature branches for active development
- Use tags on `master` (e.g., `v1.0.3`) to trigger production deployments manually
- **Delete and recreate the SIT branch at the end of each sprint** to prevent synchronization drift

**The Appirio branching pattern (illustrated workflow):**

1. `master` represents production/UAT codebase
2. At sprint start, Tech Lead creates `SIT` branch from latest `master`
3. Developers create feature branches from latest `SIT`
4. Every commit to a feature branch triggers CI validation against SIT org
5. Developer merges SIT into their feature branch (to get latest changes), then creates MR from feature branch to SIT
6. Tech Lead reviews and approves MR; merge triggers deploy to SIT + validation against UAT
7. At sprint end, merge SIT into `master` and delete SIT branch
8. Commit to `master` triggers: (a) deploy to UAT, (b) validate against production, (c) deploy to production if a version tag is present

### Deploying Individual Features

When you need to expedite features independently rather than promoting entire environment branches:

**Best approach: Use unlocked packages.** Each feature becomes an independently publishable and installable unit.

**Alternative 1 -- Cherry picking:**
- Each commit is tied to one ticket (ticket number in commit message)
- Cherry pick specific commits to promote individual features
- **Challenges:** environments drift apart, easy to forget commits, features may have hidden dependencies on other commits, work stays in progress too long

**Alternative 2 -- Feature branches merged to each environment:**
- Feature branches are merged into each environment branch independently
- **Critical rule: Do NOT merge the destination branch into the feature branch** -- if SIT has 5 other features merged in and you merge SIT into your feature branch, you will carry all 5 features with you to subsequent environments
- **Challenges:** large number of long-running branches, every merge becomes a research project, complicated merging can silently exclude files from future merges

Both cherry picking and feature branching for granular deployment should be used sparingly -- primarily for urgent hotfixes. Packages are the long-term solution.

### Forking Workflow for Large Programs

Used when multiple large teams manage metadata for a shared production org:
- Each team has its own forked repository with their own sandboxes
- All forks merge into a shared staging repository, then into a single production repository
- Allows different security restrictions per team and simpler per-team repos
- **Major downside:** repositories inevitably drift apart. Expect to dedicate people full-time to upstream/downstream merges.

---

### CI/CD and Automation

#### What CI Actually Means

CI is not just "having Jenkins." Jez Humble's three-question test for true continuous integration:
1. Does your entire team merge their work into a common trunk **at least daily**?
2. Does every change to that codebase trigger a **comprehensive set of tests**?
3. If the build breaks, is it **always fixed in less than 10 minutes**?

If you cannot answer yes to all three, you are not actually practicing CI.

**A broken build is a blocking problem for the entire team.** Fix failing CI jobs immediately -- every passing hour makes it harder to identify the root cause. Once a single error breaks the build, all subsequent contributions also fail.

#### Automating the Delivery Process

**CI tool architecture:**
- **CI engine:** Orchestrates all automated jobs (e.g., Jenkins, GitLab CI, CircleCI)
- **Configuration:** Defines which jobs run, what triggers them, what processes execute, what notifications are sent
- **Runners:** Isolated environments where jobs actually execute. Typically Docker containers, sometimes VMs or dedicated servers

**Why runners are separate from the engine:**
1. Horizontal scaling -- long-running jobs do not slow the CI engine
2. Different hardware/OS requirements (e.g., Mac hardware for iOS builds)
3. Security isolation between teams

#### Pipeline Configurations

- Jobs are organized into **pipelines** -- groups of jobs triggered by events (commits, tags, schedules, API calls)
- Pipelines are divided into **stages** (or batches); jobs within a stage run in parallel, and a failed stage blocks subsequent stages
- Different branches can trigger different pipelines (e.g., `master` commits run deployment jobs; feature branch commits run validation jobs)
- **Multiproject pipelines** allow one pipeline to trigger pipelines in other projects (useful for rebuilding dependent packages)

#### CI Results in Merge Requests

CI can validate feature branches automatically and display pass/fail status alongside code changes in merge requests. This allows reviewers to see CI results before approving merges.

#### Choosing a CI Server

**Generic CI tools (Jenkins, GitLab CI, CircleCI, Bitbucket Pipelines) vs. Salesforce-specific tools (Copado, Gearset, AutoRABIT):**

- Salesforce-specific tools natively support deployment and testing but are often less flexible than generic tools
- Generic tools can automate any aspect of the development lifecycle but require more setup and scripting
- Salesforce DX increasingly enables generic CI tools to handle Salesforce workflows natively
- **Author's honest assessment:** Unless you have truly unique needs and a skilled developer tooling team, the total cost of buying a prebuilt Salesforce CI solution will generally be far lower than building everything yourself

**Three characteristics to look for in a CI tool:**
1. Team has autonomy to control CI configuration, job status, and logs
2. CI job configuration can be stored as a code file inside the codebase itself
3. Each CI job runs in an environment the team controls (typically a Docker container)

**Team autonomy matters.** Both the State of DevOps Report and the book Accelerate emphasize that teams having autonomy to choose and configure their own tools leads to better outcomes. Enforcing a single corporate CI server often becomes a bottleneck.

#### Why Use Docker Containers for CI Jobs

Docker provides:
- **Reproducibility:** Identical, clean environment every time a job runs -- zero chance of side effects from previous jobs
- **Simplicity:** No need to load CI servers with manually installed plugins or software
- **Speed:** CI systems cache Docker images; after first download, containers spin up in milliseconds
- **Flexibility:** Different jobs in the same pipeline can use entirely different Docker images (e.g., one job uses a Salesforce DX image, another uses a Ruby image)

**Sample Salesforce DX Dockerfile:**
```dockerfile
FROM node:8

# Installing Salesforce DX CLI
RUN yarn global add sfdx-cli
RUN sfdx --version

# SFDX environment
ENV SFDX_AUTOUPDATE_DISABLE true
ENV SFDX_USE_GENERIC_UNIX_KEYCHAIN true
ENV SFDX_DOMAIN_RETRY 300
```

#### Example: GitLab CI

GitLab CI configuration lives in a `.gitlab-ci.yml` file in the repo root. Dropping this file into the repo instantly activates CI -- no separate login or configuration needed.

**Sample `.gitlab-ci.yml` for package testing:**
```yaml
image: 'myCompany/salesforceDXimage:latest'

stages:
  - build
  - test

create_test_org:
  stage: build
  script:
    - sfdx force:org:create -a testOrg --setdefaultusername --wait 10
    - sfdx force:source:push
  only:
    - master

run_tests:
  stage: test
  image: ruby:latest
  script:
    - ruby -I test test/path/to/the_test.rb
  only:
    - master
```

**Sample `.gitlab-ci.yml` for org-level metadata management:**
```yaml
image: myCompany/salesforceDXimage:latest

stages:
  - deploy

deploy_to_SIT:
  stage: deploy
  script:
    - echo Hello World
  only:
    - /^SIT/

deploy_to_staging:
  stage: deploy
  script:
    - echo Hello World
  only:
    - master

deploy_to_production:
  stage: deploy
  script:
    - echo Hello World
  only:
    - tags
    - /^v[0-9.]+$/
  when: manual
```

The `only` property establishes distinct pipelines by branch or tag pattern. The `when: manual` property requires a human to click a button in the GitLab UI to trigger the production deployment.

#### User Permissions for CI Systems

- CI system permissions are often tied to repository permissions (e.g., CircleCI bases access on GitHub permissions)
- Most users should get "developer" access -- can trigger jobs and view logs but cannot see/change secrets
- Limit visibility of secret variables (Salesforce credentials, API tokens) to a few senior team members
- Monitor and audit who has repository and CI system access

#### Creating Integration Users for Deployments

**Never use a real person's credentials for CI deployments.** If that person leaves the company, jobs break and credentials may be compromised.

Instead:
- Create a dedicated integration user account in Salesforce
- Create a permission set with "Modify Metadata through Metadata API Functions" (more secure than "Modify All Data")
- Assign the permission set to the integration user
- Store the integration user's credentials as secrets in the CI system

#### Configuring CI/CD

**Main unit of CI configuration is a job.** Each job defines:
- What source code it works on (which repo, which branch)
- What triggers it (commit, tag, schedule, API call, manual)
- What actions ("build steps") it performs
- What pre-build and post-build actions run
- What notifications are sent

**Configuration as code** -- store CI configuration in the repository itself (e.g., `.gitlab-ci.yml`, `.circleci/config.yml`, `Jenkinsfile`). Benefits:
- Changes to CI config are versioned and tracked
- Team controls their own CI processes without going through another group
- Configuration is visible to the whole team
- Easily replicated to other projects

**Environment variables** serve two purposes in CI:
1. **Secrets:** Passwords, tokens, SSH keys, Salesforce credentials -- stored in the CI system's secret store, injected as env vars at runtime, masked in logs
2. **Flags:** Non-secret configuration like feature toggles or deployment targets that you want to change without modifying config files

**Local secrets:** Use a `.env` file (not checked into version control) for project-specific secrets needed locally. Load with `source .env` or tools like `dotenv`.

**Group-level configuration:** Store shared configuration (e.g., production org credentials) at the CI group/organization level rather than per-project. This provides reuse and an additional security layer since individual project team members cannot see group-level secrets.

### Example CI/CD Configuration

#### CI Jobs for Package Publishing

A typical package publishing pipeline triggered on every commit to `master`:

1. **Static code analysis** -- run PMD, SonarQube, or similar tools against the codebase in the CI runner. Export reports as artifacts. No Salesforce org needed.
2. **Unit test execution** -- create a scratch org, push source, run Apex tests. Use `--json` flag to get machine-readable output. Parse test results and coverage with `jq`. Fail the job on test failures or insufficient coverage. Consider persisting scratch org credentials as artifacts between jobs to avoid recreating the org.
3. **Package publishing** -- execute `sfdx force:package:version:create --wait 10`. Extract the subscriber package ID (04t...) from JSON output and pass it to the next job via artifacts.
4. **Package installation** -- authorize the target org using `auth:sfdxurl:store` or `auth:jwt:grant` with credentials from CI secrets. Install the package with `sfdx force:package:install --package 04t... --targetusername yourAlias --wait 10`.

**Start with a "walking skeleton"** -- create CI jobs for every step you intend to have, even if the initial scripts are just `echo Hello World`. Get the end-to-end pipeline working, then refine each job. Philosophy: "Make it work; make it right; make it fast."

**Incremental improvements:**
- Only publish packages when metadata has actually changed (use `git diff`)
- Use Semantic Release to auto-increment version numbers
- Cache scratch org credentials between jobs instead of recreating
- Add UI testing after Apex tests
- Add security scans and custom checks (e.g., all custom fields have descriptions)
- Use Rootstock's `rstk-sfdx-package-utils` to check installed package versions and avoid redundant installations

#### CI Jobs for Org-Level Management

Org-level CI builds on the branching strategy for org-level configuration:

**Three jobs per environment:**
1. `update_packages` -- check installed package versions, install/upgrade any that differ from what is specified in `sfdx-project.json`
2. `update_configuration` -- combine common metadata + org-specific metadata into a temporary deployment folder; deploy using Salesforce CLI. Use Git tags and diffs to deploy only changed metadata.
3. `acceptance_tests` -- run integration/acceptance tests against the org. These are more comprehensive than unit tests and exercise key user workflows.

**Linking package and org-level pipelines:**
- When a new package version is published, trigger a change in the org-level repository (update package version ID)
- GitLab's API can create commits, branches, and merge requests programmatically
- If your CI system lacks such an API, use a CI job that queries the Dev Hub for latest package versions and writes them back to the repo (requires adding a repo push token as a CI secret)

**YAML anchors** reduce repetition in `.gitlab-ci.yml` when multiple jobs share the same scripts but target different environments:

```yaml
.fake_job:
  <<: **&update_packages**
    stage: install
    script:
      - ./scripts/getInstalledPackages.sh $TARGET_ORG
      - ./scripts/updatePackages.sh $TARGET_ORG

update_packages_in_SIT:
  <<: ***update_packages**
  variables:
    - TARGET_ORG: SIT
  only:
    - /^SIT/

update_packages_in_staging:
  <<: ***update_packages**
  variables:
    - TARGET_ORG: staging
  only:
    - master
```

**Salesforce DX org authorization in CI:**
- Jobs must authorize with the Dev Hub and any target orgs
- Use Auth URL (`auth:sfdxurl:store`) or JWT flow (`auth:jwt:grant`) with tokens stored as CI secrets
- The `.sfdx` folder contains local auth credentials -- can be persisted as an artifact between jobs

---

## Key Takeaways

1. **Start with trunk-based development.** It is the simplest, most efficient, and research-backed strategy. Only add branching complexity (feature branches, environment branches, forks) when you have a concrete need that trunk-based development cannot address.

2. **Keep branches short-lived.** If a branch lives longer than a day, treat it as a warning sign. Delete environment branches (like SIT) and recreate them each sprint to prevent drift.

3. **Adopt unlocked packages to decouple feature deployment.** Cherry picking and feature branching for granular deployment are fragile workarounds. Packages let features be published and installed independently without Git acrobatics.

4. **Use version control as your single source of truth.** Lock everyone out of making ad hoc metadata changes in orgs. All changes flow through version control and automated deployment. Without this foundation, no amount of CI/CD tooling will provide real control.

5. **Store CI configuration as code in the repository.** This gives the team autonomy, makes changes visible and tracked, and allows easy replication across projects.

6. **Use Docker containers as CI runners.** They provide identical, clean environments every time, eliminate plugin/dependency management on CI servers, and allow different jobs to use completely different execution environments.

7. **Create dedicated integration users for CI deployments.** Never use a real person's Salesforce credentials in CI -- use a dedicated integration user with a purpose-built permission set ("Modify Metadata through Metadata API Functions").

8. **Build a walking skeleton first.** Create placeholder CI jobs for every step in your pipeline (static analysis, unit tests, package publishing, package installation) even if they initially just echo "Hello World." Get the end-to-end flow working, then refine.

9. **Treat broken builds as the top priority.** A failing CI pipeline blocks the entire team. Fix it within 10 minutes. Every passing hour makes root cause harder to identify.

10. **Separate org-specific metadata from common metadata.** Use folder structures to manage differences between orgs (NamedCredentials, RemoteSiteSettings, org-specific Custom Metadata). Combine them at deployment time, with org-specific values taking precedence. Never store secrets in the repo.

11. **Use tags to gate production deployments.** Auto-deploy to staging/UAT on every commit to `master`, but require a version tag (e.g., `v1.0.3`) plus manual approval to deploy to production. This provides both continuous delivery and a safety gate.

12. **Commit messages are a form of cross-team communication.** Always include ticket numbers (at the beginning, before truncation), use consistent formats, and consider Conventional Commits with Semantic Release for automated version management.
