# Chapter 4: Comments

> **Source:** Clean Code by Robert C. Martin (Prentice Hall, 2008)
> **When to read this:** Before writing or reviewing comments in Apex, especially when deciding whether a comment is earning its keep or whether the code itself should be clearer.

## Key Principles

- **Comments are, at best, a necessary evil.** They compensate for the programmer's failure to express intent in code. Every time you write a comment, feel a small grimace of failure; every time you replace a comment with clearer code, feel a small win.
- **Comments lie.** Not always and not intentionally, but the older a comment is and the farther it drifts from the code it describes, the more likely it is to be wrong. Code moves; comments get left behind as orphaned blurbs of decreasing accuracy.
- **Truth lives in the code, not the comment.** Only the code can tell you what it actually does.
- **Comments do not make up for bad code.** If the code is a mess, do not paper over the mess with a comment — clean the mess.
- **Prefer expressing yourself in code over writing a comment.** Extract a well-named function or variable in place of a comment whenever you can.
- **The only truly good comment is the comment you found a way not to write.**

## Detailed Notes

### Explain Yourself in Code

Before writing a comment, ask whether the intent could be captured by a function name or a well-named boolean. Instead of:

```
// Check to see if the employee is eligible for full benefits
if ((employee.flags & HOURLY_FLAG) && (employee.age > 65))
```

Prefer:

```
if (employee.isEligibleForFullBenefits())
```

### Good Comments (the exceptions)

Martin lists a handful of comment categories that earn their keep:

1. **Legal Comments** — Copyright/authorship/license headers mandated by coding standards. Keep them short; reference an external license rather than pasting the whole thing.
2. **Informative Comments** — Basic information that cannot reasonably be moved into a name. E.g., explaining what format a regex matches: `// format matched kk:mm:ss EEE, MMM dd, yyyy`. Still prefer renaming or extracting a helper when possible.
3. **Explanation of Intent** — Documenting *why* you made a non-obvious decision. E.g., `// we are greater because we are the right type.` Tells the reader what the programmer was trying to accomplish.
4. **Clarification** — Translating an obscure argument or return value (from a standard library or code you cannot change) into something readable. Risky, because clarifying comments can themselves be wrong — verify them carefully.
5. **Warning of Consequences** — Warning other programmers about nonobvious pitfalls. E.g., `// SimpleDateFormat is not thread safe, so we need to create each instance independently.`
6. **TODO Comments** — Reminders of jobs that should be done but cannot be done now. Use sparingly and scan through regularly to clear out the ones you can. TODOs are never an excuse to leave bad code in the system.
7. **Amplification** — Amplifying the importance of something that might otherwise look inconsequential. E.g., `// the trim is real important. It removes the starting spaces that could cause the item to be recognized as another list.`
8. **Javadocs in Public APIs** — Well-written API docs are genuinely helpful. BUT the same honesty rules apply: a bad javadoc is still a bad comment.

### Bad Comments (the overwhelming majority)

1. **Mumbling** — Plopping a comment in because "the process requires it." If the comment is not clear enough that a reader can understand it without looking at other modules, it is worthless.
2. **Redundant Comments** — Comments that take longer to read than the code itself and add no new information (e.g., a header comment that just restates the method signature).
3. **Misleading Comments** — Comments that are not quite accurate. Even a slight imprecision can lead another programmer down a debugging rathole.
4. **Mandated Comments** — Rules like "every function must have a javadoc" or "every variable must have a comment" produce clutter, lies, and confusion. It is silly to require this.
5. **Journal Comments** — Change logs at the top of the file. Source control does this job. Delete these on sight.
6. **Noise Comments** — Comments that restate the obvious (`/** Default constructor. */`, `/** The day of the month. */`). The eye learns to skip them. Eventually they go stale and lie.
7. **Scary Noise** — Javadoc-style noise, especially cut-and-paste errors (two fields both documented as "The version."). If the author did not care, the reader cannot benefit.
8. **Don't Use a Comment When You Can Use a Function or a Variable** — Refactor the code so the comment is unnecessary.
9. **Position Markers** — Banners like `// Actions //////////////////`. Use very sparingly; they quickly become background noise.
10. **Closing Brace Comments** — `} // end while`. If you need these, your function is too long. Shorten the function instead.
11. **Attributions and Bylines** — `/* Added by Rick */`. Source control remembers. These go stale and pollute.
12. **Commented-Out Code** — An abomination. Delete it. Source control remembers. Commented-out code piles up because nobody dares delete it "in case it was important."
13. **HTML Comments** — Using HTML markup in source comments is an abomination. It makes the comment hard to read in the editor. If a tool extracts comments for the web, that is the tool's job.
14. **Nonlocal Information** — A comment that describes code far away from where it lives. It will go out of sync the moment the distant code changes.
15. **Too Much Information** — Historical discussions, RFC excerpts, etc. Keep comments local and relevant.
16. **Inobvious Connection** — If the comment's relationship to the code is not obvious, the reader has to decode both. The comment should clarify, not add its own puzzle.
17. **Function Headers** — A short, well-named function does not need a comment header.
18. **Javadocs in Nonpublic Code** — Formal javadoc is anathema for internal APIs; it is cruft that distracts from the code.

### The Prime Generator Example

Martin walks through refactoring a cluttered `GeneratePrimes.java` into `PrimeGenerator.java`. The original has comments scattered throughout explaining each loop. The refactor replaces them with well-named small functions (`uncrossIntegersUpTo`, `crossOutMultiples`, `putUncrossedIntegersIntoResult`, `notCrossed`, `numberOfUncrossedIntegers`). Only two comments survive: a class-level summary and one explaining the square-root-as-iteration-limit trick — the one comment that the names could not capture.

## Salesforce/Apex Caveats

**⚠ Remember: Salesforce platform requirements ALWAYS override Clean Code principles when they conflict.**

### This project's comment style explicitly overrides several Clean Code rules

The developer's Apex style guide mandates:
- `// ` with **one space** after the slashes (format preference, not philosophical conflict)
- **Disembodied narrator tone** in comments (`// begin sorting through accounts`, `// remember that we put xyz here for this reason`) — conversational storytelling through the code
- **Every method gets param/output comments** — this is a hard "mandated comments" rule, which Clean Code explicitly calls out as "just plain silly"
- **Debug statements at key points** — adds narrative comments in spirit, even when the code itself could be "clean"

**How to reconcile:** Think of Apex comments on this project as a *narrated audio tour* of the code, not as a desperate attempt to explain unclear logic. The developer is stronger on Apex than on JS, and reads Apex linearly — so the narrator tone is a deliberate teaching device, not a crutch for bad code. Clean Code's warning still applies inside method bodies: if an inline comment exists only because the code is unclear, the code is the thing to fix, not the comment.

### Method headers are mandatory on this project

Clean Code says "short functions don't need much description." The Apex style guide says every method gets param/output comments. **The style guide wins.** Write the headers. Keep them accurate. When refactoring, update the headers every time — a stale header is worse than no header.

Acceptable pattern for an Apex method header:

```apex
// handleAccountMerge — consolidates duplicate Accounts and moves their Contacts
// params:  List<Account> duplicates — the accounts to merge into master
//          Account master — the surviving Account record
// returns: Integer — count of Contacts re-parented to master
// note:    runs in without sharing so system automation can re-parent
//          Contacts owned by users the caller cannot see
public static Integer handleAccountMerge(List<Account> duplicates, Account master) {
    // begin by pulling every Contact attached to the duplicates in one SOQL hit
    ...
}
```

### Warning-of-consequences comments are critical in Apex

Apex has nonobvious platform gotchas that future maintainers *will* trip on without a warning comment. Examples where a warning comment is mandatory:

- `// DML must happen before this callout — the platform throws CalloutException otherwise`
- `// do NOT put SOQL inside this loop — governor limit is 100 per transaction`
- `// this runs without sharing so the guest user can write; object-level CRUD is still enforced`
- `// @future because the triggering context has uncommitted work (mixed DML)`
- `// Database.Stateful required so aggregates survive between execute() calls`

These are high-value "Warning of Consequences" comments in Clean Code's taxonomy, and they should never be deleted during refactors.

### TODO comments and Salesforce deployment

TODOs in Apex that say "fix this later" end up in production and get flagged by code scanners and reviewers. Use them, but scan regularly and clear them out before deploying to prod. Never leave TODOs in triggers, flows, or anywhere that touches user-visible data integrity.

### Commented-out code is still an abomination in Apex

Do not leave commented-out DML, SOQL, or Apex classes "just in case." Source control remembers. Commented-out SOQL is particularly dangerous because a future reader may re-enable it without realizing the field list is stale and the query now throws at runtime.

### Legal comments and AppExchange packages

If you are shipping a managed package on AppExchange, Salesforce requires specific headers and license metadata in the package, not in every source file. Keep per-file legal comments short and reference the package LICENSE file.

## Key Takeaways

1. **The best comment is the one you did not need to write** — because the code said it clearly.
2. **Comments lie the moment you stop maintaining them** — and you will stop maintaining them. Write fewer, and maintain the ones you keep.
3. **On this project, the Apex style guide's mandate to document every method overrides Clean Code's minimalism.** Write the header. Keep it honest. Match the "disembodied narrator" voice.
4. **Warning-of-consequences comments are MVPs in Apex** — platform gotchas (governor limits, DML-before-callout, sharing context) deserve explicit warnings and should survive refactors.
5. **Delete commented-out code, journal comments, attribution bylines, and noise comments on sight.** Source control already remembers.
6. **Before writing any comment, try the "extract to function" or "rename variable" refactor first.** If you can make the comment unnecessary, do so.
7. **TODOs are reminders, not excuses.** Sweep them before deploying to production.
