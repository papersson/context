# AND/OR Notation to Type Definitions

Transform AND/OR notation into executable type definitions in a target programming language.

## Input

Domain model documented in AND/OR notation:
- Simple types
- Compound types (AND)
- Choice types (OR)
- State types
- Workflows
- Events

## Output

Type definitions in the target language that:
- Make illegal states unrepresentable
- Encode business rules in the type system
- Are readable and maintainable
- Guide correct implementation

## Supported Languages

Specify your target language. Examples use TypeScript, but you can request:
- TypeScript
- F#
- OCaml
- Haskell
- Rust
- Scala
- Kotlin
- Others (the prompt adapts)

**Default: TypeScript** (if not specified)

## Transformation Rules

### Simple Types → Type Aliases or Branded Types

**AND/OR Notation:**
```
data EmailAddress = string matching email format
data OrderQuantity = integer between 1 and 1000
```

**TypeScript:**
```typescript
// Branded type for compile-time safety
type EmailAddress = string & { readonly __brand: 'EmailAddress' };

// Smart constructor
function createEmailAddress(value: string): EmailAddress | Error {
  if (!isValidEmail(value)) {
    return new Error('Invalid email format');
  }
  return value as EmailAddress;
}

type OrderQuantity = number & { readonly __brand: 'OrderQuantity' };

function createOrderQuantity(value: number): OrderQuantity | Error {
  if (value < 1 || value > 1000) {
    return new Error('Quantity must be between 1 and 1000');
  }
  return value as OrderQuantity;
}
```

**F#:**
```fsharp
type EmailAddress = private EmailAddress of string

module EmailAddress =
    let create (value: string) : Result<EmailAddress, string> =
        if isValidEmail value then
            Ok (EmailAddress value)
        else
            Error "Invalid email format"

type OrderQuantity = private OrderQuantity of int

module OrderQuantity =
    let create (value: int) : Result<OrderQuantity, string> =
        if value >= 1 && value <= 1000 then
            Ok (OrderQuantity value)
        else
            Error "Quantity must be between 1 and 1000"
```

### Choice Types (OR) → Discriminated Unions

**AND/OR Notation:**
```
data PaymentMethod =
  CreditCard
  OR BankTransfer
  OR Cash
```

**TypeScript:**
```typescript
type PaymentMethod =
  | { type: 'CreditCard'; cardNumber: string; cvv: string }
  | { type: 'BankTransfer'; accountNumber: string; routingNumber: string }
  | { type: 'Cash'; amount: number };
```

**F#:**
```fsharp
type PaymentMethod =
    | CreditCard of cardNumber: string * cvv: string
    | BankTransfer of accountNumber: string * routingNumber: string
    | Cash of amount: decimal
```

### Compound Types (AND) → Records/Objects

**AND/OR Notation:**
```
data CustomerInfo =
  Name
  AND EmailAddress
  AND optional PhoneNumber
```

**TypeScript:**
```typescript
type CustomerInfo = {
  readonly name: string;
  readonly email: EmailAddress;
  readonly phoneNumber?: PhoneNumber;  // optional
};
```

**F#:**
```fsharp
type CustomerInfo = {
    Name: string
    Email: EmailAddress
    PhoneNumber: PhoneNumber option  // optional
}
```

### Lists → Array/List Types

**AND/OR Notation:**
```
data Order =
  CustomerInfo
  AND list of OrderLine
```

**TypeScript:**
```typescript
type Order = {
  readonly customerInfo: CustomerInfo;
  readonly orderLines: readonly [OrderLine, ...OrderLine[]];  // non-empty array
};
```

**F#:**
```fsharp
type NonEmptyList<'T> = NonEmptyList of 'T * 'T list

type Order = {
    CustomerInfo: CustomerInfo
    OrderLines: NonEmptyList<OrderLine>
}
```

### Workflows → Function Types

**AND/OR Notation:**
```
workflow "Validate Order" =
  input: UnvalidatedOrder
  output: ValidatedOrder OR ValidationError
```

**TypeScript:**
```typescript
type ValidationError = {
  readonly message: string;
  readonly field?: string;
};

type ValidateOrder = (
  order: UnvalidatedOrder
) => ValidatedOrder | ValidationError;
```

**F#:**
```fsharp
type ValidationError = {
    Message: string
    Field: string option
}

type ValidateOrder = UnvalidatedOrder -> Result<ValidatedOrder, ValidationError>
```

### State Types → Separate Types (Not Status Enums!)

**AND/OR Notation:**
```
data ImportedContract =
  Document
  AND SuggestedProperties

data CompletedContract =
  Document
  AND BusinessRecordType
  AND ValidatedProperties
```

**TypeScript:**
```typescript
// Don't do this:
// type Contract = {
//   stage: 'Imported' | 'Completed';
//   recordType?: string;  // might not exist
//   properties?: Properties;  // might not exist
// };

// Do this:
type ImportedContract = {
  readonly document: Document;
  readonly suggestedProperties: SuggestedProperties;
  // stage is implicitly "Imported"
};

type CompletedContract = {
  readonly document: Document;
  readonly recordType: BusinessRecordType;
  readonly validatedProperties: ValidatedProperties;
  // stage is implicitly "Completed"
};

type Contract = ImportedContract | CompletedContract;
```

**F#:**
```fsharp
type ImportedContract = {
    Document: Document
    SuggestedProperties: SuggestedProperties
}

type CompletedContract = {
    Document: Document
    RecordType: BusinessRecordType
    ValidatedProperties: ValidatedProperties
}

type Contract =
    | Imported of ImportedContract
    | Completed of CompletedContract
```

## Design Principles to Apply

### 1. Make Illegal States Unrepresentable

Use the type system to prevent invalid combinations:

**Bad:**
```typescript
type Contract = {
  stage: 'Imported' | 'Completed';
  recordType: string;  // Can be wrong for the stage!
};
```

**Good:**
```typescript
type ImportedContract = { /* ... */ };
type CompletedContract = { /* ... */ };
type Contract = ImportedContract | CompletedContract;
```

### 2. Parse, Don't Validate

Create validated types at boundaries, trust them internally:

```typescript
// At boundary
function createEmailAddress(raw: string): EmailAddress | Error {
  // Validation here
}

// Inside domain
function sendEmail(to: EmailAddress, message: string): void {
  // No validation needed - type guarantees it's valid
}
```

### 3. Total Functions

Handle all cases explicitly:

```typescript
function processPayment(method: PaymentMethod): PaymentResult {
  switch (method.type) {
    case 'CreditCard':
      return processCreditCard(method);
    case 'BankTransfer':
      return processBankTransfer(method);
    case 'Cash':
      return processCash(method);
    // Compiler ensures we handle all cases
  }
}
```

### 4. Use Result Types for Errors

Don't throw exceptions in domain logic:

**Bad:**
```typescript
function validateOrder(order: Order): ValidatedOrder {
  if (!isValid(order)) {
    throw new Error('Invalid order');  // Hidden side effect
  }
  return order as ValidatedOrder;
}
```

**Good:**
```typescript
function validateOrder(order: Order): ValidatedOrder | ValidationError {
  if (!isValid(order)) {
    return { message: 'Invalid order', field: 'order' };
  }
  return order as ValidatedOrder;
}
```

## Language-Specific Patterns

### TypeScript

**Readonly by default:**
```typescript
type CustomerInfo = {
  readonly name: string;
  readonly email: EmailAddress;
};
```

**Branded types for primitives:**
```typescript
type EmailAddress = string & { readonly __brand: 'EmailAddress' };
```

**Non-empty arrays:**
```typescript
type NonEmptyArray<T> = readonly [T, ...T[]];
```

**Result type:**
```typescript
type Result<T, E> = T | E;
// Or use a library like ts-results
```

### F#

**Private constructors:**
```fsharp
type EmailAddress = private EmailAddress of string
```

**Result type (built-in):**
```fsharp
Result<'T, 'Error>
```

**Option type (built-in):**
```fsharp
'T option  // Some value or None
```

**Records (immutable by default):**
```fsharp
type CustomerInfo = {
    Name: string
    Email: EmailAddress
}
```

## Example Complete Transformation

**Input (AND/OR Notation):**
```
data BusinessRecordType = Customer OR Supplier OR Mutual

data ImportedContract =
  PDFDocument
  AND SuggestedProperties

data CompletedContract =
  PDFDocument
  AND BusinessRecordType
  AND ValidatedProperties

workflow "Verify Contract" =
  input: ImportedContract AND BusinessRecordType
  output: CompletedContract OR VerificationError
```

**Output (TypeScript):**
```typescript
// Simple types
type BusinessRecordType = 'Customer' | 'Supplier' | 'Mutual';

// State types
type ImportedContract = {
  readonly document: PDFDocument;
  readonly suggestedProperties: SuggestedProperties;
};

type CompletedContract = {
  readonly document: PDFDocument;
  readonly recordType: BusinessRecordType;
  readonly validatedProperties: ValidatedProperties;
};

// Union type
type Contract = ImportedContract | CompletedContract;

// Error type
type VerificationError = {
  readonly message: string;
  readonly issues: readonly string[];
};

// Workflow
type VerifyContract = (
  contract: ImportedContract,
  recordType: BusinessRecordType
) => CompletedContract | VerificationError;
```

**Output (F#):**
```fsharp
// Simple types
type BusinessRecordType =
    | Customer
    | Supplier
    | Mutual

// State types
type ImportedContract = {
    Document: PDFDocument
    SuggestedProperties: SuggestedProperties
}

type CompletedContract = {
    Document: PDFDocument
    RecordType: BusinessRecordType
    ValidatedProperties: ValidatedProperties
}

// Union type
type Contract =
    | Imported of ImportedContract
    | Completed of CompletedContract

// Error type
type VerificationError = {
    Message: string
    Issues: string list
}

// Workflow
type VerifyContract =
    ImportedContract -> BusinessRecordType -> Result<CompletedContract, VerificationError>
```

## Instructions

1. **Read the entire AND/OR notation** to understand the domain
2. **Choose the target language** (or ask if not specified)
3. **Transform each section**:
   - Simple types → branded types or private constructors
   - Choice types → discriminated unions
   - Compound types → records/objects
   - State types → separate types (not status fields!)
   - Workflows → function signatures with Result types
   - Events → readonly records with timestamp
4. **Add smart constructors** for constrained types
5. **Document any business rules** in comments
6. **Ensure illegal states are unrepresentable**

## Output Format

Organize by sections matching the input:
1. Simple Types
2. Choice Types
3. Compound Types
4. State Types
5. Workflows
6. Events (if any)

Include comments explaining design choices where helpful.

## Tips

- Favor immutability (readonly, const, etc.)
- Use discriminated unions over boolean flags or status enums
- Encode constraints in types, not runtime checks (when possible)
- Make error cases explicit in function signatures
- Keep types readable - domain experts should recognize the concepts
- Use language idioms (branded types in TS, private constructors in F#, etc.)
