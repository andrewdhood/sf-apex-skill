# Chapter 3: DevOps

> **Source:** Mastering Salesforce DevOps by Andrew Davis (Apress, 2019)
> **When to read this:** When you need a comprehensive understanding of DevOps principles, metrics, and change management strategies -- especially to justify or lead a DevOps adoption initiative within a Salesforce organization.

## Key Concepts

- **DevOps is a principle, not a destination** -- it is never "done." The core metric is lead time: how long it takes to deploy a single line of code to production.
- **The Three Ways of DevOps** (from Gene Kim): continuous delivery (left-to-right flow), continuous feedback (right-to-left feedback loops), and continuous improvement (improving the system of work itself).
- **Five key performance metrics**: Lead Time, Deployment Frequency, Change Fail Percentage, Mean Time to Restore, and System Uptime (Availability).
- **High-performing teams** deploy 46x more frequently, with 2,555x shorter lead times, 7x lower failure rates, and 2,604x faster recovery than low-performing teams.
- **Lean management principles** (just-in-time and stop-the-line) are foundational to DevOps -- break work into small batches, and halt the pipeline when defects are found.
- **Generative culture** (per Westrum's model) is performance-oriented, values collaboration over control, and is both a predictor and an outcome of DevOps adoption.
- **Blameless postmortems** are essential -- never cite "human error" as a root cause; instead investigate the circumstances that allowed the error to occur.
- **Theory of constraints** -- improvements to any part of the system other than the bottleneck will not yield significant overall benefit.
- **Kotter's 8 steps for leading change** provide a practical framework for driving DevOps adoption across an organization.
- **Diffusion of innovation** -- expect only 2.5% innovators and 13.5% early adopters initially; majority adoption follows a tipping point and requires crossing a "chasm."

## Detailed Notes

### What DevOps Is (and Is Not)

When explaining DevOps to stakeholders, emphasize that it is not about specific tools or a single technical implementation. DevOps is the practice of regarding IT processes as central to a business's ability to deliver value, and continually improving the process of delivering new functionality while ensuring quality and stability. It is a portmanteau of "Development" and "Operations," implying collaboration between these traditionally siloed teams.

When discussing DevOps in a Salesforce context, recognize that Salesforce itself takes the place of the "Operations" team in many ways -- Admins work as declarative developers building new capabilities, not just keeping the lights on. The Dev vs. Ops tension in Salesforce is between developers wanting to change things and admins wanting stability, but the roles are less clearly separated than in traditional IT.

### The Three Ways of DevOps

**1. Continuous Delivery (The First Way)**

When implementing the First Way, focus on these building blocks in order:
- **Version control**: The foundation. Use Git. All code and configuration must be tracked. This enables everything else because automated scripts can access code stored in version control.
- **Continuous integration**: Teams work on a common trunk, minimize branching, and run automated tests with every commit. The underlying principle is that no two developers should ever diverge significantly from one another. Early detection and fixes ensure quality at the earliest possible stage.
- **Continuous delivery**: All configuration needed to recreate your application is stored in and deployed from version control in an automated way. This is the main technical capability at the heart of DevOps.

When starting a DevOps journey, the first and most important step is to capture the state of all your systems in version control and perform all deployments using continuous delivery. These practices set the foundation for increasingly refined automation.

**2. Continuous Feedback (The Second Way)**

When implementing feedback loops, build fast automated test suites, monitor security and quality, and fail the pipeline as soon as a test fails. The code should always be in a deployable state. This ensures problems are detected and fixed before they reach production, or detected and resolved quickly if they do.

**3. Continuous Improvement (The Third Way)**

When investing in process improvement, recognize that entropy causes systems to break down over time. DevOps demands continual experimentation to improve the "system of work" -- the delivery pipeline itself -- and to prioritize this improvement above the work being delivered through it. This requires a culture of innovation, risk taking, and high trust.

### Lean Management Principles

When optimizing your development process, apply these two pillars from the Toyota Production System:

- **Just-in-time**: Minimize the time to deliver a requested feature by automating deployments and identifying bottlenecks that do not add value. Break work into small batches so each can be released as soon as it is completed. Deployment frequency is an excellent indicator of whether teams are following this approach.
- **Stop-the-line**: When a defect or abnormality is found, the pipeline stops and the top priority becomes identifying and uprooting the source of the problem. Build automated testing and validations that rerun on every change, making the system autonomic (it reacts to problems without conscious intervention).

When measuring waste in your process, track Lead Time -- the total time a customer waits for a feature to be delivered. Lead Time consists of three things: valuable work being done, non-value-added work being done (waste), and waiting for someone else. The difference between Lead Time and Process Time (the time if only valuable work were done) is a primary measure of waste. Reducing lead times allows for fast feedback, which enables innovation and success.

### Generative Culture and Blameless Postmortems

When building a DevOps culture, aim for Westrum's "generative" model: performance-oriented, focused on getting the job done rather than emphasizing control or process for its own sake. Control and rules still matter, but only in service of organizational effectiveness.

When conducting incident reviews, always use blameless postmortems. Never settle on "human error" as a root cause. Instead:
- Ask what circumstances allowed this mistake to unfold.
- Implement new protections and safety checks.
- Do not fire or discipline the person responsible -- invest energy in eliminating the risk of recurrence.

The AWS S3 outage of February 2017 is a model: a typo by an admin triggered a 4-hour outage affecting major Internet companies, but AWS's postmortem never cited "human error" as a factor, instead implementing new safety checks.

Adopting DevOps practices such as continuous delivery itself drives further cultural improvements by reducing barriers to innovation and collaboration. Culture and technical practices reinforce each other in a virtuous cycle.

### The Research: Software Delivery Performance

When justifying DevOps investment, use these statistics from the 2018 State of DevOps Report (Accelerate):

High Software Delivery Performance teams vs. low SDP teams:
- **46x** more frequent code deployments
- **2,555x** shorter lead time from commit to deploy
- **7x** lower change failure rate (1/7 as likely to fail)
- **2,604x** faster mean time to recover from downtime

High SDP teams spend **66% more time** on new constructive work and **38% less time** on rework and unplanned work. High-performing DevOps teams also have higher employee NPS than low-performing teams.

Companies like Amazon (23,000 deploys/day), Google (5,500/day), Netflix (500/day) deploy with high reliability, high customer responsiveness, and lead times measured in minutes. A typical enterprise deploys once every 9 months with lead times measured in months or quarters.

Software Delivery Performance also predicts corporate performance: high performers are twice as likely to exceed both commercial goals (profitability, market share) and noncommercial goals (efficiency, customer satisfaction, mission goals).

### DevOps as "Better Value, Faster, Safer, Happier"

When framing DevOps goals, use Jonathan Smart's summary:
- **Better**: Continuous improvement -- always striving to improve quality of work and products.
- **Value**: Agile development and design thinking to address end user needs. Software is not "done" until it is in users' hands.
- **Faster**: Continuous delivery to release more quickly. The longer between requesting and getting a solution, the less valuable it remains.
- **Safer**: Automated quality and security scanning integrated into the workflow. Automated testing gives confidence that each change is functional and safe.
- **Happier**: Continuously improving both the developer experience and the customer experience.

### Measuring Performance: The Five Key Metrics

When establishing DevOps metrics for your Salesforce team, track these five:

1. **Lead Time** (code committed to code deployed): Shorter lead time means faster feedback, faster innovation. Measure from code committed, not from feature requested (the latter has too much inherent variability).
2. **Deployment Frequency** (to production): Inversely related to batch size. Large batches delay value delivery, increase failure risk, and make diagnosis harder. Track this metric to measure progress toward smaller, less painful deployments.
3. **Change Fail Percentage** (production deployments that cause outage/rollback): High-performing teams achieve both high frequency and low failure rate. Use this to tune testing processes to catch failures before they occur.
4. **Mean Time to Restore** (from production failure): Teams that can quickly release features can also quickly release patches. Measure this to set a baseline and work toward faster incident resolution.
5. **System Uptime / Availability**: Added by the 2018 report. Aligns with SRE principles and sysadmin priorities. Consider Google's SRE concept of an "error budget" -- there is an acceptable level of downtime, and the goal is to balance reliability with innovation rather than maximizing uptime at the expense of all else.

When measuring these metrics on Salesforce, note that production orgs do not currently expose deployment history for querying. You will need to measure deployment frequency through your deployment tools. For uptime, `trust.salesforce.com` shows Salesforce platform uptime but not whether your custom services are working.

Surveys are a reasonable proxy for these metrics when automated measurement is not yet in place. Never use survey results to reward or punish -- only to track progress and enable self-improvement.

### Enhancing Performance: Five Factor Groups

When planning DevOps improvements, the State of DevOps Report identifies 32 factors (grouped into five categories) that drive software delivery performance:

1. **Continuous delivery** (version control, CI/CD, automated testing)
2. **Architecture** (modular, loosely coupled systems)
3. **Product and process** (agile, lean practices)
4. **Lean management and monitoring** (value stream optimization, observability)
5. **Cultural** (generative culture, collaboration, trust)

Focus on continuous delivery and architecture first for Salesforce teams, as these are the most directly actionable technical capabilities.

### Optimizing the Value Stream

When identifying where to improve your development process, use Value Stream Mapping. For each stage of your development and delivery process, gather:
- **Lead time**: Total time from a work item entering the stage until it leaves.
- **Process time**: Time that would be required if the activity were performed without interruption.
- **Percent complete and accurate**: Percentage of output from that stage that can be used as-is without rework.

The goal is to identify stages where work items spend time waiting or where quality issues cause rework. Begin by mapping the current state, then craft a future state value map that increases percent complete and accurate while reducing waste (lead time where no process is being performed).

### Theory of Constraints

When deciding where to invest improvement effort, apply the Theory of Constraints (from Goldratt's *The Goal*):

1. **Identify the constraint**: Look for buildup of "inventory" (work in progress). Where is the backlog largest? Where are things waiting longest? In software: how large is the dev backlog? How many items await code review? How much work is waiting to be tested or deployed?
2. **Exploit the constraint**: Maximize throughput of the bottleneck. Do not optimize other parts of the system first -- local optimization of non-bottleneck areas does not improve overall performance.
3. **Subordinate to the constraint**: Every other part of the system should be organized to support and feed the constraint. Ensure the constraint is never waiting on "raw materials" (specifications, requirements) and that its output is quickly moved to the next phase.
4. **Elevate the constraint**: If throughput is still insufficient after exploiting and subordinating, increase the capacity of the constraint itself (better tools, training, additional staff, restructuring).

This is a continuous cycle. Once you resolve one constraint, a new one emerges and the process repeats. Improvements to non-constraint areas are waste.

### Enabling Change: Diffusion of Innovation

When planning a DevOps rollout, account for the diffusion of innovation curve:
- **Innovators (2.5%)**: Will experiment with new tools and processes on their own. Find and empower these people.
- **Early adopters (13.5%)**: Bold, enthusiastic followers of innovators. They clear the trail and make the path easier.
- **Tipping point**: Once innovators + early adopters (16%) are on board, adoption can become self-sustaining through word-of-mouth and network effects.
- **The chasm** (Geoffrey Moore): There is a gap between early adopter enthusiasm and early majority pragmatism. Different messaging and enablement strategies are needed for each group.

For internal change enablement, drive adoption through intrinsic motivators -- people adopting because the process truly benefits them and their team -- rather than mandates. Start by attracting DevOps innovators within your company and enable them. Build from that initial group outward.

### Leading Change: Kotter's 8 Steps (Applied to DevOps)

When leading a DevOps transformation across a Salesforce organization, follow Kotter's framework:

1. **Create urgency**: 75% of management must buy in. Track metrics (deploy time, merge conflicts, regression tests, outage costs) and present them. Document the financial cost of ad hoc production changes and outages.
2. **Form a powerful coalition**: Find influential people across roles and regions who are committed to change. Survey to identify influencers (e.g., "which Appirian helped you most to improve your coding skills?"). Give them early access to training and demos.
3. **Create a vision for change**: Write a 1-2 sentence vision statement. Salesforce DX's tagline is a good model: "Build together and deliver continuously with a modern Salesforce developer experience." Define your values: speed? security? reliability? ease of debugging?
4. **Communicate the vision**: Repeat the message across many channels over a prolonged period. Use email, Chatter, all-hands calls, webinars, small group meetings. Embed concepts into daily work. Your actions speak louder than words -- model the practices yourself.
5. **Remove obstacles**: Listen to skeptics, but actively work to overcome structural and process barriers. Help skeptics understand that the benefits outweigh the learning curve. Every developer who has adopted version control and continuous delivery says they would never go back.
6. **Create short-term wins**: Start with one simple unlocked package that deploys across all orgs. Do not attempt to convert the entire org into a single massive package. Choose targets you can confidently achieve. Advertise and celebrate successes.
7. **Build on the change**: Do not declare victory too early. Continue setting targets for training and adoption. Encourage teams to track their own metrics and practice continuous improvement. Consider establishing a center of excellence.
8. **Anchor in corporate culture**: Embed these practices so deeply that they become self-sustaining and survive personnel turnover. Connect DevOps to the broader business context so that sales and business people also recognize its value.

## Key Takeaways

DevOps combines lean management principles with continuous delivery practices to maximize the flow of valuable software to end users while maintaining stability. The research conclusively shows that high-performing DevOps teams dramatically outperform their peers on both speed and reliability metrics, and that these technical practices predict positive business outcomes, organizational culture, and employee satisfaction. For Salesforce teams, the most important first step is getting all systems into version control and automating deployments; from there, apply the Theory of Constraints to identify and systematically eliminate bottlenecks. Driving organizational adoption requires deliberate change management -- start with innovators, build short-term wins, and create a clear, repeatable vision that can spread across the organization.
