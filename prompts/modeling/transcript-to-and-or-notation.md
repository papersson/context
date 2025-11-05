# Transcript to AND/OR Notation

Transform domain expert interview transcripts into precise AND/OR notation that captures the domain model.

## Input

One or more interview transcripts where domain experts discuss:
- Business entities and their attributes
- Business processes and workflows
- Rules and constraints
- States and transitions
- Valid combinations and alternatives

## Output

Domain model documented in AND/OR notation - a lightweight formalism that makes the structure explicit before implementing types.

## AND/OR Notation Syntax

### Data Types

**AND - Required Combination:**
```
data Order =
  CustomerInfo
  AND ShippingAddress
  AND list of OrderLine
  AND AmountToBill
```
All components required. Order must have customer AND address AND lines AND amount.

**OR - Choice Between Alternatives:**
```
data PaymentMethod =
  CreditCard
  OR BankTransfer
  OR Cash
```
Exactly one alternative. Payment is credit card OR bank transfer OR cash, not multiple.

**Simple Types:**
```
data EmailAddress = string  // constrained
data OrderQuantity = integer between 1 and 1000
data CustomerType = New OR Established
```

**Lists:**
```
data Order = list of OrderLine  // one or more
data OptionalNotes = optional string  // zero or one
```

### Workflows

```
workflow "Place Order" =
  input: UnvalidatedOrder
  output: OrderPlaced OR ValidationError

workflow "Validate Order" =
  input: UnvalidatedOrder
  output: ValidatedOrder OR ValidationError
```

## Your Task

### 1. Extract Domain Concepts

From the transcript, identify:

**Nouns → Data Types**
- Entities: Order, Customer, Invoice, Contract
- Value objects: Address, Money, Date
- Enumerations: Status, Type, Category

**Verbs → Workflows**
- Actions: PlaceOrder, ValidateOrder, SendInvoice
- Processes: ClassifyContract, ApproveRequest
- Transformations: ConvertToX, CalculateY

**Adjectives → Constraints**
- "Required" → AND
- "Optional" → optional
- "Must be one of" → OR
- "Between X and Y" → constrained type

**Business Rules → Type Structure**
- "Must have X and Y" → AND
- "Either X or Y" → OR
- "Can't do X without Y" → structure that enforces this
- "X only valid when in state Y" → separate types per state

### 2. Identify Patterns

**Required Combinations:**
When you hear: "must have both X and Y", "needs X, Y, and Z"
```
data Thing =
  X
  AND Y
  AND Z
```

**Alternatives:**
When you hear: "either X or Y", "can be X, Y, or Z"
```
data Thing =
  X
  OR Y
  OR Z
```

**Optional Fields:**
When you hear: "might have X", "X is optional"
```
data Thing =
  RequiredField
  AND optional OptionalField
```

**Lists:**
When you hear: "one or more X", "multiple Xs", "list of X"
```
data Thing = list of X
```

**Constraints:**
When you hear: "must be between X and Y", "can't be negative", "starts with", "valid email"
```
data ConstrainedValue = integer between 1 and 100
data PositiveAmount = decimal greater than 0
data EmailAddress = string matching email format
```

**State-Dependent Types:**
When you hear: "when it's in state X, it has these fields; when in state Y, different fields"
```
data UnvalidatedOrder =
  RawCustomerInfo
  AND RawOrderLines

data ValidatedOrder =
  ValidatedCustomerInfo
  AND ValidatedOrderLines
  AND PricingInfo
```
Don't use a status enum. Create separate types.

**Workflows with Error Handling:**
When you hear: "this process can succeed or fail", "might return error"
```
workflow "Do Thing" =
  input: UnvalidatedThing
  output: ValidatedThing OR ValidationError
```

### 3. Structure Your Output

Organize the notation into sections:

**Simple Types** - Basic constrained values
```
data EmailAddress = string matching email format
data OrderQuantity = integer between 1 and 1000
data RecordStage = Imported OR Review OR Sign OR Completed
```

**Compound Types** - Combinations using AND
```
data CustomerInfo =
  Name
  AND EmailAddress
  AND optional PhoneNumber
```

**Choice Types** - Alternatives using OR
```
data PaymentMethod =
  CreditCard
  OR BankTransfer
  OR Cash

data RecordType =
  Customer
  OR Supplier
  OR Mutual
```

**State Types** - Different states of the same concept
```
data ImportedContract =
  RawDocument
  AND DefaultRecordType  // always "Imported"

data ValidatedContract =
  VerifiedDocument
  AND BusinessRecordType  // Customer, Supplier, etc.
  AND ValidatedProperties
```

**Workflows** - Function signatures
```
workflow "Classify Contract" =
  input: ImportedContract
  output: ValidatedContract OR ClassificationError

workflow "Verify Properties" =
  input: SuggestedProperties
  output: ValidatedProperties OR VerificationError
```

**Events** - Things that happened
```
event ContractClassified =
  ContractId
  AND RecordType
  AND Timestamp
  AND ClassifiedBy
```

### 4. Make Illegal States Unrepresentable

Look for business rules that prevent certain combinations:

**Bad (allows invalid states):**
```
data Contract =
  Stage  // can be Imported or Completed
  AND RecordType  // can be "Imported" or "Customer"
  // Problem: Can have Stage=Completed but RecordType="Imported" (invalid!)
```

**Good (invalid states impossible):**
```
data ImportedContract =
  // Always has Stage=Imported and RecordType="Imported"
  Document

data CompletedContract =
  // Stage=Completed and RecordType is business type
  Document
  AND BusinessRecordType  // Customer, Supplier, Mutual
```

If the transcript mentions "X can't happen when Y", model them as separate types.

### 5. Examples to Guide You

**Example 1 - From Contract Interview:**

Transcript excerpt:
> "Records in Imported stage are only visible to advanced users. The Record Type defaults to 'Imported' and must be changed to a business type like Customer or Supplier before verification. After verification, it moves to Completed stage."

Becomes:
```
// Simple types
data RecordStage = Imported OR Review OR Sign OR Completed
data BusinessRecordType = Customer OR Supplier OR Mutual
data UserLevel = Basic OR Advanced

// State types
data ImportedRecord =
  Document
  AND SuggestedProperties
  // Stage is implicitly "Imported"
  // RecordType is implicitly "Imported"
  // Only visible to Advanced users

data CompletedRecord =
  Document
  AND BusinessRecordType
  AND ValidatedProperties
  // Stage is implicitly "Completed"
  // Visible to all users

// Workflow
workflow "Verify Record" =
  input: ImportedRecord AND BusinessRecordType
  output: CompletedRecord OR VerificationError
```

**Example 2 - From E-commerce Interview:**

Transcript excerpt:
> "An order has customer info, shipping address, and one or more order lines. Each order line has a product, quantity, and price. Payment can be credit card, bank transfer, or cash."

Becomes:
```
// Simple types
data OrderQuantity = integer between 1 and 1000
data Price = decimal greater than 0

// Choice types
data PaymentMethod =
  CreditCard
  OR BankTransfer
  OR Cash

// Compound types
data OrderLine =
  Product
  AND OrderQuantity
  AND Price

data Order =
  CustomerInfo
  AND ShippingAddress
  AND list of OrderLine  // one or more
  AND PaymentMethod
  AND TotalAmount
```

## What NOT to Do

**Don't create flags or status enums if states have different data:**
```
// Bad - allows invalid combinations
data Contract =
  Stage  // enum
  AND RecordType  // string
  AND optional Properties  // might or might not exist
```

**Don't use generic types when business concepts exist:**
```
// Bad
data Thing = string

// Good
data RecordType = Customer OR Supplier OR Mutual
```

**Don't forget error cases in workflows:**
```
// Bad
workflow "Validate" =
  input: X
  output: Y

// Good
workflow "Validate" =
  input: X
  output: Y OR ValidationError
```

## Output Format

Present the AND/OR notation organized by sections:
1. Simple Types
2. Choice Types
3. Compound Types
4. State Types
5. Workflows
6. Events (if relevant)

Include comments explaining business rules where helpful.

## Example Complete Output

```
# Contract Classification Domain Model

## Simple Types
data EmailAddress = string matching email format
data RecordStage = Imported OR Review OR Sign OR Completed
data BusinessRecordType = Customer OR Supplier OR Mutual
data UserLevel = Basic OR Advanced

## State Types

// Imported contracts - waiting for classification
data ImportedContract =
  PDFDocument
  AND SuggestedProperties
  // Stage: Imported (implicit)
  // RecordType: "Imported" (implicit)
  // Visible to: Advanced users only

// Completed contracts - classified and verified
data CompletedContract =
  PDFDocument
  AND BusinessRecordType
  AND ValidatedProperties
  // Stage: Completed (implicit)
  // Visible to: All users

## Workflows

workflow "Smart Import" =
  input: PDFDocument
  output: ImportedContract
  // AI analyzes PDF and suggests properties

workflow "Verify Contract" =
  input: ImportedContract AND BusinessRecordType
  output: CompletedContract OR VerificationError
  // User classifies type and validates properties

workflow "Validate Property Suggestions" =
  input: SuggestedProperties
  output: ValidatedProperties OR ValidationError
  // User reviews and accepts/edits suggestions
```

## Tips

- Read the transcript multiple times - first pass for entities, second for relationships, third for rules
- Look for phrases like "must have", "either/or", "only when", "can't do X without Y"
- When experts mention exceptions, those often reveal separate types or OR choices
- If something "changes" or "transitions", consider separate types for each state
- Make the notation readable by a domain expert - they should verify it's correct
