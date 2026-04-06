# Flow Security, Testing, and Null Handling

> **When to read this:** When choosing a flow's run context, testing flows from Apex test classes, debugging "insufficient privileges" errors, handling null values from Get Records elements, or running Salesforce Code Analyzer against flow metadata.

## Rules

- Always choose the most restrictive run context that still allows the flow to function. Default to User Context for screen flows. Escalate to System Context With Sharing or System Context Without Sharing only when users genuinely need access to data beyond their own permissions.
- Never set a screen flow to System Context Without Sharing when that flow is accessible by guest users or community/portal users -- this gives unauthenticated users unrestricted access to every record in the org.
- Never assume invocable Apex inherits the flow's run context. An `@InvocableMethod` in a class declared `with sharing` enforces sharing rules even if called from a System Context Without Sharing flow. Conversely, a class declared `without sharing` bypasses sharing even if called from a User Context flow. The Apex class's own sharing declaration always governs.
- Always remember that record-triggered flows, scheduled-triggered flows, and platform-event-triggered flows run in System Context Without Sharing by default -- they bypass CRUD, FLS, sharing rules, and profile object permissions entirely.
- Always add a Decision element immediately after every Get Records element to check whether the result is null before accessing any fields on it -- accessing a field on a null record variable causes "The flow failed to access the value for [variable] because it hasn't been set or assigned."
- Always check the "Assign null values to variable(s) if no records are found" checkbox on Get Records elements that use manual output, because when unchecked, the variable retains its previous value if no records match, causing silent stale-data bugs in loops.
- Never use ISNULL() on text fields in flow formulas -- text fields are never technically null, they are empty strings. Always use ISBLANK(), which handles both null and empty string.
- Always use BLANKVALUE() over NULLVALUE() in flow formulas because BLANKVALUE() handles both null and empty strings while NULLVALUE() only handles null.
- Always create Flow Tests for every record-triggered flow before deploying to production because Flow Tests run automatically during deployment validation and catch regressions.
- Always test flows that coexist with Apex triggers by running Apex unit tests that insert/update 200+ records inside `Test.startTest()` / `Test.stopTest()` -- declarative Flow Tests only run single-record scenarios and cannot validate bulk behavior.
- Always run Salesforce Code Analyzer with the Flow Scanner engine (`sf scanner run --rule-selector flow`) against flow metadata before deploying to production, especially for flows running in System Context Without Sharing.
- After the "Restrict User Access to Run Flows" enforcement (September-October 2024), always confirm that every user who needs to run a flow has the "Run Flows" permission explicitly assigned.

## How It Works

### The Three Run Contexts

Every flow executes in one of three run contexts that determine data visibility:

**User Context** enforces the running user's full security model: CRUD, FLS, sharing rules (OWD, role hierarchy, sharing rules, manual shares), and profile object permissions. If the user cannot see a field, the flow cannot read or write it.

**System Context With Sharing** bypasses CRUD and FLS -- the flow can read and write any object and field regardless of the user's profile. But it still enforces record-level sharing: OWD, role hierarchy, sharing rules, manual sharing. Use this when users need to interact with objects their profile does not grant access to, but you still want to restrict which specific records they see.

**System Context Without Sharing** bypasses everything: CRUD, FLS, sharing, all record-level access. The flow has unrestricted access to every record and every field. This is the most dangerous context.

| Security Layer | User Context | System With Sharing | System Without Sharing |
|---|---|---|---|
| Object CRUD permissions | Enforced | Bypassed | Bypassed |
| Field-Level Security (FLS) | Enforced | Bypassed | Bypassed |
| Record sharing (OWD, role hierarchy, sharing rules) | Enforced | Enforced | Bypassed |
| Profile object permissions | Enforced | Bypassed | Bypassed |

### Default Run Context by Flow Type

| Flow Type | Default Run Context | Can Be Changed? |
|---|---|---|
| Screen Flow | User Context | Yes -- User or System With Sharing (NOT Without Sharing for top-level screen flows) |
| Record-Triggered Flow (before-save) | System Context Without Sharing | No |
| Record-Triggered Flow (after-save) | System Context Without Sharing | No |
| Schedule-Triggered Flow | System Context Without Sharing | No |
| Platform Event-Triggered Flow | System Context Without Sharing | No |
| Autolaunched Flow (no trigger) | Inherits from caller | Yes -- can be set to User, System With Sharing, or System Without Sharing |
| Autolaunched Flow called from Apex | System Context Without Sharing (always) | No -- Apex overrides the flow's context setting |

**Critical for Apex developers:** When your Apex code launches a flow via `Flow.Interview`, the flow runs in System Context Without Sharing regardless of the flow's own configuration. However, a Winter '25 versioned update changed this: when an Apex class declared `with sharing` launches an autolaunched flow in the default context, the flow now respects sharing rules.

### How Run Context Interacts with Invocable Apex

This is where Apex developers frequently get confused. The flow's run context and the Apex class's sharing declaration are independent:

| Flow Context | Apex Class Declaration | What Happens |
|---|---|---|
| User Context | `with sharing` | Apex enforces sharing. Double-restricted. |
| User Context | `without sharing` | Apex ignores sharing. The Apex code sees records the user cannot. |
| System Without Sharing | `with sharing` | Apex enforces sharing even though the flow does not. The Apex code sees fewer records than the flow. |
| System Without Sharing | `without sharing` | Both wide open. No restrictions anywhere. |
| System Without Sharing | `inherited sharing` | Apex inherits System Without Sharing from the flow context. Effectively `without sharing`. |

The `inherited sharing` keyword on an Apex class means "use whatever sharing context I was called from." When called from a System Context Without Sharing flow, it behaves as `without sharing`. When called from a User Context flow, it behaves as `with sharing`. This is the safest choice for invocable methods that are called from multiple flow types.

**Recommendation for invocable methods:** Use `inherited sharing` when the method should respect whatever context the caller provides. Use `with sharing` when the method must always enforce sharing regardless of the calling flow. Use `without sharing` only when the method must access all records (e.g., a utility that counts global totals).

```apex
// this invocable method inherits sharing from whatever flow calls it
// if called from a User Context screen flow, it enforces sharing
// if called from a System Context record-triggered flow, it does not
public inherited sharing class FlexibleDataAction {

    @InvocableMethod(label='Flexible Data Lookup')
    public static List<Result> execute(List<Request> requests) {
        // SOQL here respects or ignores sharing based on the calling flow's context
        List<Account> accounts = [SELECT Id, Name FROM Account WHERE Id IN :accountIds];
        // ...
    }
}
```

### The Subflow Privilege Escalation Pattern

When a screen flow needs to perform a single privileged operation (e.g., creating a Case owned by a queue the user cannot see), isolate that operation in an autolaunched subflow running in System Context Without Sharing:

- Parent screen flow: **User Context** (default). Collects input, validates data.
- Child autolaunched subflow: **System Context Without Sharing**. Receives fields as input, creates the record, returns the Id.

This way, only the specific DML operation that needs elevation has elevated access. The parent flow's screens still respect the user's permissions.

**Security warning:** Subflows are technically independent flows that can be invoked via the API. If a subflow runs in System Context Without Sharing, an attacker who discovers its API name could invoke it directly. Always use the "Override default behavior" setting on privileged subflows and restrict access to specific profiles.

### Null vs Blank String vs Empty Collection

These three concepts are distinct in Salesforce Flow and require different handling:

**Null** means the variable has no value at all -- it was never set, or it was explicitly set to null. A record variable is null when Get Records returns no results (with the null-assignment checkbox checked). Checking `{!myText} Equals ""` does NOT catch null -- the null variable has no value to compare.

**Blank String ("")** means the variable holds a text value that is an empty string with zero characters. This happens when a user submits a screen flow with a text input left empty, or when a text field on a record exists but contains no data. `ISNULL({!myText})` returns false for a blank string, but `ISBLANK({!myText})` returns true for both null and blank.

**Empty Collection** means a collection variable has been initialized but contains zero elements. This is different from a null collection (which was never set). Use the **Is Empty** operator (Spring '25+) for empty collections, or check `{!myCollection.Count} Equals 0` in older API versions.

### The Get Records Null Checkbox

The "Assign null values to variable(s) if no records are found" checkbox on Get Records elements is one of the most important settings in Flow Builder. It controls what happens when a query returns zero results.

**When checked (recommended):** Output variables are set to null. You can check for null with a Decision element and handle the no-results case cleanly.

**When unchecked (dangerous in loops):** Output variables retain their previous value. This creates the classic stale-value-in-loop bug:

1. Flow loops through Contacts.
2. Inside the loop, Get Records looks up each Contact's related Account.
3. Contact #1 has an Account -- variable is set to that Account.
4. Contact #2 has no Account -- variable retains Contact #1's Account.
5. Flow processes Contact #2 using Contact #1's Account data. Silent, wrong.

With **Automatic Output Handling** (the modern default since API v56.0), the flow automatically sets the variable to null when no records match. There is no checkbox -- null assignment is built-in. Always check the variable for null using `{!Get_Records_Element} Is Null {!$GlobalConstant.True}`.

### Decision Element Operators for Null Checking

| Operator | Available Since | What It Checks | Use For |
|---|---|---|---|
| **Is Null** | Always | Whether the value is null (no value assigned) | Record variables, lookup fields, any variable that might never have been set |
| **Equals** (blank) | Always | Whether the value equals empty string "" | Text fields only -- does NOT catch null |
| **Is Blank** | Spring '25 | Whether a text value is null, empty, or whitespace-only | Preferred for text -- catches both null and "" in one check |
| **Is Empty** | Spring '25 | Whether a collection contains zero elements | Collection variables only |

Before Spring '25, checking for "no value" on a text field required two conditions: `Is Null` OR `Equals {!$GlobalConstant.EmptyString}`. The **Is Blank** operator eliminates this.

### Formula Functions for Null Handling

| Function | Behavior | Recommendation |
|---|---|---|
| `ISBLANK(expr)` | Returns TRUE if null, empty string, or whitespace | Use this. Covers all cases. |
| `ISNULL(expr)` | Returns TRUE only if null. False for empty string. | Avoid. Text fields are never technically null. |
| `BLANKVALUE(expr, default)` | Returns default if expr is null or blank | Use for safe defaults: `BLANKVALUE({!Amount}, 0)` |
| `NULLVALUE(expr, default)` | Returns default only if null, not for empty string | Avoid. Use BLANKVALUE instead. |

**Null propagation in math:** `null + 5 = null`. `null * 10 = null`. Always wrap operands: `BLANKVALUE({!Amount}, 0) + BLANKVALUE({!Tax}, 0)`.

**Text concatenation exception:** `null & "hello" = "hello"`. Concatenation treats null as empty string. This means concatenation will not fail, but can produce "Dear ," when a name is null.

## Testing Flows

### Declarative Flow Tests

Flow Tests are Salesforce's built-in declarative testing framework, GA from Winter '23. They only support record-triggered flows and Data Cloud-triggered flows. Screen flows, autolaunched flows, scheduled-triggered flows, and platform-event-triggered flows are NOT supported.

Each test defines:
1. **Initial record values** -- field values on the record that triggers the flow
2. **Updated record values** (optional) -- for update-triggered flows, the "after" field values
3. **Assertions** -- conditions that must be true after the flow executes

Flow Tests do NOT create real records. They simulate the trigger event in an isolated context. They also do NOT support asynchronous paths (scheduled paths, async after-save), do NOT support delete triggers, and do NOT support formulas or variables in initial values or assertions -- only literals.

**When they run:**
- Manually from Setup > Flow Tests
- Automatically during change set and Metadata API deployments
- They do NOT run on `sf project deploy start` unless test coverage thresholds are configured

### Testing Flows from Apex Test Classes

For Apex developers, the most powerful flow testing approach is Apex unit tests that trigger the flow indirectly by performing DML, then assert the flow's outcomes. This tests the flow AND the Apex trigger in the same transaction -- exactly how they run in production.

```apex
@IsTest
private class AccountFlowAndTriggerTest {

    @TestSetup
    static void setupTestData() {
        // create test data that the flow and trigger will process
        List<Account> accounts = new List<Account>();
        for (Integer i = 0; i < 200; i++) {
            accounts.add(new Account(
                Name = 'Bulk Test ' + i,
                Industry = 'Technology',
                BillingState = 'CA'
            ));
        }
        insert accounts;
    }

    @IsTest
    static void testBulkInsert_flowAndTriggerCoexist() {
        // re-query to get fresh records
        List<Account> accounts = [SELECT Id, Name, Region__c FROM Account];
        System.assertEquals(200, accounts.size(), 'Should have 200 test accounts');

        Test.startTest();
        // update all 200 records -- this fires:
        // 1. before-save flow (step 4)
        // 2. before Apex trigger (step 5)
        // 3. after Apex trigger (step 10)
        // 4. after-save flow (step 15)
        for (Account a : accounts) {
            a.Industry = 'Finance';
        }
        update accounts;
        Test.stopTest();

        // assert flow outcomes
        List<Account> results = [SELECT Id, Region__c, Custom_Field__c FROM Account WHERE Id IN :accounts];
        for (Account a : results) {
            // assert that the before-save flow set Region__c
            System.assertNotEquals(null, a.Region__c,
                'Before-save flow should have set Region__c');
            // assert that the after-save flow or trigger set Custom_Field__c
            System.assertEquals('Processed', a.Custom_Field__c,
                'Flow/trigger should have set Custom_Field__c on all 200 records');
        }
    }

    @IsTest
    static void testAsStandardUser_flowRespectsSharing() {
        // test the flow's behavior under a restricted user's permissions
        User standardUser = [SELECT Id FROM User
            WHERE Profile.Name = 'Standard User' AND IsActive = true LIMIT 1];

        System.runAs(standardUser) {
            Account acc = new Account(Name = 'Standard User Test', BillingState = 'NY');
            insert acc;

            Account result = [SELECT Id, Region__c FROM Account WHERE Id = :acc.Id];
            System.assertNotEquals(null, result.Region__c,
                'Flow should set Region__c even for standard user');
        }
    }
}
```

### Testing Flows with Flow.Interview from Apex

You can directly invoke autolaunched flows and screen flows from Apex test classes using the `Flow.Interview` class. This is useful for testing flows that are NOT record-triggered (e.g., screen flows, autolaunched flows called from buttons).

```apex
@IsTest
static void testAutolanchedFlowDirectly() {
    // create the input variables map
    Map<String, Object> flowInputs = new Map<String, Object>{
        'recordId' => testAccountId,
        'selectedContactId' => testContactId,
        'approvalAction' => 'Approve'
    };

    Test.startTest();
    // create and start the flow interview
    // the string is the flow's API name (not the label)
    Flow.Interview myFlow = Flow.Interview.createInterview(
        'Process_Purchase_Order', flowInputs
    );
    myFlow.start();
    Test.stopTest();

    // read output variables from the flow
    Boolean isSuccess = (Boolean) myFlow.getVariableValue('isSuccess');
    String errorMessage = (String) myFlow.getVariableValue('errorMessage');

    System.assertEquals(true, isSuccess, 'Flow should succeed');
    System.assertEquals(null, errorMessage, 'No error expected');

    // assert side effects (records created, fields updated, etc.)
    Purchase_Order__c po = [SELECT Status__c, Digital_Authorization__c
        FROM Purchase_Order__c WHERE Id = :testAccountId];
    System.assertEquals('Approved', po.Status__c, 'Flow should have updated status');
}
```

**Important:** When a flow is invoked from Apex via `Flow.Interview`, it runs in System Context Without Sharing regardless of the flow's own setting (see the run context table above). If you need to test the flow's behavior under a specific user's permissions, wrap the `Flow.Interview` call in `System.runAs()`.

```apex
@IsTest
static void testFlowAsPortalUser() {
    User portalUser = createTestPortalUser();

    System.runAs(portalUser) {
        Map<String, Object> inputs = new Map<String, Object>{
            'recordId' => testCaseId
        };

        Flow.Interview caseFlow = Flow.Interview.createInterview(
            'Submit_Case_Flow', inputs
        );

        // this should succeed -- the flow handles privilege escalation via subflow
        caseFlow.start();

        Boolean success = (Boolean) caseFlow.getVariableValue('isSuccess');
        System.assertEquals(true, success,
            'Flow should succeed for portal user via privilege escalation subflow');
    }
}
```

### Debug Logs for Flow Execution

When Flow Builder's Debug tool is insufficient (e.g., debugging flows triggered by integrations, batch jobs, or platform events), use Salesforce debug logs.

**Setup:** Setup > Debug Logs > New Trace Flag
- Traced Entity: the user whose actions trigger the flow
- Debug Level: create a custom level named "FlowDebug" with these settings:

| Category | Level | Why |
|---|---|---|
| Workflow | FINER | Captures FLOW_START_INTERVIEW, FLOW_ELEMENT_BEGIN, FLOW_VALUE_ASSIGNMENT with full variable values |
| Apex Code | DEBUG | Captures invocable action execution |
| Database | INFO | Shows DML and SOQL from flow elements |
| Validation | INFO | Shows validation rule results that may block flow DML |
| All others | ERROR or NONE | Reduces noise so the log does not hit the 20 MB limit |

**Critical:** Set Workflow to FINER, not FINE. FINE shows which elements executed but not the variable values. FINER shows both. Without variable values, you are debugging blind.

**Key debug log events:**

| Event | What It Shows |
|---|---|
| `FLOW_START_INTERVIEW` | Flow interview started. Shows flow API name and version. |
| `FLOW_ELEMENT_BEGIN` | Entering an element. Shows element type and API name. |
| `FLOW_VALUE_ASSIGNMENT` | Variable assigned a value. Shows variable name and new value (FINER only). |
| `FLOW_ELEMENT_ERROR` | Element threw an error. Shows error message and fault path. |
| `FLOW_ELEMENT_FAULT` | Flow entered a fault path. |
| `FLOW_ELEMENT_LIMIT_USAGE` | CPU time and governor limit consumption for that element. |

**Pro tip:** Search the raw log for `FLOW_ELEMENT_ERROR` first. If present, it pinpoints exactly which element failed and why. Then trace backward through `FLOW_VALUE_ASSIGNMENT` events to see what data the flow was working with when it failed.

### Flow Scanner / Salesforce Code Analyzer Integration

Run Salesforce Code Analyzer against your flow metadata to catch security and performance issues before they reach production:

```bash
sf scanner run --rule-selector flow --target force-app/main/default/flows/
```

Key rules the scanner checks:

| Rule | Severity | What It Detects |
|---|---|---|
| CRUD in System Context Without Sharing | High (2) | User-controlled data in CRUD operations within System Without Sharing flows -- privilege escalation risk |
| CRUD in System Context With Sharing | Low (4) | User-controlled data in CRUD operations within System With Sharing flows |
| Database Operations in Loops | High (2) | DML or SOQL inside loops -- governor limit risk |
| Missing Fault Handlers | High (2) | CRUD elements without fault paths |
| Circular Subflow References | Critical (1) | Subflows creating circular dependencies |

**Integrating with CI/CD:** Add the scanner to your deployment pipeline alongside Apex test execution. Flow metadata lives in `force-app/main/default/flows/` as XML files. The scanner analyzes the XML directly -- no org connection needed.

```bash
# run flow scanner + Apex scanner in CI pipeline
sf scanner run --rule-selector flow --target force-app/main/default/flows/ --format json --outfile flow-scan-results.json
sf scanner run --rule-selector pmd --target force-app/main/default/classes/ --format json --outfile apex-scan-results.json
```

### Clearing Field Values in Flows

Clearing a field (setting it to null/empty) in a flow requires different techniques depending on the field type:

**Text fields:** Use `{!$GlobalConstant.EmptyString}` in an Assignment element.

**Date, Number, and Lookup fields:** Create a Formula resource of the correct type (Date, Number, etc.) whose formula is simply `NULL`. Then assign that formula resource to the field. Date and number fields do not accept `{!$GlobalConstant.EmptyString}`.

**Lookup fields (alternative):** You can also use `{!$GlobalConstant.EmptyString}` for lookup fields -- Salesforce treats it as clearing the relationship.

This distinction matters when building flows that reset fields conditionally. Apex developers are used to `record.Field__c = null` working universally; in flows, the mechanism depends on the data type.

### Testing Strategies for Apex Developers

**Strategy 1: Indirect DML Testing (most common).**
Insert or update records in Apex to trigger record-triggered flows. Assert the flow's side effects (field values, child records, etc.). This tests the flow AND any Apex triggers in the same transaction, exactly as they run in production. Use 200+ records to catch bulkification issues.

**Strategy 2: Direct Flow.Interview Testing.**
Use `Flow.Interview.createInterview()` to invoke autolaunched or screen flows directly. Pass inputs, start the interview, and read outputs with `getVariableValue()`. This isolates the flow from trigger side effects.

**Strategy 3: System.runAs() for Security Testing.**
Wrap DML or Flow.Interview calls in `System.runAs(someUser)` to test how the flow behaves under different profiles. Essential for testing User Context flows where sharing and FLS matter.

**Strategy 4: Negative Testing.**
Intentionally trigger error conditions: delete a parent record before the flow looks it up, set values that violate validation rules, leave required fields blank. Verify that fault paths and null checks handle these gracefully.

**Strategy 5: Limit Regression Testing.**
Assert that combined flow + trigger SOQL and DML consumption stays below a budget threshold (e.g., 70% of limits). This catches regressions when someone adds a new query to a flow. See the governor limits doc for a full example.

## Common Error Messages and What They Mean

### "The flow failed to access the value for [variable] because it hasn't been set or assigned"

**Cause:** The flow reached an element that references a record variable from a Get Records element that returned no results. The variable is null, and accessing any field on it throws this error.

**Fix:** Add a Decision element immediately after every Get Records. Check `{!Get_Records_Element} Is Null {!$GlobalConstant.True}`. Route the null path to a graceful exit or fallback logic.

### "REQUIRED_FIELD_MISSING"

**Cause:** A Create/Update Records element is saving a record without a required field. Required fields come from schema-level requirements, validation rules, and record-type-specific page layouts.

**Fix:** Ensure the flow populates every required field. Pay special attention to fields conditionally required by validation rules -- these are easy to miss. Remember that validation rules run AFTER before-save flows and before-save triggers (step 7), so the flow's field values will be evaluated.

### "INVALID_CROSS_REFERENCE_KEY"

**Cause:** A lookup field is being set to an ID that Salesforce cannot resolve -- the record was deleted, the ID is hardcoded from a different org, or the ID prefix does not match the expected object.

**Fix:** Never hardcode record IDs in flows. Use Get Records to dynamically look up the target record and verify it exists.

### "The flow tried to update these records: null"

**Cause:** An Update Records element received a null record variable. The word "null" replaces where the record ID would normally appear.

**Fix:** Trace backward to find where the variable should have been populated. Usually a Get Records returned no results. Add null checks before every Update Records element.

### "MIXED_DML_OPERATION"

**Cause:** Setup object DML and non-setup object DML in the same transaction. This can happen entirely within a flow, entirely within Apex, or split across both.

**Fix:** Move one DML operation to an async context. See the governor limits doc for Scheduled Path, Platform Event, and @future solutions.

### "UNABLE_TO_LOCK_ROW"

**Cause:** Another transaction has a lock on the record. Common when multiple automations or integrations update the same record simultaneously.

**Fix:** Add a fault path and implement retry logic or a user-friendly error. Consider moving the update to an asynchronous path to reduce lock contention.

## Common Mistakes

1. **Setting an entire screen flow to System Context Without Sharing instead of isolating privileged operations in a subflow.** Every user who can run the flow gets unrestricted access to all org data. Fix: Keep the parent screen flow in User Context. Move only the specific DML that needs elevation into an autolaunched subflow in System Context Without Sharing.

2. **Assuming invocable Apex inherits the flow's run context.** A `without sharing` Apex class called from a User Context flow still bypasses sharing. A `with sharing` class called from a System Without Sharing flow still enforces sharing. Fix: Always check the sharing declaration on the Apex class itself. Use `inherited sharing` when the method should respect whatever context the caller provides.

3. **Accessing fields on a Get Records result without checking for null first.** If the query returns no records, any field access throws "hasn't been set or assigned." Fix: Add a Decision element checking Is Null immediately after every Get Records.

4. **Leaving the "Assign null values" checkbox unchecked in a loop.** The variable retains the previous iteration's value when the query returns no results. Silent, wrong data. Fix: Always check the checkbox when using manual output. Or use Automatic Output Handling (the modern default), which automatically sets to null.

5. **Using ISNULL() instead of ISBLANK() on text fields.** Text fields are never technically null -- they are empty strings. ISNULL() on a text field always returns FALSE. Fix: Use ISBLANK(), which returns TRUE for both null and empty string.

6. **Only testing the happy path with declarative Flow Tests.** A flow that works for one well-formed record will fail with null values, missing related records, or bulk data loads. Fix: Test edge cases. Test null inputs. Test 200+ records via Apex unit tests.

7. **Assuming Flow Tests cover async paths.** Scheduled paths and asynchronous after-save paths are NOT executed by declarative Flow Tests. Fix: Use Apex unit tests with `Test.startTest()` / `Test.stopTest()` to force async execution, then assert outcomes.

8. **Not testing the flow + trigger interaction in bulk.** A flow and a trigger each work independently. Together, they exceed governor limits at 200 records. Fix: Write Apex tests that insert/update 200+ records and assert outcomes from both the flow and the trigger.

9. **Setting Workflow log level to FINE instead of FINER.** FINE shows which elements executed. FINER shows variable values at each step. Without values, debugging is guesswork. Fix: Create a custom debug level with Workflow at FINER and everything else at ERROR.

10. **Not re-verifying "Override default behavior" after saving a new flow version.** A documented known issue silently resets this checkbox when a new version is created. Fix: Always check the Flow Detail page after activating a new version.

11. **Forgetting that screen components query in User Context even when the flow is in System Context.** The Lookup, Address, Dependent Picklist, File Upload, and Record Field components always query using the running user's permissions. If a user cannot see an Account due to sharing, the Lookup will not return it even if the flow runs in System Context. Fix: Be aware of this limitation and test with restricted user profiles.

12. **Performing math on potentially null values without BLANKVALUE().** `{!Amount} + {!Discount}` returns null if either is null. Fix: Wrap each operand: `BLANKVALUE({!Amount}, 0) + BLANKVALUE({!Discount}, 0)`.

13. **Forgetting that the "Automated Process" user runs scheduled and platform-event flows.** When a schedule-triggered or platform-event-triggered flow executes, the running user is "Automated Process," not a real person. This affects audit trails (CreatedById, LastModifiedById) and any logic checking `$User` fields.

14. **Passing unvalidated user input into SOQL filters in system-context flows.** In a System Without Sharing flow, a Get Records with user-supplied filter values can return any record in the org. This is the privilege escalation vulnerability that Flow Scanner detects. Fix: Always validate and constrain filter values before using them in data operations.

15. **Relying solely on declarative Flow Tests for complex flows.** Flows that call Apex actions, interact with external services, or process bulk data need Apex unit tests in addition to declarative tests. Declarative tests are regression checks, not comprehensive integration tests.

## See Also

- [Flow + Apex Coexistence Patterns](./flow-apex-coexistence.md) (order of execution, invocable method error handling, automation density)
- [Flow Governor Limits](./flow-governor-limits.md) (shared limits, DML-in-loop anti-pattern, Transform element, mixed DML solutions)
- [Invocable Methods](../apex-fundamentals/invocable-methods.md) (building the flow-to-Apex bridge)
- [Apex Fundamentals](../apex-fundamentals/apex-fundamentals.md) (sharing keywords, test patterns)
- Salesforce Help: [Flow Run Context](https://help.salesforce.com/s/articleView?id=platform.flow_distribute_context.htm&language=en_US&type=5)
- Salesforce Help: [Testing Your Flow Before Activation](https://help.salesforce.com/s/articleView?id=platform.flow_concepts_testing.htm&language=en_US&type=5)
- Salesforce Help: [Flow Scanner Rules Reference](https://developer.salesforce.com/docs/platform/salesforce-code-analyzer/guide/rules-flow.html)
- Salesforce Help: [Restrict User Access to Run Flows](https://help.salesforce.com/s/articleView?id=release-notes.rn_automate_flow_release_update_restrict_user_access.htm&language=en_US&release=258&type=5)
- Salesforce Help: [Flow Operators in Decision Elements](https://help.salesforce.com/s/articleView?id=platform.flow_ref_operators_condition.htm&language=en_US&type=5)
