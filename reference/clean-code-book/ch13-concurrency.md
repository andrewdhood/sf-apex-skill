# Chapter 13: Concurrency

> **Source:** Clean Code by Robert C. Martin (Prentice Hall, 2008), chapter by Brett L. Schuchert
> **When to read this:** When writing async Apex (Queueable, Batch, Future, Platform Events), dealing with `UNABLE_TO_LOCK_ROW` errors, handling multi-user record contention, or designing approval/locking flows.

---

## Key Principles (from the book, as written)

1. **Concurrency is a decoupling strategy** — it separates *what* gets done from *when* it gets done. The goal is structural clarity and throughput, not just speed.
2. **Concurrency is hard.** Code that works single-threaded can be silently broken under load. Concurrency bugs are rarely repeatable, so they get dismissed as one-offs when they are really defects.
3. **Single Responsibility Principle applies hard.** Keep concurrency-aware code strictly separate from concurrency-ignorant code. Thread management is its own reason to change.
4. **Limit the scope of shared data.** The more places shared data can be updated, the more places you can forget to protect — and the more places a bug can hide. Encapsulate ruthlessly.
5. **Use copies of data.** Avoiding shared state is better than synchronizing access to it. Copy, let threads work locally, merge at the end.
6. **Threads should be as independent as possible.** Each worker ideally has everything it needs passed in, works on local data, writes no shared state. No synchronization needed when there is nothing shared.
7. **Know your library.** Use thread-safe collections, executor frameworks, and nonblocking primitives rather than hand-rolling locks.
8. **Know your execution models.** Producer-Consumer, Readers-Writers, and Dining Philosophers cover most real-world concurrency problems. Learn the canonical solutions.
9. **Beware dependencies between synchronized methods.** If two `synchronized` methods on the same shared object can be called in sequence, the whole sequence is not atomic. Use client-based, server-based, or adapted-server locking.
10. **Keep synchronized sections small.** Locks are expensive and contention degrades throughput. Protect only what must be protected.
11. **Writing correct shut-down code is hard.** Graceful termination of concurrent systems is notoriously buggy — design it early.
12. **Testing threaded code requires different techniques.** Run tests many times, under varied configurations and loads, on all target platforms. Never ignore a failure as a "one-off." Get non-threaded code working first. Make threaded code pluggable and tunable. Instrument the code to force failures (jiggling).

---

## Detailed Notes

### Why Concurrency Exists

Concurrency decouples *what* from *when*. Structural benefits are often more important than raw throughput — systems become collaborating workers rather than one giant main loop. Response time and throughput constraints can also force concurrency (an aggregator hitting 1000 websites serially can't fit in 24 hours).

### Myths Martin Debunks

- "Concurrency always improves performance." No — only when there's substantial wait time to amortize.
- "Design doesn't change." Wrong — concurrent design is fundamentally different.
- "You don't need to understand concurrency if a container handles it." Wrong — containers hide concurrency but don't eliminate the defects you can introduce.

### The Trivial Example That Isn't Trivial

A single `return ++lastIdUsed;` in Java has 12,870 possible execution paths across two threads because `++` is not atomic at the bytecode level. Change `int` to `long` and it becomes 2,704,156 paths. Most produce correct results. Some don't. That's the whole problem with concurrency: the failure modes are rare and non-deterministic.

### Defense Principles (Martin's Recommendations)

- **SRP:** "Keep your concurrency-related code separate from other code." Thread-aware code has its own lifecycle, challenges, and failure modes.
- **Limit Scope of Data:** "Take data encapsulation to heart; severely limit the access of any data that may be shared."
- **Use Copies of Data:** The GC overhead of copies is almost always cheaper than lock contention.
- **Independent Threads:** "Attempt to partition data into independent subsets that can be operated on by independent threads."

### Execution Model Vocabulary

| Term | Meaning |
|---|---|
| **Bound Resources** | Fixed-capacity shared resources (DB connections, buffers) |
| **Mutual Exclusion** | Only one thread can access a shared resource at a time |
| **Starvation** | A thread is indefinitely prevented from progressing |
| **Deadlock** | Two+ threads wait on each other forever |
| **Livelock** | Threads keep acting but make no progress |

### The Three Canonical Problems

- **Producer-Consumer:** Producers write to a queue, consumers read from it. Both must signal when the queue state changes.
- **Readers-Writers:** Many readers, occasional writers. Balancing throughput vs. staleness vs. starvation is the core tension.
- **Dining Philosophers:** Competing for multiple resources where each acquisition can block. Naive implementations deadlock or starve.

### Testing Threaded Code (the fine-grained checklist)

- Treat spurious failures as candidate threading issues — **do not ignore one-offs.**
- Get nonthreaded code working first. Separate threading bugs from logic bugs.
- Make threaded code pluggable (one thread, several threads, test doubles).
- Make it tunable (adjust thread counts at runtime).
- Run with more threads than processors to force task-swapping.
- Run on all target platforms.
- Instrument the code to force failures — hand-coded `yield()`/`sleep()` calls, or an automated "jiggling" framework that randomly perturbs execution order.

---

## Salesforce/Apex Caveats

**⚠ Remember: Salesforce platform requirements ALWAYS override Clean Code principles when they conflict.**

**The entire chapter is, at a literal level, inapplicable to Apex.** Apex is a managed runtime with no user-controllable threads. You cannot `synchronized`, you cannot `Thread.start()`, you cannot `wait()/notify()`, and you cannot write code that has two threads sharing an in-memory field. Every Apex transaction executes on a single thread, in a single context, with its own governor limits and its own copy of state. When the transaction ends, all static state goes away.

**But concurrency-LIKE problems absolutely exist in Apex**, and Martin's underlying principles — decoupling, isolation, copying, avoiding shared mutable state, small critical sections — map onto them almost perfectly if you translate the vocabulary. This section is the real content of this chapter for an Apex developer.

### What "Concurrency" Actually Means in Apex

Apex concurrency-like concerns fall into five categories:

1. **Record-level contention** between simultaneous user transactions (row locks, `UNABLE_TO_LOCK_ROW`)
2. **Optimistic concurrency** from two users editing the same record
3. **Async work decoupling** (Queueable, Batch, Future, Scheduled, Platform Events)
4. **Governor limit isolation** between transactions
5. **Mixed DML / setup-object ordering** problems

None of these are "threads" in Martin's sense. All of them benefit from his design principles.

### 1. Row Locking and `UNABLE_TO_LOCK_ROW`

The closest analog to Martin's "two threads stepping on each other." When two Apex transactions try to update the same record (or a related parent in the sharing hierarchy), the second one waits up to ~10 seconds for the first lock to release. If it doesn't, you get `UNABLE_TO_LOCK_ROW`.

**Common triggers of this error:**
- Updating child records in parallel batches where all children share one parent Account
- Trigger + Process Builder + Flow all updating the same record
- Roll-up summary recalculation cascading up a hierarchy
- `FOR UPDATE` clauses held across a slow callout (which you can't actually do — callouts before DML — but the pattern of holding a lock too long applies)

**Apex translation of Martin's "keep critical sections small":**

```apex
// bad: locks the parent for the entire batch
public class BadBatch implements Database.Batchable<SObject> {
    public void execute(Database.BatchableContext bc, List<Contact> scope) {
        // every contact in scope shares the same Account, so every chunk
        // locks the same parent row — guaranteed row lock contention
        for (Contact c : scope) {
            c.Processed__c = true;
        }
        update scope;
        // then we do 15 seconds of other work while holding implicit locks
    }
}

// good: sort the scope so each chunk contains different parents,
// and keep the DML section as tight as possible
public class GoodBatch implements Database.Batchable<SObject> {
    public Database.QueryLocator start(Database.BatchableContext bc) {
        // remember: ORDER BY AccountId groups children by parent so chunks
        // don't straddle parents and cause lock contention across batches
        return Database.getQueryLocator(
            'SELECT Id, AccountId, Processed__c FROM Contact ' +
            'WHERE Processed__c = false ORDER BY AccountId'
        );
    }

    public void execute(Database.BatchableContext bc, List<Contact> scope) {
        // begin updating contacts — do the minimum inside the DML window
        for (Contact c : scope) {
            c.Processed__c = true;
        }
        update scope;
    }

    public void finish(Database.BatchableContext bc) {}
}
```

**Martin's "Limit the scope of data" translates directly:** the smaller the set of records a transaction touches, the fewer lock conflicts it can have.

### 2. `FOR UPDATE` — Apex's Explicit Locking

`SELECT ... FOR UPDATE` is the one lock primitive Apex exposes. It locks the returned rows for the duration of the transaction.

```apex
// use FOR UPDATE when you need read-then-write semantics without
// another transaction sneaking in and overwriting your assumptions
public static void assignNextTicketNumber(Id caseId) {
    // lock the counter row so nobody else can grab the same number
    Ticket_Counter__c counter = [
        SELECT Id, Next_Number__c
        FROM Ticket_Counter__c
        WHERE Name = 'Global'
        LIMIT 1
        FOR UPDATE
    ];

    Case c = new Case(Id = caseId, Ticket_Number__c = counter.Next_Number__c);
    counter.Next_Number__c += 1;

    // keep the critical section short: update both and let the lock release
    update c;
    update counter;
}
```

**Martin's "keep synchronized sections small" applies literally here.** Never put a callout, a long loop, or a Queueable enqueue between the `FOR UPDATE` query and the corresponding `update`. Those extend the window during which other transactions are blocked.

**Never do this:**
```apex
// BAD — the callout extends the lock window by 120 seconds
List<Account> accts = [SELECT Id FROM Account WHERE ... FOR UPDATE];
HttpResponse r = http.send(req); // other transactions waiting the whole time
update accts;
```

### 3. Optimistic Locking (User Edit Collisions)

When two users load the same record in a UI and both save, Salesforce uses `LastModifiedDate` as an optimistic concurrency token. If your Apex `update` passes a stale `LastModifiedDate`, the platform throws an `INVALID_FIELD_FOR_INSERT_UPDATE` or equivalent. In LWC + Apex patterns, this surfaces as a save conflict that must be presented to the user.

**Design rule:** if you're building a screen where two users might edit the same record, either (a) re-query immediately before updating to merge changes, or (b) expose the conflict to the user and let them resolve it. Never silently overwrite.

### 4. Async Apex Is Your "Concurrency" Model

Martin's decoupling principle — *separate what from when* — is the whole point of async Apex. Use it when:

| Situation | Async Mechanism |
|---|---|
| Need to do work after current DML commits (callouts, etc.) | `@future(callout=true)` or Queueable |
| Chained jobs with state | Queueable implementing `Database.AllowsCallouts`, chained via `System.enqueueJob()` |
| Large data volume | `Database.Batchable<SObject>` |
| Scheduled work | `Schedulable` + `System.schedule()` |
| Fan-out decoupling between processes | Platform Events with `after insert` trigger subscribers |

**Martin's "threads should be independent" = each async job should be self-contained.** Never assume the state of another async job. Pass IDs, re-query fresh, don't rely on static variables surviving between transactions (they don't).

```apex
// good pattern: queueable carries its own input state, re-queries fresh,
// writes only to its own scope
public class AccountScoringJob implements Queueable {
    private Set<Id> accountIds;

    public AccountScoringJob(Set<Id> accountIds) {
        // remember: pass IDs, not sObjects — sObjects serialize awkwardly
        // and may be stale by the time the job runs
        this.accountIds = accountIds;
    }

    public void execute(QueueableContext ctx) {
        // re-query so we see whatever state exists NOW, not when we enqueued
        List<Account> accts = [
            SELECT Id, AnnualRevenue, NumberOfEmployees
            FROM Account
            WHERE Id IN :accountIds
        ];

        for (Account a : accts) {
            a.Health_Score__c = computeScore(a);
        }
        update accts;
    }

    private static Decimal computeScore(Account a) {
        // pure function, no shared state — easy to unit test
        return (a.AnnualRevenue ?? 0) / 1000 + (a.NumberOfEmployees ?? 0);
    }
}
```

### 5. Governor Limits Give You Free Isolation — Don't Fight Them

Every Apex transaction is bounded by governor limits (SOQL, DML, CPU, heap). This is Martin's "independent threads" taken to a hard extreme: each transaction cannot even see another transaction's counters. The platform enforces isolation you would otherwise have to design for.

**What this buys you:** you almost never need to worry about shared mutable in-memory state across transactions. It literally cannot exist.

**What this costs you:** when you need coordination (e.g., "only one of these jobs at a time"), you have to build it with records, Platform Cache, or Big Objects — you cannot use in-memory locks.

### 6. Approval Process Locks

Records pending approval are automatically record-locked by the platform. Apex `update` on a locked record throws `ENTITY_IS_LOCKED`. If your trigger or automation needs to update a record that's pending approval, you must call `Approval.unlock()` first, do the update, and then `Approval.lock()` again — and even that is a smell.

```apex
// only do this when you genuinely must bypass an approval-pending lock
List<Id> idsToUpdate = new List<Id>{ recordId };
Approval.unlock(idsToUpdate);
try {
    update someRecord;
} finally {
    // always re-lock, even on exception
    Approval.lock(idsToUpdate);
}
```

### 7. Mixed DML Ordering (Setup vs. non-Setup sObjects)

Not strictly concurrency, but a related isolation problem: you cannot DML a User/Group/PermissionSet and a non-setup sObject (Account, Contact, etc.) in the same transaction. If you must, use `@future` to defer one of them. This is Apex's way of saying "these are in separate consistency domains and we won't let you mix them."

### 8. Platform Events as Producer-Consumer

Platform Events are Martin's producer-consumer pattern, built into the platform. A producer publishes an event; one or more subscribers (Apex triggers, Flows, external systems) consume it asynchronously. The "queue" is managed by Salesforce. You get:

- Decoupling (producer doesn't know consumers)
- Async execution (consumer runs in its own transaction with fresh limits)
- Automatic retry (subscribers get retry-on-failure semantics)

**Use Platform Events specifically when:** you need to escape the current transaction's governor limits, or you need to hand work off to a different system, or you want fire-and-forget behavior without Queueable chaining.

```apex
// publish-subscribe — producer side
Order_Shipped__e evt = new Order_Shipped__e(Order_Id__c = orderId);
Database.SaveResult sr = EventBus.publish(evt);

// consumer side is a trigger on Order_Shipped__e (after insert)
// it runs in its OWN transaction with a fresh governor-limit allotment
```

### 9. Testing Async Apex

Martin says "make threaded code pluggable and tunable; run tests many times." The Apex equivalent:

- **`Test.startTest()` / `Test.stopTest()`** — forces queued async work to execute synchronously within the test, so you can assert on its effects. Use this religiously when testing Queueable/Batch/Future.
- **Don't test the scheduling** — test the pure logic separately, the way Martin says to separate thread-aware code from thread-ignorant POJOs. Your Queueable's `execute` should delegate to a normal static method that you can unit test without the async wrapper.
- **Avoid `SeeAllData=true`** — this is the closest Apex equivalent to "global mutable state leaking into tests." Always create your own test data.
- **Use `@TestSetup`** to create shared test data once per class. Martin's "use copies of data" — each test method gets its own view of the setup data.

```apex
// separate the thread-aware shell from the thread-ignorant core
public class AccountScoringJob implements Queueable {
    private Set<Id> accountIds;
    public AccountScoringJob(Set<Id> accountIds) { this.accountIds = accountIds; }

    public void execute(QueueableContext ctx) {
        // delegate immediately — the queueable is just a shipping container
        AccountScoringService.scoreAccounts(accountIds);
    }
}

// all logic lives here, unit-testable without enqueuing anything
public class AccountScoringService {
    public static void scoreAccounts(Set<Id> accountIds) {
        // ... real work ...
    }
}
```

### 10. Apex-Specific Concurrency Smells to Watch For

- **Static variables used to "pass data" between trigger invocations.** They persist only within one transaction. If you're relying on them across async boundaries, you have a bug.
- **Recursive trigger guards using booleans.** They work within a transaction but are invisible to concurrent transactions. Two users hitting the same trigger simultaneously don't share your guard.
- **Long-running Queueable chains that update overlapping records.** Each link in the chain can collide with the next if they touch the same records.
- **`@future` methods with no idempotency.** If Salesforce retries a future (it can, on failure), non-idempotent logic will double-apply.
- **Callouts inside loops.** Not concurrency per se, but it blows the 100-callout limit and turns into a de facto denial-of-service against yourself.

---

## Key Takeaways

1. **Literal Clean Code concurrency doesn't apply to Apex** — you have no threads, no `synchronized`, no shared in-memory state across transactions.
2. **The principles absolutely do apply** — decouple *what* from *when*, isolate state, keep critical sections small, test the logic separately from the async wrapper.
3. **`UNABLE_TO_LOCK_ROW` is your real concurrency bug.** Sort batch scopes by parent, keep DML sections tight, never hold `FOR UPDATE` across a callout.
4. **Async Apex is the platform's concurrency model.** Use Queueable for chained work with state, Batch for volume, Platform Events for decoupled producer-consumer, Future only for the narrow cases it's good at.
5. **Pass IDs, not sObjects, between async boundaries.** Re-query fresh inside the async context.
6. **Separate thread-aware shells from thread-ignorant logic.** The Queueable should be a thin wrapper around a static service method that you can unit test directly.
7. **`Test.startTest()/stopTest()` is your only real tool for testing async.** Use it.
8. **Governor limits give you isolation for free.** Don't design around their absence — lean into them.
9. **Approval-locked records, mixed DML, and Platform Event retries are Apex's concurrency edge cases.** Know them by name so you recognize the failure modes.
10. **When Clean Code says "know your library," in Apex that means know `Database.Batchable`, `Queueable`, `@future`, `Schedulable`, Platform Events, `FOR UPDATE`, and `Approval.lock/unlock`.** These are your entire concurrency toolkit.
