# PowerBI DAX UDF Library

Namespaced, reusable DAX User-Defined Functions (UDFs) for Power BI. Import functions, call them like built-ins, and keep business logic consistent, documented, and versioned.

## What is a DAX UDF?

A DAX UDF is a reusable function you define in DAX (preview) and call from measures/expressions. Benefits:
- Consistency
- Maintainability
- Context control

### Signature Pattern

```dax
DEFINE 
FUNCTION <Namespace.FunctionName> = (
    param1 : <TYPE> [VAL|EXPR],
    ...
) => <FunctionBody>
```

### Types
- `NUMERIC`
- `STRING`
- `BOOLEAN`
- `DATETIME`
- `TABLE`
- `VARIANT`
- `INT64`
- `DECIMAL`
- `DOUBLE`

### Modes

- `VAL` (default): evaluate before call (eager)
- `EXPR`: evaluate inside function (lazy, context-aware)

## Prerequisites

1. Power BI Desktop (latest)
2. Enable DAX UDF:
   - File → Options → Preview features
   - Enable "DAX user-defined functions"
   - Restart Power BI Desktop


## Importing Functions

### Option A: DAX Query View (Power BI Desktop)

1. Open DAX Query View
2. Paste the `DEFINE FUNCTION` block(s) from `/src/.../*.dax`
3. Save to model

### Option B: 


## Usage

### Example Function and Measure

```dax
DEFINE
/// Financial.AddTax(amount: NUMERIC VAL) -> NUMERIC
/// Returns amount with 10% tax.
/// Model Requirements: none
FUNCTION Financial.AddTax = (
    amount : NUMERIC
) => amount * 1.10

-- In your model
Total Sales w/ Tax = Financial.AddTax ( [Total Sales] )
```

### EXPR vs VAL (context control)

```dax
DEFINE
FUNCTION Stats.CountAllDates_VAL = ( t: TABLE VAL ) =>
    COUNTROWS( CALCULATETABLE( t, ALL('Date') ) )

FUNCTION Stats.CountAllDates_EXPR = ( t: TABLE EXPR ) =>
    COUNTROWS( CALCULATETABLE( t, ALL('Date') ) )
```

**Note**: With an external 'Date' filter:
- `VAL` preserves the filter at call time
- `EXPR` re-evaluates inside the function and can modify context (e.g., `ALL('Date')`)

## Model Requirements

Each `.dax` file begins with `///` docs including Model Requirements:
- Required tables
- Keys
- Relationships
- Cardinality
- Data quality

⚠️ **Important**: Ensure your model matches these requirements, or outputs may be incorrect.

## Conventions

### Documentation (mandatory in each file)

```dax
/// <Namespace.FunctionName>
/// Summary: <what it does>
/// Params:
///   param (TYPE [VAL|EXPR]): <meaning>
/// Returns: <TYPE>
/// Model Requirements: <tables/columns/relationships>
/// Example:
///   <one-line measure usage>
/// Notes: <edge cases/perf>
```

### Naming

Format: `<Namespace>.<VerbNoun>` 
Examples:
- `Financial.AddTax`
- `DateTime.FiscalYearIndex`

### Performance Best Practices

1. Be explicit with context:
   - `KEEPFILTERS`
   - `REMOVEFILTERS`
   - `ALL`
   - `ALLEXCEPT`

2. Prefer set-based patterns
   - Avoid repeated heavy calculations
   - Wrap once in a helper UDF

3. Parameter modes:
   - Use `EXPR` for context-sensitive parameters (tables/measures)
   - Use `VAL` for scalars

## Contributing

### Workflow

1. Fork → create feature branch
2. Add function under the correct namespace folder in `/src`
3. Include complete `///` header:
   - Summary
   - Params with type & mode
   - Returns
   - Model Requirements
   - Example
   - Notes
4. Validate in a sample model (e.g., Contoso/AdventureWorks)
5. Open a PR with:
   - Purpose
   - Usage
   - Trade-offs
   - Performance notes (timings/screens if helpful)

### PR Checklist

- [ ] File name = full function name; correct namespace folder
- [ ] `///` header complete (incl. Model Requirements)
- [ ] Compiles and runs in a basic model
- [ ] No breaking changes (or clearly documented)

### Folder Additions

When adding a new namespace:
- Include a short `README.md` inside the folder
- List all functions
- Document typical model prerequisites