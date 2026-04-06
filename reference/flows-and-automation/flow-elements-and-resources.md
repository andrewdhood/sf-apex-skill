# Flow Builder Elements & Resources Reference (for Apex Developers)

> **When to read this:** When building any flow and you need to know which element to use, how to configure it, what resources are available, or how formula syntax works -- especially when deciding whether a Flow element or Apex code is the right tool for a given operation.

## Rules

- Never place DML elements (Create Records, Update Records, Delete Records) or SOQL elements (Get Records) inside a Loop element because each execution counts against governor limits: 150 DML statements and 100 SOQL queries per transaction. This is the Flow equivalent of the cardinal Apex sin: SOQL/DML inside a `for` loop. A loop of 200 records with a Get Records inside causes an immediate limit exception.
- When using a Get Records element, always decide whether to store "Only the first record," "All records," or "All records, up to a specified limit" (Spring '25+) because the choice determines whether the flow creates a single record variable or a record collection variable, and mixing them up causes type-mismatch errors downstream.
- When building a Decision element with multiple outcomes, always order outcomes from most specific to least specific because the flow evaluates outcomes top-to-bottom and takes the first match. Put your Default Outcome last as the catch-all. Same principle as ordering `if/else if` chains in Apex.
- Always use meaningful API names for every element and resource, following the pattern `[ObjectOrContext]_[Action]_[Detail]` (e.g., `Get_Active_Accounts`, `Decision_Check_Status`, `Assign_Priority_High`), because generic names like `myVariable_1` make debugging nearly impossible -- same as naming Apex variables `temp1` and `data2`.
- When creating formulas in flows, always handle null values explicitly with ISBLANK() or BLANKVALUE() because flow variables that have not been assigned are null, and null in a formula produces unexpected results or errors. Apex developers: this is the equivalent of always null-checking before `.method()` calls.
- Never directly modify a collection variable that you are actively looping through. Always create a separate target collection variable inside the loop, add modified records to it, and then use the target collection for DML after the loop. Same principle as not modifying a `List<>` while iterating over it in Apex.
- When calling Apex from a flow via an invocable action, always design the Apex method to accept and return `List<>` types because the Flow engine bulkifies calls -- multiple flow interviews may be batched into a single Apex invocation.
- When using a Subflow element, always verify that variables in the child flow are marked "Available for Input" and/or "Available for Output" because only explicitly exposed variables appear in the parent flow's Subflow element configuration.
- When you need to manipulate an entire collection (extract IDs, map fields between object types, apply formulas to every record), always use the Transform element instead of a Loop + Assignment pattern because Transform processes the entire collection in a single operation without explicit iteration.

## How It Works

Flow Builder elements fall into three categories: **Interaction** (Screen, Action, Subflow, Pause, Custom Error), **Logic** (Assignment, Decision, Loop, Collection Sort, Collection Filter, Transform, Custom Error), and **Data** (Get Records, Create Records, Update Records, Delete Records). Every element executes in sequence along the connectors you draw on the canvas.

**Resources** are the containers that hold data within a flow. They include Variables, Constants, Formulas, Text Templates, Choices, Record Choice Sets, and Stages. Resources are created from the Toolbox panel (left side of Flow Builder) and referenced by elements throughout the flow.

The flow transaction model is critical to understand: in record-triggered flows, the entire flow runs in a single transaction shared with Apex triggers and all other automation. In screen flows, each screen boundary acts as a transaction commit point. DML performed between screens is committed before the next screen renders. Governor limits (150 DML, 100 SOQL, 50,000 records retrieved, 10,000 records per DML, 10,000 CPU milliseconds per transaction) apply per-transaction, not per-flow -- and they are shared with any Apex triggers, workflow rules, and other flows firing in the same transaction.

## Flow Builder Navigation

### Creating Elements
Click **+** between two elements on the canvas (or after the Start element) → Select element category (Interaction / Logic / Data) → Select the specific element → Configure in the right panel

### Creating Resources
Click **Toolbox** (left panel, toggle icon) → **New Resource** → Select resource type → Configure name, data type, and default value

### Managing Elements
Click any element on the canvas → right panel shows configuration → **Edit** to modify, **Delete** to remove → Drag connectors to rearrange flow path. As of Summer '25, right-click context menus support cut, copy, delete, and fault path assignment directly on the canvas.

---

## Data Elements

### Get Records

Retrieves records from the database. Equivalent to a SOQL query -- `[SELECT ... FROM Object WHERE ...]`.

**Configuration Options:**

| Setting | Options | Notes |
|---------|---------|-------|
| Object | Any standard or custom object | Select the object to query |
| Filter Conditions | Field + Operator + Value | Combine with AND/OR logic. Operators: Equals, Not Equal To, Greater Than, Less Than, Greater Than or Equal, Less Than or Equal, Contains, Starts With, Ends With, Is Null, Is Blank (Spring '25+) |
| Sort Order | Field + Direction (Asc/Desc) | Only one sort field allowed per Get Records element |
| How Many Records | First record only / All records / All records up to a specified limit (Spring '25+) | First record → record variable. All records → record collection variable |
| Max Record Count (Spring '25+) | Number (up to 2,000) or Flow resource reference | Limits the number of records retrieved. Supports dynamic values via flow variables. Combine with Sort Order to get "top N" results without a separate Collection Filter |
| Store Fields | All fields / Choose fields | "Choose fields" is more performant; retrieves only what you need. Equivalent to selecting specific fields in SOQL vs `SELECT *` |
| Also Add Related Records (Winter '26 beta) | Select related child objects | Pulls parent + children in a single query. Equivalent to a SOQL subquery |

**Usage Pattern:**
```
Get Records → Filter: Status Equals "Active" → Sort: CreatedDate Desc → Limit: 10 → Store: All Records
Result: {!Get_Active_Accounts} (collection variable of up to 10 records)
```

**Key behavior:** When "Only the first record" is selected and no records match, the variable is set to null. Always add a Decision after a Get Records to check for null before referencing the record's fields -- same as checking `list.isEmpty()` in Apex before accessing `list[0]`.

**When to use Apex instead:** Use Apex (`@InvocableMethod` called from the flow, or direct trigger logic) when you need:
- **Dynamic SOQL**: Field names or object names determined at runtime (`Database.query()`)
- **Aggregate queries**: `GROUP BY`, `HAVING`, `COUNT()`, `SUM()`, `AVG()` -- Get Records cannot perform aggregations
- **SOSL search**: Full-text search across multiple objects
- **50,000+ rows**: Flows hit the same 50,000 SOQL row limit. Batch Apex can process millions.
- **Complex subqueries**: Beyond what "Also Add Related Records" supports
- **Polymorphic lookups**: Querying across `WhatId` / `WhoId` type fields with `TYPEOF`

### Create Records

Inserts one or more new records into the database. Equivalent to `insert` DML in Apex.

**Configuration Options:**

| Setting | Options | Notes |
|---------|---------|-------|
| How Many Records | One / Multiple (from collection) | One = set field values manually. Multiple = pass a record collection variable |
| Object | Any standard or custom object | Required when creating one record |
| Field Values | Field + Value | Map each field to a value (literal, variable, formula) |
| Store Record ID | Variable name | Automatically stores the new record's ID in a variable |

**Single record pattern:**
```
Create Records → Object: Case → Set: Subject = {!Screen_Subject}, ContactId = {!Screen_ContactId}
→ Store ID in: {!New_Case_Id}
```

**Collection pattern (bulk create):**
```
Assignment (inside loop): Add {!Loop_Variable} to {!Cases_To_Create}
→ (after loop) Create Records → Use collection: {!Cases_To_Create}
```

**Winter '26 enhancement:** After a Create Records element, you can immediately access fields from the newly inserted record without a separate Get Records. Previously you had to re-query to access auto-populated fields like `Name` (auto-number) or formula fields.

**When to use Apex instead:** Use Apex when you need partial-success DML (`Database.insert(records, false)` with `AllOrNone = false`), when you need to inspect individual `Database.SaveResult` entries, or when you need to insert records across multiple objects in a specific order with cross-references that exceed what flow variables can manage.

### Update Records

Modifies existing records in the database. Equivalent to `update` DML in Apex.

**Two modes:**

1. **Filter and update:** Select object → set filter criteria → set field values. Updates all matching records (like `UPDATE ... WHERE ...` in SQL). There is no LIMIT option on this mode -- always make your filter specific enough to avoid unintended mass updates.
2. **Use record variable:** Pass a record variable or record collection variable that already has the updated field values. The flow updates all records in the variable/collection.

**The "In" operator with text collections:** When using filter-and-update mode, you can use the "In" operator to match a field against a text collection variable. This is powerful in combination with the Transform element: extract a list of IDs via Transform, then use Update Records with `Id In {!Id_Collection}` to bulk-update all matching records in a single DML statement. Equivalent to `WHERE Id IN :idSet` in SOQL.

**When to use Apex instead:** Use Apex when you need `Database.update(records, false)` for partial success, when you need to update different fields on different records in the same collection (flow's filter-and-update sets the same field values on all matching records), or when the update logic requires complex conditional field assignment per record.

### Delete Records

Removes records from the database. Equivalent to `delete` DML in Apex.

**Configuration:** Select object → set filter conditions → the flow deletes all matching records.

**Key behavior:** Deleted records go to the Recycle Bin (recoverable for 15 days). There is no confirmation prompt in an autolaunched flow -- the records are just gone. In a screen flow, add a confirmation screen before the Delete element.

---

## Logic Elements

### Assignment

Sets or modifies variable values. The workhorse element for data manipulation -- equivalent to simple assignment statements in Apex.

**Operators by data type:**

| Data Type | Available Operators |
|-----------|-------------------|
| Text | Equals, Add (concatenate) |
| Number / Currency | Equals, Add, Subtract, Multiply, Divide |
| Boolean | Equals |
| Date / DateTime | Equals, Add (days), Subtract (days) |
| Record | Equals (assign entire record) |
| Collection | Add (append item), Add (append collection), Remove First, Remove After, Remove Before, Remove All, Remove Equals, Remove Position |

**Common patterns:**

```
// build a collection for bulk DML -- the #1 most important Flow pattern
Assignment: Add {!Loop_Current_Item} to {!Records_To_Update}

// set a flag variable
Assignment: {!Is_Approved} Equals {!$GlobalConstant.True}

// concatenate text
Assignment: {!Full_Name} Add {!First_Name}
Assignment: {!Full_Name} Add " "
Assignment: {!Full_Name} Add {!Last_Name}

// increment a counter
Assignment: {!Counter} Add 1
```

### Decision

Creates branching paths based on conditions. Equivalent to an `if/else if/else` chain in Apex.

**Configuration:**
- Each outcome has a label, an API name, and one or more conditions
- Conditions use operators: Equals, Not Equal To, Greater Than, Less Than, Contains, Starts With, Is Null, **Is Blank** (Spring '25+), **Is Empty** (Spring '25+ -- for collections), and more
- Condition logic: "All Conditions Are Met" (AND), "Any Condition Is Met" (OR), or custom logic (`1 AND (2 OR 3)`)
- Outcomes are evaluated top-to-bottom; the first matching outcome is taken
- The Default Outcome has no conditions and fires when no other outcome matches

**Spring '25 operators -- Is Blank and Is Empty:**
- **Is Blank**: Checks whether a text value contains no characters or only whitespace. For non-text data types, checks whether the value is null. Replaces the old pattern of `Equals` + `{!$GlobalConstant.EmptyString}` or checking `Is Null`.
- **Is Empty**: Checks whether a collection variable has zero elements. Replaces the old pattern of using an Assignment to count elements + a Decision to check the count.

**Example:**
```
Decision: Check Priority
├── Outcome "High Priority": Priority Equals "High" AND Amount Greater Than 100000
├── Outcome "Medium Priority": Priority Equals "Medium"
└── Default Outcome: "Low or Unset"
```

### Loop

Iterates over a record collection variable, processing one record at a time. Equivalent to a `for (SObject record : collection)` loop in Apex.

**Configuration:**
| Setting | Description |
|---------|-------------|
| Collection Variable | The record collection to iterate over |
| Loop Variable | Automatically created; holds the current record on each iteration |
| Direction | First to Last / Last to First |

**Bulkification pattern (the single most important flow pattern):**

```
1. Get Records → store in {!All_Accounts} (collection)
2. Loop → iterate over {!All_Accounts}
   └── Inside loop:
       a. Assignment: set {!Loop_Account}.Rating = "Hot"
       b. Assignment: Add {!Loop_Account} to {!Accounts_To_Update}
3. (After loop) Update Records → use {!Accounts_To_Update} collection
```

This pattern performs 1 SOQL query + 1 DML statement regardless of how many records are processed. Putting the Update inside the loop would use N DML statements. Apex developers: this is the exact same principle as bulkifying DML outside of `for` loops in trigger handlers.

**When to use Transform instead of Loop:** If your loop only extracts field values, maps fields between object types, or applies a formula to every record, the Transform element does the same work without explicit loop/assignment wiring. See the Transform section below.

### Collection Sort

Sorts a record collection variable by one or more fields. Equivalent to calling `list.sort()` with a custom `Comparable` implementation in Apex, except declarative.

**Configuration:** Select collection variable → add sort fields → choose Ascending or Descending for each field → nulls first or last.

**Best practice:** When possible, use the Sort Order on Get Records instead of a separate Collection Sort element -- it is more efficient because the sorting happens at the database level (SOQL `ORDER BY`) rather than in memory.

### Collection Filter

Filters a record collection variable down to records matching conditions. Equivalent to iterating a list and copying matching records to a new list in Apex -- but declarative and without explicit looping.

**Configuration:** Select collection variable → add filter conditions (same operators as Decision conditions, including Is Blank and Is Empty as of Spring '25) → the element creates a filtered copy of the collection. The original collection remains untouched.

**Key enhancement:** You can use formula expressions in filter conditions for complex evaluation criteria, not just simple field comparisons.

**When to use Apex instead:** Use Apex when filter criteria require complex calculations, cross-object lookups, or when the collection is very large and you need the efficiency of `Map` or `Set` operations for O(1) lookups rather than linear filtering.

### Transform

Processes an entire collection without explicit looping -- the modern replacement for the Loop + Assignment pattern for collection manipulation. Available as of Winter '24, with major enhancements in Spring '25 (Join) and subsequent releases.

**When to use Transform instead of Loop:**
- Extracting a list of field values from a record collection (e.g., all Account IDs from a list of Contacts) -- equivalent to `for (Contact c : contacts) { accountIds.add(c.AccountId); }` in Apex, but without writing the loop
- Converting a collection of one SObject type to another (e.g., mapping Contact fields to Lead fields) -- equivalent to a field-mapping loop in Apex
- Applying formula transformations to every record in a collection
- Merging two collections via Join (Spring '25+) -- equivalent to a Map-based join in Apex

**Configuration:**

| Setting | Description |
|---------|-------------|
| Source Collection | The input record collection to transform |
| Target Collection | The output collection (can be a different SObject type or a primitive type collection) |
| Field Mappings | Map source fields to target fields, with optional formula transformations or static values |
| Join (Spring '25+) | Merge two source collections by matching on a key field |

**Supported target types (Winter '25+):** Text, Number, Currency, Boolean, Date, DateTime. This means you can transform a record collection into a collection of primitives (e.g., extract all Email addresses into a Text collection).

**Join operation (Spring '25+):**
The Join capability merges data from two different collections into a single target collection based on a matching key field. This is the declarative equivalent of a Map-based join in Apex:

```apex
// what Transform Join replaces in Apex:
Map<Id, Account> accountMap = new Map<Id, Account>(accounts);
for (Contact c : contacts) {
    Account a = accountMap.get(c.AccountId);
    if (a != null) {
        // merge fields from both into a target record
    }
}
```

**Join configuration steps:**
1. Select both source collections
2. Choose the key fields to match on (e.g., Contact.AccountId matches Account.Id)
3. Map fields from both source collections into the target collection
4. Currently supports inner join (only records with matches in both collections)

**Formula and static value mappings (Spring '25+):**
Within a Transform, you can apply formulas or set static values on target fields, not just direct field-to-field copies. Example: map `Amount` to a target field with the formula `{!Source.Amount} * 1.1` to apply a 10% markup.

**The "In" operator pattern with Transform:**
Extract a list of IDs or text values via Transform into a text collection. Then use that text collection with the `In` operator on an Update Records or Delete Records element to target all matching records in a single DML statement. This eliminates the need for a Loop + Assignment + DML-after-loop pattern when you just need to update records matching a set of criteria.

```
1. Get Records → {!Contacts} (collection)
2. Transform → Source: {!Contacts}, Target: {!Account_Ids} (text collection), Mapping: AccountId
3. Update Records → Object: Account → Filter: Id In {!Account_Ids} → Set: Last_Contact_Activity__c = TODAY()
```

**What Transform cannot do:**
- Filter records -- use Collection Filter for that. Transform processes every record in the source collection.
- Conditional per-record logic -- if you need `if/else` per record, use a Loop with Decision elements inside.
- Complex multi-step transformations that require intermediate variables per record.

**When to use Apex instead of Transform:** Use Apex when you need complex conditional mapping logic per record, when you need to join on multiple keys or perform outer joins (Transform currently supports inner join only), or when the transformation involves callouts, complex calculations, or access to related records not in the source collection.

---

## Interaction Elements

### Screen

Displays a user interface. Only available in screen flows (not record-triggered flows). Screens support Lightning components, input fields, display text, data tables, and more.

### Action

Calls an invocable action: Apex `@InvocableMethod`, standard Salesforce actions (Send Email, Post to Chatter, Submit for Approval), REST API calls, or external service actions.

**Configuration:** Select action category → select specific action → map input values → map output values to flow variables.

**Send Email enhancements (Spring '25+):** The Send Email action now supports attaching files by providing a comma-separated list of Attachment IDs or ContentVersion IDs. Previously, sending emails with attachments required an `@InvocableMethod` in Apex. For simple email-with-attachment scenarios, the flow action now suffices. Use Apex when you need complex attachment generation (PDF via `getContentAsPDF()`), dynamic templates, or conditional attachment logic.

### Subflow

Calls another flow as a child flow. The parent flow passes input variables to the child and receives output variables back.

**Configuration:**
| Setting | Description |
|---------|-------------|
| Flow | Select the child flow by API name |
| Input Values | Map parent variables → child input variables |
| Output Values | Map child output variables → parent variables |

**When to use subflows:**
- When the same logic is needed in multiple flows (e.g., a standard error-logging routine)
- When a flow grows beyond 30-40 elements and becomes hard to read
- When different teams own different parts of the automation
- When you need to call an autolaunched flow from a screen flow to run DML in a separate context

**Key behavior:** Subflows share the parent flow's transaction and governor limits. They do not get a fresh set of limits. If the parent has used 100 DML statements, the subflow has 50 remaining. This is identical to how calling a helper method from an Apex trigger handler shares the same transaction limits.

**When to use Apex instead:** If the shared logic is computation-heavy (string parsing, complex math, data transformations), writing it as a shared Apex utility class called via `@InvocableMethod` is more performant and testable than a subflow.

### Pause

Pauses the flow interview until a defined resume event occurs. Only available in screen flows and autolaunched flows.

**Resume conditions:** A specific date/time is reached, or a platform event is received.

**Key behavior:** When a flow pauses, all pending DML is committed. When it resumes, a new transaction begins with fresh governor limits. Similar to breaking Apex work across a Queueable chain where each job gets fresh limits.

### Custom Error

Available in record-triggered flows (before-save and after-save). Displays a custom error message to the user and rolls back the entire transaction.

**Configuration:** Set the error message (supports merge fields). Optionally specify which field the error should appear on (field-level vs page-level).

**Key behavior:** Custom Error blocks the triggering DML from committing. This is the Flow equivalent of `addError()` on a record in an Apex trigger. The behavior is very similar: the save is rejected, and the user sees the error on the record page.

**Apex parallel:**
```apex
// Apex trigger equivalent of a Custom Error element:
trigger MyTrigger on Account (before insert) {
    for (Account acc : Trigger.new) {
        if (acc.Rating == null) {
            acc.addError('Rating is required for new accounts.');
            // or field-level: acc.Rating.addError('This field is required.');
        }
    }
}
```

---

## Flow Resources

### Variables

Containers that hold a single value or a single record during flow execution. Equivalent to variable declarations in Apex.

**Data types available:**
| Type | Description | Apex Equivalent |
|------|-------------|-----------------|
| Text | String value | `String` |
| Number | Decimal number | `Decimal` / `Integer` |
| Currency | Currency amount | `Decimal` |
| Boolean | True/False | `Boolean` |
| Date | Date without time | `Date` |
| DateTime | Date with time | `DateTime` |
| Picklist | Single picklist value | `String` |
| Multi-Select Picklist | Semicolon-separated values | `String` |
| Record | Single sObject record | `SObject` (e.g., `Account`) |
| Apex-Defined | Custom Apex class instance | Instance of a class with `@AuraEnabled` properties |

**Input/Output settings:**
- **Available for Input**: The variable can receive values from external sources (Lightning page, URL parameter, parent flow, Apex)
- **Available for Output**: The variable can pass values out when the flow finishes
- Mark both for bidirectional use (e.g., a record that is passed in, modified, and passed back)

**Security consideration:** Marking a variable "Available for Input" in a screen flow means any user can set that variable's value via URL parameters. Never trust input variables for security decisions (e.g., `?isAdmin=true`). Always validate input server-side. Apex developers: think of it like a public API endpoint with no authentication -- anyone can pass anything.

### Collection Variables

Hold multiple values of the same data type. Equivalent to `List<SObject>` or `List<String>` in Apex.

**Key operations:**
- **Add** (from Assignment): appends an item to the collection -- equivalent to `list.add(item)`
- **Add** (collection to collection): merges two collections -- equivalent to `list.addAll(otherList)`
- **Remove First / Remove All / Remove Position**: removes items -- equivalent to `list.remove(index)`
- **Loop**: iterates over the collection -- equivalent to `for (SObject s : list)`
- **Get Records (All Records)**: automatically stores results in a record collection variable
- **Transform**: processes the entire collection into a new collection without looping

### Constants

Fixed values that do not change during flow execution. Equivalent to `static final` constants in Apex.

**Example:** A constant `MAX_RETRY_COUNT` with value `3`, referenced in a Decision element. If the business rule changes, update the constant in one place instead of hunting through multiple elements.

### Formulas

Calculate values dynamically using formula syntax. Formulas re-evaluate whenever their referenced variables change.

**Key differences between Flow formulas and Apex:**

| Aspect | Flow Formulas | Apex Code |
|--------|--------------|-----------|
| Data source | Flow variables and resources | Any Apex variable, SOQL result, or method return |
| Null handling | Must use ISBLANK()/BLANKVALUE() -- null causes errors | Can use `== null` checks, ternary operator, or null-safe navigation `?.` |
| Cross-object references | Not supported (use Get Records first) | Direct SOQL relationship queries |
| String operations | LIMITED: CONTAINS, BEGINS, LEFT, RIGHT, SUBSTITUTE, LEN | Full String class: `contains()`, `substring()`, `split()`, regex via `Pattern` |
| Character limit | 3,900 characters | No practical limit |

**Common Flow Formula Syntax:**

```
// null handling -- always check for null before string operations
IF(ISBLANK({!Input_Name}), "Unknown", {!Input_Name})

// BLANKVALUE is shorthand for the above pattern
BLANKVALUE({!Input_Name}, "Unknown")

// text contains check (case-sensitive)
CONTAINS({!Account_Name}, "Corp")

// text starts-with check
BEGINS({!Case_Subject}, "URGENT")

// picklist comparison -- use TEXT() to convert picklist to string
TEXT({!Lead_Status}) = "Qualified"
// or use ISPICKVAL (preferred, does not require TEXT conversion)
ISPICKVAL({!Lead_Status}, "Qualified")

// date calculations
// days between two dates
{!End_Date} - {!Start_Date}
// add 30 days to today
TODAY() + 30
// current date and time
NOW()

// CASE statement (multi-value switch) -- equivalent to Apex switch statement
CASE({!Priority},
    "High", 1,
    "Medium", 2,
    "Low", 3,
    4
)
// last value (4) is the default/else

// IF with AND/OR logic
IF(
    AND(
        {!Amount} > 100000,
        NOT(ISBLANK({!Account_Id}))
    ),
    "High Value",
    "Standard"
)

// text manipulation
LEFT({!Full_Name}, 10)
RIGHT({!Phone_Number}, 4)
LEN({!Description})
TRIM({!User_Input})
UPPER({!Email_Domain})
LOWER({!First_Name})
SUBSTITUTE({!Template_Text}, "{NAME}", {!Contact_Name})

// number formatting
TEXT({!Amount})           // number to string
VALUE({!String_Number})   // string to number
ROUND({!Calculated_Price}, 2)
CEILING({!Division_Result})
FLOOR({!Division_Result})
```

**Functions NOT available in Flow formulas (use alternatives):**
- `ISNEW()` -- use `$Record__Prior` global variable in record-triggered flows
- `ISCHANGED()` -- compare `$Record.Field` to `$Record__Prior.Field` in a Decision element
- `PRIORVALUE()` -- use `$Record__Prior.FieldName`
- `VLOOKUP()` -- use a Get Records element
- `IMAGE()` -- use a Display Image screen component
- `HYPERLINK()` -- use an HTML-enabled Display Text component

**When to use Apex instead of Flow formulas:** When the calculation requires regex, complex string parsing (`split`, `substring` with dynamic indices), mathematical operations beyond basic arithmetic (trigonometry, statistical functions), or when the formula exceeds 3,900 characters.

### Text Templates

Store formatted text with merge fields. Useful for email bodies, display text, and confirmation messages.

**Supports:**
- Plain text or rich text (HTML)
- Merge fields referencing flow variables: `{!Variable_Name}`
- Merge fields referencing record fields: `{!Record_Variable.FieldName}`
- Global variables: `{!$User.FirstName}`, `{!$Organization.Name}`

**Example:**
```
Hi {!Contact_FirstName},

Your case #{!New_Case_Number} has been created.
Our team will respond within {!SLA_Hours} hours.

Thank you,
{!$Organization.Name}
```

### Choices and Record Choice Sets

**Choice:** A static option for screen components (Radio Buttons, Picklists, Checkboxes, Multi-Select Picklists).
- Configuration: Label, Value (stored value), and Data Type
- Use when options are fixed and known at design time

**Record Choice Set:** Dynamic options generated from a database query.
- Configuration: Object, filter conditions, choice label field, choice value field, sort order
- Use when options come from data (e.g., a list of active Products, a list of Queues)

**Picklist Choice Set:** Automatically generates choices from a picklist field's metadata.
- Configuration: Object and field
- Use when you want choices to stay in sync with picklist field definitions

### Stages

Represent user progress in a multi-screen flow. Display progress indicators (e.g., step 1 of 4).

**Configuration:**
- Create Stage resources (e.g., "Contact Info", "Case Details", "Review", "Confirmation")
- Use Assignment elements to set `{!$Flow.CurrentStage}` and add stages to `{!$Flow.ActiveStages}`
- The Stage progress indicator automatically renders at the top of screen flows
- Spring '25+: Built-in progress indicators with Simple (dots/lines) and Path (stage names visible) display modes

**Assignment syntax for stages:**
```
// set the current stage
Assignment: {!$Flow.CurrentStage} Equals {!Stage_Contact_Info}

// add a stage to the active stages list
Assignment: {!$Flow.ActiveStages} Add {!Stage_Contact_Info}
```

---

## Invocable Actions (Calling Apex from Flow)

This is the bridge between Flow and Apex. For comprehensive patterns including error handling, see [Invocable Methods](../apex-fundamentals/invocable-methods.md) and [Flow + Apex Coexistence](./flow-apex-coexistence.md).

### @InvocableMethod

Makes an Apex method callable as a Flow action.

**Annotation attributes:**

| Attribute | Required | Description |
|-----------|----------|-------------|
| `label` | No (recommended) | Display name in Flow Builder. Defaults to method name |
| `description` | No | Description shown in Flow Builder |
| `category` | No | Groups the action under a category. Default: "Uncategorized" |
| `callout` | No | Set to `true` if the method makes HTTP callouts. Default: `false` |

**Requirements:**
- The method must be `public` or `global` and `static`
- Only one `@InvocableMethod` per Apex class
- Input parameter must be `List<T>` (single parameter only)
- Return type must be `void` or `List<T>`
- The class must be an outer class (not an inner class)

**Basic example:**
```apex
public class CreateTaskAction {
    // label appears in Flow Builder's action picker
    @InvocableMethod(
        label='Create Follow-Up Task'
        description='Creates a follow-up task for the given record IDs'
        category='Task Management'
    )
    public static List<OutputWrapper> createTasks(List<InputWrapper> inputs) {
        // remember that the input is a List because Flow bulkifies calls --
        // multiple flow interviews can be boxcarred into one Apex call
        List<Task> tasksToInsert = new List<Task>();
        List<OutputWrapper> results = new List<OutputWrapper>();

        for (InputWrapper input : inputs) {
            Task t = new Task();
            t.Subject = input.subject;
            t.WhatId = input.recordId;
            t.ActivityDate = Date.today().addDays(input.daysUntilDue);
            t.OwnerId = input.assigneeId;
            tasksToInsert.add(t);
        }

        // single bulk DML outside the loop
        insert tasksToInsert;

        // build output list (must be same size as input list)
        for (Task t : tasksToInsert) {
            OutputWrapper result = new OutputWrapper();
            result.taskId = t.Id;
            result.success = true;
            results.add(result);
        }
        return results;
    }

    // input wrapper -- each @InvocableVariable becomes a flow input field
    public class InputWrapper {
        @InvocableVariable(label='Record ID' required=true)
        public Id recordId;

        @InvocableVariable(label='Task Subject' required=true)
        public String subject;

        @InvocableVariable(label='Days Until Due' required=false)
        public Integer daysUntilDue;

        @InvocableVariable(label='Assignee User ID' required=false)
        public Id assigneeId;
    }

    // output wrapper -- each @InvocableVariable becomes a flow output field
    public class OutputWrapper {
        @InvocableVariable(label='Created Task ID')
        public Id taskId;

        @InvocableVariable(label='Success')
        public Boolean success;
    }
}
```

### @InvocableVariable

Exposes class properties as input/output fields in Flow Builder.

**Annotation attributes:**

| Attribute | Required | Description |
|-----------|----------|-------------|
| `label` | No | Display name in Flow Builder |
| `description` | No | Description shown in Flow Builder |
| `required` | No | Whether the field is required. Default: `false` |

**Supported data types for @InvocableVariable:**
- Primitives: `String`, `Integer`, `Long`, `Decimal`, `Double`, `Boolean`, `Date`, `DateTime`, `Time`
- `Id`
- `SObject` (any standard or custom sObject)
- `List<T>` of any of the above (for collection inputs/outputs)
- `Blob` (for file content)

---

## When to Use Flow Elements vs Apex: Quick Reference

| Operation | Flow Element | When Apex is Better |
|-----------|-------------|-------------------|
| Simple field query | Get Records | Dynamic SOQL, aggregate queries, SOSL, 50k+ rows, polymorphic lookups |
| Record insert/update/delete | Create/Update/Delete Records | Partial-success DML, per-record conditional field setting, complex cross-object orchestration |
| Conditional branching | Decision | Complex nested conditions, regex matching, switch on enum types |
| Collection iteration | Loop + Assignment | When Transform cannot handle the logic; very complex per-record conditional transforms |
| Collection transformation | Transform | Multi-key joins, outer joins, conditional per-record logic with branching |
| Collection filtering | Collection Filter | Complex filter criteria requiring Map/Set lookups, regex, or cross-collection comparisons |
| Sending email | Send Email action (with attachments as of Spring '25) | PDF generation + attachment, complex dynamic templates, conditional attachments, merge field logic beyond basic merge |
| External callout | Action (external service) or async flow path | Certificate-based auth, retry/backoff, complex response parsing, chained callouts |
| Validation/error | Custom Error element | Complex validation requiring aggregate data, cross-object checks, or regex |
| Scheduling | Scheduled paths | Nightly batch processing, complex schedule logic, millions of records |

---

## Governor Limits Quick Reference

| Limit | Per-Transaction Value | Shared With |
|-------|----------------------|-------------|
| SOQL queries | 100 | Every Get Records element, every Apex trigger query, every workflow rule evaluation |
| DML statements | 150 | Every Create/Update/Delete element, every Apex DML statement |
| Records retrieved (SOQL) | 50,000 | Total rows across all Get Records + all Apex SOQL |
| Records per DML | 10,000 | Records passed to a single Create/Update/Delete element or Apex DML |
| CPU time | 10,000 ms | Complex formulas, loops, Apex actions, Transform operations |
| Callouts | 100 | Apex callouts, external service actions, async flow actions |
| Email invocations | 5,000 per day (org-wide) | Send Email actions + Apex `Messaging.sendEmail()` |
| Flow interviews per transaction | 2,000 | Subflows, record-triggered flows on related records |

**Critical pattern:** In a record-triggered flow processing a data load of 200 records, the flow runs once per batch of 200. Each Get Records/Create/Update/Delete inside a Loop multiplied by 200 records can easily exceed limits. Always use the bulkification pattern: Get before loop, assign inside loop, DML after loop. Apex developers: this is the same principle as bulkifying in trigger handlers -- the governor limits are literally the same pool.

## Common Mistakes

1. **Get Records returns null but the flow tries to access a field on it** -- The flow throws an unhandled fault: "element tried to access a null record." Fix: Always add a Decision element after Get Records checking `{!Record_Variable} Is Null = {!$GlobalConstant.True}` (or use the Is Blank operator as of Spring '25) and route the null path to skip or display an appropriate message. Apex parallel: always check `list.isEmpty()` before `list[0]`.

2. **Using "All fields" in Get Records when only 2-3 fields are needed** -- Retrieving all fields wastes SOQL row size and can contribute to heap size issues in complex flows. Fix: Select "Choose fields and let Salesforce do the rest" and only select the fields you actually reference downstream. Apex parallel: never write `SELECT *` -- always specify fields.

3. **Putting Get Records inside a Loop to look up related data** -- Each lookup inside a 100-iteration loop uses 100 of the 100 SOQL query limit. Fix: Get all related records before the loop using a single Get Records with appropriate filters, then use an Assignment or formula to match records during iteration. Or use the Transform Join to combine two collections by key. Apex parallel: exactly the same as putting SOQL inside a `for` loop.

4. **Building one massive flow instead of decomposing into subflows** -- Flows over 50+ elements become impossible to debug and maintain. Fix: Extract reusable logic (error logging, notification sending, validation routines) into separate autolaunched flows and call them via the Subflow element. Apex parallel: same as putting all logic in one 500-line method instead of extracting helper methods.

5. **Forgetting that Decision outcomes evaluate top-to-bottom** -- A broader condition placed above a more specific one catches records that should have gone to the specific outcome. Fix: Order outcomes from most restrictive to least restrictive. Place the Default Outcome last.

6. **Not bulkifying invocable Apex actions** -- Writing Apex that processes `inputs.get(0)` and ignores the rest of the list. When Flow boxcars multiple interviews, only the first record is processed. Fix: Always loop through the entire input list and return a result list of the same size. This is the most common mistake Apex developers make when writing Flow-callable code.

7. **Using CONTAINS() on multi-select picklists** -- CONTAINS returns true for partial matches (e.g., `CONTAINS("High;Highway", "High")` is true even when you only want the exact value "High"). Fix: Use INCLUDES() for multi-select picklist values, which checks for exact value matches.

8. **Referencing a collection variable where a single variable is expected (or vice versa)** -- Type mismatches between record variables and record collection variables cause save errors or runtime failures. Fix: Pay attention to the "How Many Records to Store" setting on Get Records and use the correct variable type downstream. If you stored "All records," you must loop or use collection operations to access individual records.

9. **Not setting default values on formula null paths** -- A formula like `{!Amount} * 0.1` fails if `{!Amount}` is null. Fix: Wrap in BLANKVALUE: `BLANKVALUE({!Amount}, 0) * 0.1`. Apex developers: this is the equivalent of `amount == null ? 0 : amount` before arithmetic.

10. **Creating records without mapping all required fields** -- The Create Records element does not validate required fields at design time. At runtime, the DML fails with a `REQUIRED_FIELD_MISSING` error. Fix: Check the object's required fields (both standard and custom) and map every required field in the Create Records element. Add a fault path to catch any missed fields.

11. **Using a Loop + Assignment to extract IDs when Transform would work** -- Manually looping through a collection to pull Account IDs into a text collection is 4+ elements (loop, assignment inside loop, assignment to add to collection). Transform does it in one element with one field mapping. Fix: Use Transform with a text collection target for ID extraction, field mapping, and type conversion. Reserve Loops for cases where you need per-record conditional logic.

12. **Not using the Get Records max count setting (Spring '25+)** -- Retrieving all records when you only need the top 5 sorted by CreatedDate. The flow fetches potentially thousands of records, consuming heap and SOQL row limits. Fix: Use the "All records, up to a specified limit" option with a max count of 5, combined with Sort Order. The limit value can even be a flow variable for dynamic control.

## See Also

- [Record-Triggered Flows](./record-triggered-flows.md)
- [Flow + Apex Coexistence Patterns](./flow-apex-coexistence.md)
- [Invocable Methods -- Flow-to-Apex Bridge](../apex-fundamentals/invocable-methods.md)
- [Apex Fundamentals (sharing, limits, triggers, testing)](../apex-fundamentals/apex-fundamentals.md)
