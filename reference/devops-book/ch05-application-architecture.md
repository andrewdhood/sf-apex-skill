# Chapter 5: Application Architecture

> **Source:** Mastering Salesforce DevOps by Andrew Davis (Apress, 2019)
> **When to read this:** When you need to structure Apex code for maintainability, implement enterprise design patterns (service/domain/selector layers), manage trigger architecture, or prepare a codebase for unlocked packaging.

## Key Concepts

- **Architecture is strategy, not tactics.** Martin Fowler defined architecture as "those aspects of a system that are difficult to change." On Salesforce, architectural decisions about how code is structured, layered, and packaged determine whether teams can scale or get buried in technical debt.
- **Loosely coupled architecture is the single biggest enabler of continuous delivery** -- bigger than test automation or deployment automation. High-performing teams with loosely coupled architectures actually *increase* deployments per developer as team size grows. Low-performing teams see the opposite.
- **Modular architecture on Salesforce** means constraining interdependencies so that portions of the codebase maintained by different teams are not tightly dependent on one another. The goal is unlocked packages that can be developed, tested, and deployed independently.

## Detailed Notes

### Understanding Dependencies

Every time a piece of Salesforce metadata references another piece of metadata, it establishes a dependency. The net result is a tangled web of dependencies -- "tight coupling" -- where different parts of the system have definite dependencies on other parts. The opposite is "loose coupling," in which multiple systems can interact with one another but don't depend on one another.

When building Apex, always think about the dependency graph. The Amazon vs. Google contrast is instructive: Amazon enforced strict separation between teams with communication only through published APIs, which produced AWS as a byproduct. Google maintains a single 2-billion-line repository with little isolation. Google achieves loose coupling through extraordinary tooling automation. Without that level of tooling, constrain interdependencies explicitly.

### Salesforce DX Projects and Modular Architecture

When structuring an SFDX project for modularity, understand these key points:

- **The `sfdx-project.json` file** defines package folders. Each package must be associated with a specific folder with no overlap between package folders.
- **Always keep one "default" package.** Any newly created metadata is downloaded and stored in this default package until it is moved elsewhere. A common setup: `force-app/main/default` for the default package, then additional folders like `force-app/sales` for other packages.
- **Package Directories** in `sfdx-project.json` allow you to specify dependencies on other packages in the same project or on packages defined elsewhere. They can also specify environment requirements by linking to a scratch org definition file.
- **Scratch org definition files** specify the characteristics of a scratch org (edition, features, settings) and are used in the package publishing process.

When managing code repositories for packages:

- Have at least two separate code repositories: one for org-level metadata and one for packaged metadata. You can then have separate repositories for each package or combine multiple packages into a single repository.
- Create a single parent folder to hold these multiple repositories on your local machine and a team/group/project in your version control system.
- Never strictly segregate the codebase between different teams -- this causes confusion and inefficiency. Salesforce metadata can interact in complex ways. Everyone on the development team should have access to all repositories so they can see what code and configuration has been defined, reuse it, and help refactor across package boundaries.
- Use CI systems with an Integration user's credentials, never personal credentials. Personal credentials create security risks and service interruptions if the user account is deactivated.

### Naming Conventions

Naming conventions are "poor man's packaging" but remain one of the few available methods to segment metadata in Salesforce:

- **Prefixes (fake namespaces):** Use prefixes like `HR_` to represent a department or group of capabilities. Salesforce packages use real namespaces like `SBOQ__` (separated by two underscores), but any team can create fake namespaces with a single underscore. As you move metadata into SFDX package folders, existing fake namespaces can serve as clues about which metadata belongs together.
- **Suffixes indicate function, not application ownership:** By convention, Apex test classes use the suffix `Test` (same name as the class they test, plus `Test`). Visualforce controllers use suffix `Controller`. Schedulable Apex uses suffix `Schedule`. And so forth.
- None of these conventions are enforced by the platform, but always adopt and adhere to a convention for your team.

### Object-Oriented Programming

When writing Apex, always leverage OOP principles because they are prerequisites for the more advanced techniques that follow:

- **Inheritance and interfaces** are requirements for Dependency Injection. Inheritance lets you write code based on the generic behavior of classes, even if you don't know which specific class will be used at runtime.
- Teams already making wise use of Apex's object-oriented architecture will find it easier to adopt dependency injection, packaging, and separation of concerns.

### Dependency Injection

Dependency Injection (DI) allows you to define dependencies at runtime, as opposed to when the code is compiled. This is critical for modular architecture because it breaks hard compile-time dependencies between classes that would otherwise prevent them from being separated into different packages.

**The progression from hard dependency to full DI:**

1. **Hard dependency (anti-pattern for packages):** `OrderingService` directly instantiates `FedExService`. The Apex compiler requires `FedExService` to exist when `OrderingService` is saved. This prevents them from being in separate packages deployed independently.

```apex
// anti-pattern: hard compile-time dependency
public class OrderingService {
    private FedExService shippingService = new FedExService();

    public void ship(Order order) {
        String trackingNumber = shippingService.generateTrackingNumber();
    }
}
```

2. **Inversion of Control:** Use an interface (`ShippingService`) and a strategy class (`ShippingStrategy`) to determine the implementation indirectly. `OrderingService` only sees the interface, not the implementation class.

```apex
// define an interface that multiple implementations can fulfill
public interface ShippingService {
    String generateTrackingNumber();
}

public class FedExImpl implements ShippingService {
    public String generateTrackingNumber() {
        return 'FEX-XXXX';
    }
}

public class DHLImpl implements ShippingService {
    public String generateTrackingNumber() {
        return 'DHL-XXXX';
    }
}

// strategy class determines which implementation to use
public class ShippingStrategy {
    public static ShippingService getShippingService(Order order) {
        if (order.ShippingCountry == 'United States') {
            return new FedExImpl();
        } else {
            return new DHLImpl();
        }
    }
}
```

3. **Full Dependency Injection with Custom Metadata:** Use `Type.forName()` and Custom Metadata to determine the class at runtime, eliminating all compile-time dependencies on the implementation classes.

```apex
// generic injector -- uses Type.forName() for dynamic instantiation
public class Injector {
    public static Object instantiate(String className) {
        Type t = Type.forName(className);
        return t.newInstance();
    }
}

// the implementation class name comes from a Custom Metadata record
Service_Implementation__mdt services = ServiceImplementation.load();
ShippingService shipping = (ShippingService)Injector.instantiate(services.shipping);
String trackingNumber = shipping.generateTrackingNumber();
```

**The Force-DI package** (by Andrew Fawcett, Douglas Ayers, and John Daniel) is a robust, open-source solution for DI on Salesforce. It defines a Custom Metadata type to store references to the metadata you want to call at runtime, and supports DI in Apex, Visualforce, Lightning, and Flows.

**When to use DI vs. events:** DI allows components from multiple packages to combine in a single transaction, so the entire transaction can be rolled back if there's a problem. Event-driven architecture (platform events) makes the publisher and subscriber use two different transactions, making them perhaps *too* loosely coupled for some scenarios.

### Event-Driven Architecture

Event-driven architecture is the ultimate loosely coupled architecture. Components communicate by passing events, not by being directly connected.

- **Pub-sub pattern:** One system publishes an event; other systems subscribe to categories of events they're interested in. Neither side breaks if the other is absent.
- **Lightning Events (2015):** Pass messages between Lightning Components in the same browser window. Extremely flexible for having many publishers and subscribers interacting without tightly coupled relationships.
- **Platform Events (2017):** Exchanged on the Salesforce platform itself (the Enterprise Messaging Platform / EMP). Provide a way to exchange messages between standard Salesforce components, custom components, and external systems.
- **Enterprise Service Buses (ESBs):** For integrating n systems, direct integrations require n * (n-1) connections. Point-to-point connectors require n * (n-1) / 2. An ESB requires only n connectors. The reduction in complexity is dramatic when connecting more than three systems.
- When using event-driven architecture for inter-package communication, remember that the publisher and subscriber of an event use two different transactions. Use DI instead when you need transactional integrity.

### Enterprise Design Patterns (Separation of Concerns)

These patterns originate from Andrew Fawcett's adaptation of Martin Fowler's *Patterns of Enterprise Application Architecture* into the Salesforce context. The book *Force.com Enterprise Architecture* and the Trailhead modules on Apex Enterprise Patterns are the definitive references. The FFLib Apex Commons library on GitHub provides prebuilt interfaces and classes to implement these patterns.

**The four layers of separation of concerns:**

| Layer | Responsibility | Salesforce Examples |
|---|---|---|
| **Presentation Layer** | User interface | Page Layouts, Lightning App Builder, LWC |
| **Business Logic Layer** | Calculations, business rules | Validation Rules, Processes, Workflow Rules, Apex services |
| **Database Layer** | Data model, stored data | Object model, standard record edit pages |
| **Data Access Layer** | Retrieve from database | Selectors, Reports, Dashboards |

**The classic anti-pattern to avoid:** Writing an Apex controller for Lightning or Visualforce that mixes UI management, calculations, data updates, and data access all in one class. This hides business logic inside what should be just a controller, prevents reuse if data is accessed through a different channel (like the API), and means data access and updates are defined only in that one controller even though other code may require similar methods.

#### Service Layer

The service layer centralizes business logic in a single place that can be accessed by many types of components: Visualforce controllers, Lightning controllers, API, Visual Flows, Batch Apex, Scheduled Apex, Queueable, Inbound Email Handlers, Apex REST/Web services, and Invocable Methods.

- **This is the first place to start** if you want to implement enterprise patterns. It generally does not require much refactoring -- it's mostly a matter of copying code into a central place and updating references to point to that service class.
- Each group of logic should be held in a single service class. For example, create a `PartnerCommissionService` that centralizes commission calculation logic. This service can then be used by any controller, the API, Visual Flows, etc. If you update the calculations, you only have to update a single place.

#### Unit of Work

The Unit of Work concept helps manage database transactions and bulkification when implementing a service layer.

- **Core idea:** Allow a group of records to be passed from one class to another, being modified and updated as needed, before finally being committed to the database in a single transaction.
- **FFLib implementation:** The FFLib module enables you to manage parent-child relationships, bulkify database operations, wrap multiple database updates in a single transaction to enable rollbacks.
- **Why it matters for packaging:** The Unit of Work object provides a common data type for passing records between classes from multiple packages. It provides an abstraction on top of DML and SOQL that allows multiple packages to collaborate on database transactions and queries.
- These techniques add overhead and complexity to your codebase, but reduce overhead and complexity as your project scales. Make a cost-benefit assessment for each project.

#### Domain Layer

The Domain Layer is an abstraction on top of DML that allows you to manage validation, defaulting, and trigger handling logic in a single place that can be used by any class involved in updating data.

- **Not just a trigger handler:** The Domain Layer provides a place to centralize business logic specific to a particular object, whether it's accessed through a trigger or through a service class.
- **FFLib naming convention:** Name Domain objects similarly to the native Salesforce objects they represent. For example, a Domain object called `Opportunities` wraps the native `Opportunity` object, giving you access to additional methods and properties (such as `.applyDiscount()`).
- **The Domain Layer provides a wrapper around DML** for both triggers and services. A trigger fires the Domain's `.triggerHandler()`, which dispatches to `.onBeforeUpdate()`, `.onBeforeInsert()`, `.onAfterUpdate()`, `.onAfterInsert()`, etc. A service class can call methods like `.applyDiscount()` on the same Domain object.

#### Selector Layer

The Selector Layer is a wrapper around SOQL that provides an object-oriented interface for building and sharing queries.

- **Problem it solves:** When query definitions are distributed across many classes, it becomes challenging to maintain them if fields change or new fields/filters are required. A Selector class lets you specify the base query in a single place.
- **Fluent syntax:** Selectors allow multiple methods to be chained together to modify the query. This is powerful when passing Selectors across classes from different packages -- the consuming class can be confident that required fields have been selected and row-level security has been enforced, and it can add requests for additional fields before execution.

```apex
// selector example with fluent method chaining
public List<OpportunityInfo> selectOpportunityInfo(Set<Id> idSet) {
    List<OpportunityInfo> opportunityInfos = new List<OpportunityInfo>();
    for (Opportunity opportunity : Database.query(
        newQueryFactory(false).
            selectField(Opportunity.Id).
            selectField(Opportunity.Amount).
            selectField(Opportunity.StageName).
            selectField('Account.Name').
            selectField('Account.AccountNumber').
            selectField('Account.Owner.Name').
            setCondition('id in :idSet').
            toSOQL()))
        opportunityInfos.add(new OpportunityInfo(opportunity));
    return opportunityInfos;
}
```

#### Factory Method Pattern

The Factory Method (or "Application Factory") pattern is another example of Inversion of Control:

- Instead of directly instantiating objects (e.g., `new Opportunity()`), create methods that return the object(s) you need.
- This enables mock interfaces for unit testing, polymorphic methods that determine behavior at runtime, and gathering repetitive boilerplate code in a single underlying method.

### Trigger Management

#### The One Trigger Rule

Always have exactly one trigger per object, and delegate all trigger logic to one or more handler classes. The reasons:

- **Execution order is unpredictable.** If you have more than one trigger on a single object, you cannot predetermine the order in which those triggers will fire. This leads to confusing scenarios where a trigger seems to behave correctly sometimes but not other times.
- **Redundant queries and loops.** It's common to need reference data in trigger logic. Having more than one trigger can lead to redundant queries and redundant iterations through records.
- Note: Managed packages may supply their own triggers, but this isn't normally a concern.

#### The Trigger Handler Pattern

Triggers have inherent limitations: you cannot define methods in them or use most object-oriented design patterns. Therefore:

- **Never put logic inside the trigger itself.** Instead, create a trigger handler class that holds all the logic.
- **Common trigger handler patterns** (like Mike Leach's "Comprehensive Trigger Template") provide the ability to:
  - Quickly disable triggers for certain users or at certain times (using Hierarchical Custom Settings) -- critical during data loads and integrations where trigger execution slows writes and may not be desirable.
  - Override trigger behavior by profile or by user.
  - Delegate some processing to `@Future` methods so they run asynchronously and don't count against governor limits.
- Trigger handler patterns are probably the earliest widespread use of software design patterns in the Salesforce community.

#### The Domain Pattern for Triggers

The Domain pattern (from enterprise design patterns) provides an object-oriented alternative to trigger handlers. The Domain object is associated with a specific object and ensures that logic is always associated with that object, whether it's accessed through a trigger or through a service class.

#### Modular Triggers with Dependency Injection

When trigger logic is distributed across multiple Apex classes (especially across multiple unlocked packages), DI becomes critical:

- **Use Custom Metadata to track handler classes.** Store the names of classes that should be executed in a trigger context and the order in which they should execute.
- **Multiple packages contribute Custom Metadata records** to register themselves to execute when a particular trigger runs. An admin on that org can modify the order of execution if desired.
- **When a trigger runs:** It first queries the Custom Metadata records (fast and "free" since they're stored in the platform cache), retrieves the list and order of handler classes, and then executes them in order.
- **This provides loose coupling** between the trigger and the handler classes, but still allows all execution to happen in a single transaction and to succeed or fail as a unit.
- John Daniel's `at4dx` project on GitHub demonstrates how to modularize a codebase into packages using this technique.

### Packaging Code

Packaging is the culmination of all the modular architecture techniques:

- **Packages define a clear organizational unit** for the codebase. They make code easier to understand, loosely coupled, easier to test, and easier to deploy consistently across orgs (including to more than one production org).
- **Metadata going into packages must be decoupled** from all other metadata except for declared package dependencies. This is the hardest part and is why the techniques above (DI, enterprise patterns, modular triggers) exist.
- **Packaging does not perfectly guarantee loose coupling.** You can still encounter unexpected behavior if package versions are out of sync between different orgs. But keeping track of package version numbers is infinitely easier than tracking subtle variations between unpackaged metadata spread across many orgs.
- **Specify dependencies in `sfdx-project.json`** using either an alias (e.g., `industry@0.1.0.12546` pointing to an ID, for packages not in the same project folder) or a package name + `versionNumber` (for packages on the same Dev Hub).

```json
"packageDirectories": [
  {
    "path": "force-app/healthcare",
    "package": "healthcare",
    "versionName": "ver 0.1",
    "versionNumber": "0.1.0.NEXT",
    "dependencies": [
      { "package": "industry@0.1.0.12546" },
      { "package": "schema", "versionNumber": "0.1.0.LATEST" }
    ]
  },
  {
    "path": "force-app/schema",
    "package": "schema",
    "versionName": "ver 0.1",
    "versionNumber": "0.1.0.NEXT",
    "default": true
  }
],
"packageAliases": {
  "industry@0.1.0.12546": "04t1T000000703YQAQ",
  "schema": "0Ho6F000000XZIpSAO"
}
```

- **Refactoring for packages is gradual.** Start with new applications that have limited dependencies on other metadata -- these are the easiest to begin with unlocked packages. In most cases, common metadata like Account fields are shared, so the entire development organization first needs to move together onto a common delivery pipeline to deploy metadata at the org level.
- **The "base-metadata" package pattern:** Create a base package that all other packages depend on (directly or indirectly). Changes to it should be made with care. Leaf packages (like "Account Planning" or "Customer Community") don't need to worry about causing side effects in other packages.

## Key Takeaways

1. **Loosely coupled architecture is the #1 enabler of DevOps performance** -- more impactful than test automation or deployment automation. Always design for loose coupling.

2. **Start with the Service Layer.** It's the simplest enterprise pattern to adopt and requires the least refactoring. Move business logic out of controllers and triggers into service classes. Everything else (Unit of Work, Domain, Selector) can be added incrementally.

3. **One trigger per object, always.** Delegate all logic to handler classes. Never put logic in the trigger body. Use Hierarchical Custom Settings or Custom Metadata to allow disabling triggers during data loads and integrations.

4. **Use Dependency Injection for cross-package communication.** Store class names in Custom Metadata and use `Type.forName()` + `.newInstance()` to instantiate at runtime. This eliminates compile-time dependencies that prevent code from being split into separate packages. Consider the Force-DI package for a robust implementation.

5. **Never mix concerns in a single class.** An Apex controller should not contain business logic, data access, and UI management. Separate into Service (business logic), Selector (data access), and Domain (object-level validation/defaulting) layers.

6. **Use naming conventions consistently.** Prefixes for organizational grouping (fake namespaces like `HR_`), suffixes for function (`Test`, `Controller`, `Schedule`). These become clues for refactoring into packages later.

7. **Choose DI over events when transactional integrity matters.** DI lets components from multiple packages combine in a single transaction with rollback capability. Platform events put publisher and subscriber in separate transactions.

8. **The FFLib Apex Commons library** provides prebuilt interfaces and classes for all the enterprise patterns. Learn it before implementing these patterns from scratch.

9. **Packaging is the goal, but it's a journey.** Start by setting up an org-level CI/CD pipeline using Salesforce DX file format. Then begin migrating metadata out of the org-level repository into package repositories. Refactor using the architectural techniques in this chapter as you go. This is the hardest part of adopting Salesforce DX, but also the most beneficial.

10. **Give everyone access to all code repositories.** Don't segregate the codebase between teams. Salesforce metadata can interact in complex ways, and everyone needs visibility to reuse code, interface with it successfully, and help refactor across package boundaries.
