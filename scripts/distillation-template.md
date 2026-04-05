# Distillation Template

Every reference doc in `reference/` MUST follow this format. The goal is to produce
**instruction-oriented** content that changes downstream code generation quality.
NOT summaries. NOT documentation rewrites. INSTRUCTIONS.

---

## Template

```markdown
# [Topic Title]

> **When to read this:** [1-sentence trigger]

## Rules

- When [situation], always [do this] because [reason].
- Never [anti-pattern] because [consequence].

## How It Works

[2-4 paragraphs explaining the mechanism]

## Code Examples

[Real, working Apex/Visualforce/XML. Not pseudocode.]

## Common Mistakes

1. **[Mistake]** — [Why it happens]. Fix: [concrete fix].

## See Also

- [Related topic](../category/related-doc.md)
```

## Key Principles

1. **Instruction over description.** "Always use `with sharing` on controllers
   that serve Visualforce pages because the PDF renders in the running user's
   context" beats "Sharing rules affect Visualforce page rendering."

2. **Concrete over abstract.** Include actual class names, method signatures,
   governor limit numbers, and annotation syntax.

3. **Failure modes are gold.** Every "Common Mistakes" section should contain
   things that Claude (and junior devs) actually get wrong.

4. **Governor limits are critical.** Always mention the specific limit number.

5. **Code examples must be valid Apex.** Not Java. No `System.out.println`.
   Use `System.debug()`. Use Apex collections (`List<>`, `Map<>`, `Set<>`).
   Use SOQL syntax, not SQL.

6. **Test patterns alongside implementation.** Show how to test every pattern.
