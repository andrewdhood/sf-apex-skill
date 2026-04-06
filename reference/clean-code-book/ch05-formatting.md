# Chapter 5: Formatting

> **Source:** Clean Code by Robert C. Martin (Prentice Hall, 2008)
> **When to read this:** Before writing or reviewing an Apex class or trigger — especially one that another developer will read cold.

## Key Principles

- **Formatting is about communication, not aesthetics.** Communication is the professional developer's first order of business.
- **Today's functionality changes, but today's readability sets precedents that outlive the code.** Your style and discipline survive even when the code does not.
- **Agree on a set of simple rules and apply them consistently.** On a team, the team's rules win over the individual's preferences.
- **Use an automated formatter whenever possible.** Don't rely on human willpower to apply formatting rules across a codebase.

## Detailed Notes

### The Purpose of Formatting

Code formatting is too important to ignore and too important to treat religiously. The functionality you write today may change next release, but the *readability* of the code will influence every change for as long as the code lives. Choose simple rules. Apply them consistently. Automate them if you can.

### Vertical Formatting

**File size.** Martin surveyed seven open-source Java projects and found that significant systems (FitNesse ~50,000 lines) can be built from files that are typically under 200 lines, with an upper limit of about 500. Small files are easier to understand than large ones. This is not a hard rule, but "short" should be the default.

**The newspaper metaphor.** A source file should read like a well-written newspaper article:
- The *name* is the headline. It should tell you whether you are in the right module.
- The *top* of the file gives high-level concepts — the synopsis.
- As you scroll down, *detail increases* until you reach the lowest-level functions at the bottom.
- A newspaper is made of many short articles; a good source file is made of many short, focused pieces.

**Vertical openness between concepts.** Blank lines separate groups of lines that represent complete thoughts. Separate the package declaration from imports; separate imports from class body; separate methods from each other. Each blank line is a visual cue that says "new thought starts here." Removing blank lines has an "obscuring effect" on readability — you can feel it if you unfocus your eyes.

**Vertical density.** Lines of code that are *tightly related* should appear vertically close. Comments jammed between closely-related instance variables break that association. Keep a class small enough that you can take it in "in one eye-full" without moving your head.

**Vertical distance.** Concepts that are closely related should live close together. Hunting up and down a file looking for where a function is defined wastes mental energy on *where* things are instead of *what* they do.

- **Variable declarations.** Declare variables as close to their usage as possible. Loop control variables should be declared in the loop. In a long-ish function, a variable might be declared at the top of a block just before a loop uses it.
- **Instance variables.** Declare all instance variables at the top of the class. Everyone should know where to go to find them. Don't hide them halfway down the listing.
- **Dependent functions.** If one function calls another, they should be vertically close, and the *caller should be above the callee* whenever possible. This gives the module a natural top-down flow: as you read downward, you descend into detail.
- **Conceptual affinity.** Functions that share a naming scheme or perform variations of the same task (e.g., `assertTrue(message, condition)` and `assertTrue(condition)`) want to be near each other even if they do not directly call each other.

**Vertical ordering.** Function call dependencies should point *downward*. A function that is called should be below the function that calls it. Most important concepts come first, with the least amount of polluting detail. Low-level details come last.

### Horizontal Formatting

**Line width.** Survey data says programmers overwhelmingly prefer short lines (40 percent of lines are under 60 characters in the surveyed projects). The old 80-character Hollerith limit is arbitrary; 100 or 120 is fine on modern monitors. Beyond 120 is just careless. Never scroll horizontally.

**Horizontal openness and density.** Use whitespace to associate strongly related things and disassociate weakly related things.
- Put spaces around assignment operators: `lineSize = line.length();` — the left and right sides are distinct and the space accentuates that.
- Do NOT put a space between a function name and its opening paren: `measureLine(line);` — the name and its arguments are one unit.
- Put spaces after commas in argument lists: `f(a, b, c)` — commas separate the arguments.
- Use whitespace to accentuate operator precedence: `return (-b + Math.sqrt(determinant)) / (2*a);` — no spaces around high-precedence `*`, spaces around low-precedence `+` and `/`. (Many auto-formatters will destroy this, sadly.)

**Horizontal alignment.** Martin used to line up variable names in declarations like old assembly code. He stopped. Lined-up columns emphasize the wrong things — your eye slides down the column of names and misses the types, or down the column of rvalues and misses the assignment operators. If you have a list long enough to *need* alignment, the real problem is that the list is too long and the class should be split up.

**Indentation.** A source file is a hierarchy — file, class, method, block, nested block. Indentation makes that hierarchy visible at a glance. Without indentation, code is essentially unreadable. Programmers scan the left margin to see scopes they can skip over.

- **Don't break indentation for short blocks.** Even for a one-line `if` body or a tiny class, expand and indent the scope instead of collapsing it to one line. Collapsed scopes hide structure.

**Dummy scopes.** If you have a `while` or `for` loop with an empty body, indent the semicolon on its own line and surround with braces so the reader cannot miss it.

### Team Rules

Every programmer has their own favorite formatting rules. When working on a team, the team rules win. A team should sit down, decide the rules in maybe ten minutes (brace placement, indent size, naming conventions), encode them in an IDE formatter, and then stick with them. A consistent style across the codebase is more valuable than any individual developer's preferences. The software should look like it was written by one group, not a bunch of bickering individuals.

## Salesforce/Apex Caveats

**⚠ Remember: Salesforce platform requirements ALWAYS override Clean Code principles when they conflict.**

### File size is not a free variable in Apex

Clean Code suggests files under 200 lines and a 500-line upper limit. Apex has realities that push files larger:

- **Trigger handlers often centralize logic.** A single `AccountTriggerHandler` that handles `before insert`, `after insert`, `before update`, `after update`, `before delete`, `after delete`, and `after undelete` can legitimately run 300+ lines even when cleanly factored. Don't break this into six micro-handlers just to hit a line count — the handler framework wants one class per SObject.
- **Batch classes have required `start`/`execute`/`finish` methods** that force a minimum file size and structure.
- **Test classes need `@IsTest` methods that set up entire record graphs.** A well-bulkified test class can easily run 500+ lines and that's fine — Salesforce mandates 75% coverage, so thorough tests beat short files.

**Rule of thumb:** Keep Apex classes focused on a single concern (the "Single Responsibility" principle) but let the line count be whatever it needs to be. An 800-line class that does one thing well is better than eight 100-line classes that require reading all eight to understand the flow.

### Vertical ordering has a hard platform constraint

Clean Code's "caller above callee, call dependencies point downward" rule is fine for Apex *within a class*. But across transactions, Apex has an **immutable execution order** dictated by the platform:

1. Validation rules
2. `before` trigger logic
3. DML (record saved, not committed)
4. `after` trigger logic
5. Assignment rules, auto-response, workflow
6. Process Builder / Flow
7. Escalation rules
8. Entitlement rules
9. Roll-up summaries
10. Commit to database
11. Post-commit: emails, async Apex (`@future`, Queueable), callouts from async context

**DML-must-happen-before-callouts** is a hard rule. If your method makes both DML and a callout, the DML has to come first and the callout has to come after — even if that means a method doesn't read top-to-bottom the way Clean Code wants. You may need to split the work across methods or even across transactions (using `Queueable` or `@future(callout=true)` to run the callout after the DML commits).

### Vertical density for SOQL and DML

SOQL queries should sit vertically close to the DML or loop that consumes them. Do NOT scatter a SOQL query at the top of a method and then reference its results 40 lines later — the reader has to hunt. Even better, wrap the query in a well-named helper method so the query lives next to its purpose.

```apex
// begin by pulling every Contact for the target accounts in one hit
Map<Id, List<Contact>> contactsByAccount = getContactsByAccount(accountIds);

// walk each account and reparent
for (Account a : accounts) {
    // ...
}
```

### Horizontal alignment and SOQL

Multi-line SOQL queries are the one place Apex developers do use horizontal alignment — lining up `FROM`, `WHERE`, `AND`, `ORDER BY` at the left margin for readability. This is fine. Clean Code's "don't align columns" rule is about *variable declarations*, not SQL.

```apex
List<Account> accts = [
    SELECT Id, Name, Industry,
           (SELECT Id, Email FROM Contacts WHERE Email != null)
    FROM Account
    WHERE Industry = 'Technology'
      AND BillingCountry = 'USA'
    ORDER BY Name ASC
    LIMIT 200
];
```

### Line width and Apex bulkification

Apex method signatures that handle bulk operations (`List<SObject>`, `Map<Id, List<Thing>>`, etc.) get long fast. 120 characters is a reasonable ceiling, but do NOT break a line in the middle of a generic type (`Map<Id,` newline `List<Thing>>`) — break at the commas between parameters instead.

### Team rules win — and the project's style guide is the team rule

This project's Apex style guide (comments with `// `, disembodied narrator tone, debug statements at key points, no single-line conditionals except ternary) is the team rule. Apply it consistently. A developer's personal preference for a different brace style or line width does not override the project style.

### System.debug at key points is a vertical-density cue

The Apex style guide asks for `System.debug()` statements at key points. Treat these as vertical-formatting signposts — place them where the logic hits a meaningful milestone, not every other line. They double as vertical cues that separate one logical phase from the next, similar to how Clean Code uses blank lines.

## Key Takeaways

1. **Formatting is communication.** It outlives the code. Invest in it.
2. **Newspaper layout for classes: headline (class name), synopsis (public methods, high-level flow), detail (private helpers) at the bottom.**
3. **Declare variables close to usage; declare instance variables at the top of the class; keep dependent functions vertically close with the caller above the callee.**
4. **File size for Apex is dictated by the platform, not by Clean Code's 200-line guideline.** Keep classes focused on a single concern; let the line count fall where it must.
5. **DML-before-callout is a platform rule that overrides "caller above callee" flow.** When they conflict, the platform wins.
6. **Use whitespace to signal relationships** (spaces around `=`, no space before `(` in calls, spaces after commas). Don't align columns of variable declarations.
7. **Never scroll horizontally.** 120 characters is the ceiling.
8. **The team's style guide is the team rule.** On this project, that means `// ` comments, disembodied narrator tone, no single-line conditionals (except ternary), and debug statements at key points.
9. **Use an automated formatter where Apex tooling supports it** (Prettier-apex, SFDX formatter) so consistency doesn't depend on human willpower.
