# Chapter 1: Introduction

> **Source:** Mastering Salesforce DevOps by Andrew Davis (Apress, 2019)
> **When to read this:** When you need to understand why DevOps matters for Salesforce teams and how Salesforce DX enables source-driven development for the first time.

## Key Concepts

- **Salesforce DX enables source-driven development** -- scratch orgs allow version control to be the source of truth for the first time, rather than always lagging behind the state of development orgs.
- **DevOps is the convergence of SaaS/PaaS and the DevOps movement** -- Salesforce provides the platform, and DevOps provides the processes to deploy on it effectively.
- **Most traditional DevOps tooling solves problems Salesforce doesn't have** -- infrastructure provisioning, database management, server monitoring are handled by Salesforce. The challenge is configuration consistency across orgs, not infrastructure management.
- **DevOps on Salesforce exists at a higher level of abstraction** -- application configuration rather than infrastructure provisioning. The same principles apply, but the tooling must be adapted.
- **The Three Ways of DevOps**: continuous delivery (left-to-right flow), continuous feedback (right-to-left feedback), and continuous improvement (improving the system of work itself).
- **Research-backed benefits**: High-performing DevOps teams deploy 46x more frequently, have 2,555x shorter lead times, 7x lower change failure rate, and 2,604x faster mean time to recover (per the State of DevOps Report / Accelerate).
- **Organizational culture matters**: Continuous delivery itself predicts positive organizational culture, high job satisfaction, and less burnout -- not just technical outcomes.
- **The book is structured in four parts**: Foundations, Salesforce Dev, Innovation Delivery (deployment/testing), and Salesforce Ops (administration).

## Detailed Notes

### Why Salesforce Was Slow to Adopt DevOps

When evaluating whether DevOps practices are worth adopting for a Salesforce team, understand that most "DevOps tools" on the market solve infrastructure problems that Salesforce already handles. Moving to Salesforce eliminates concerns about infrastructure, security patches, database management, and monitoring at the platform level. This is why adoption has been slow -- many DevOps practices simply weren't needed.

However, the problem that remains is ensuring database fields, configuration, and customizations are consistent across all Salesforce instances. When working across multiple orgs (dev, QA, staging, production), tracking and deploying changes reliably is the core challenge DevOps addresses for Salesforce.

### The Scratch Org Breakthrough

When choosing a development workflow, prefer scratch orgs over shared development orgs because scratch orgs are created entirely from version control. This means version control becomes the source of truth rather than always trailing behind the org. Before scratch orgs, vendors like Flosum and AutoRABIT offered synchronization tools, but the fundamental approach of org-as-source-of-truth was flawed.

### What DevOps Means in Practice

When explaining DevOps to Salesforce stakeholders, use these definitions:
- **Version control**: A mechanism to track and merge changes smoothly.
- **Continuous integration**: Teams work on a common master branch, minimize branching, and run automated tests after every change.
- **Continuous delivery**: All configuration needed to recreate your application is stored and deployed from version control in an automated way.
- **DevOps**: Both developers and operators/admins work together to optimize the flow of valuable work to end users, using the same mechanisms (version control) to both maintain and improve applications.

### The Book's Four Parts and When Each Applies

- **Part 2 (Salesforce Dev)**: When you need to maximize development quality and ease so that subsequent testing, deployment, and operations succeed. Develop with operation in mind.
- **Part 3 (Innovation Delivery)**: When you need to deploy, test, and release. Deploy in small batches to reduce risk, enable fast feedback, and avoid the QA squeeze that comes from delayed deployments.
- **Part 4 (Salesforce Ops)**: When managing production. In Salesforce, admins are the "operators" -- they build new capabilities declaratively and keep the lights on. Locking admins out of production is actually a requirement for DevOps goals, but it must be paired with a smooth process for committing and deploying declarative changes.

### The Admin Tension

When designing a DevOps process for a Salesforce org, always account for admins making changes directly in production. In Salesforce, the tension between dev and ops is inverted compared to most platforms: it is admins (not developers) who are more likely to make uncontrolled changes in production. For DevOps to work, there must be a smooth, admin-friendly process for small declarative changes to be made, committed, and deployed. Some organizations teach admins Git as their first skill.

## Key Takeaways

Salesforce DX finally makes source-driven development possible on the Salesforce platform through scratch orgs, which in turn opens the door to DevOps practices that have been proven to dramatically improve software delivery performance. The research (Accelerate, State of DevOps Report) shows that high-performing teams using these practices are not just faster but also more stable, with better employee satisfaction and less burnout. For Salesforce teams, the challenge is not infrastructure management but configuration consistency across orgs -- and any DevOps adoption must include a workable process for admins, not just developers.
