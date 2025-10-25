# PowerBI DAX UDF Library

## Overview

This toolkit provides a comprehensive collection of DAX (Data Analysis Expressions) User-Defined Functions (UDFs) specifically designed for Power BI reports and data models. UDFs allow you to package reusable, parameterized DAX logic into your models, making your DAX code more maintainable, consistent, and efficient.

## Why Use DAX UDFs?

1. **Code Reusability**: Define calculations once and reuse them across multiple measures, calculated columns, and visuals
2. **Maintainability**: Centralize business logic in one place, making updates and fixes more manageable
3. **Consistency**: Ensure uniform calculations across your entire data model
4. **Type Safety**: Benefit from optional type hints and type check helpers for more reliable code
5. **Version Control**: Track changes and manage different versions of your business logic
6. **Collaboration**: Share standardized calculations across team members and projects

## Prerequisites

- Power BI Desktop (latest version recommended)
- Enable DAX UDFs in Preview Features:
  1. Go to **File > Options and settings > Options**
  2. Select **Preview features**
  3. Check **DAX user-defined functions**
  4. Click **OK** and restart Power BI Desktop

## Function Categories

This library includes UDFs for various common business scenarios:

1. **Financial Functions**
   - Currency conversion

2. **Date/Time Functions**
   - Fiscal period calculations
   - Business day computations
   - Date range manipulations

3. **Statistical Functions**
   - Moving averages
   - Statistical distributions

4. **Text Manipulation**
   - String formatting
   - Text parsing
   - Pattern matching

5. **Business Logic**
   - Ranking functions
   - Custom aggregations

## Using UDFs in Power BI Data Models

### Basic Syntax

```dax
FUNCTION <FunctionName> = (
    <ParameterName>: <ParameterType>,
    ...
) => <FunctionBody>
```

### Example: Simple Tax Calculation

```dax
DEFINE
/// AddTax takes in amount and returns amount including tax
FUNCTION AddTax = (
    amount : NUMERIC
) =>
    amount * 1.1

// Usage in a measure
Total Sales with Tax = AddTax([Total Sales])
```

### Parameter Types

#### Data Types
- `Variant`: Any scalar value
- `Int64`: Whole numbers
- `Decimal`: Fixed-precision decimal
- `Double`: Floating-point decimal
- `String`: Text values
- `DateTime`: Date/time values
- `Boolean`: TRUE/FALSE values
- `Numeric`: Any numerical value

#### Parameter Modes
DAX UDFs support two parameter modes that control when and how parameters are evaluated:

- `VAL` (Eager Evaluation)
  - Default mode if not specified
  - Expression is evaluated once before invoking the function
  - Result is passed into the function
  - Best for simple scalar or table inputs
  - Example: `amount : NUMERIC VAL`

- `EXPR` (Lazy Evaluation)
  - Expression is evaluated inside the function
  - Can be evaluated multiple times in different contexts
  - Allows control over evaluation context (row/filter)
  - Required for reference parameters
  - Useful for context-sensitive calculations
  - Example: `table : TABLE EXPR`

Example showing the difference:
```dax
DEFINE
// VAL parameter - evaluated before function call
FUNCTION CountRowsNow = (
    t : TABLE VAL
) =>
    COUNTROWS(CALCULATETABLE(t, ALL('Date')))

// EXPR parameter - evaluated inside function
FUNCTION CountRowsLater = (
    t : TABLE EXPR
) =>
    COUNTROWS(CALCULATETABLE(t, ALL('Date')))

// Different results when called with filtered table
// CountRowsNow keeps external filter
// CountRowsLater removes date filter
```


## Scope and Model Requirements

Note: The UDFs in this library are implemented to be generally available and reusable across Power BI projects — they are not limited to a single Power BI data model. However, each UDF includes a documented "Model Requirements" section describing the exact tables, relationships, keys and data-quality expectations necessary for correct operation. You must ensure your data model satisfies the requirements listed for a given UDF (for example required date or currency tables, specific key columns and relationship cardinality); otherwise the function may return incorrect results or fail.

For details on the model required by a particular UDF, open the file for that function and review the `Model Requirements` section included with the function's documentation.


## Best Practices

1. **Documentation**
   - Always include function descriptions using `///` comments
   - Document parameter types and expected values
   - Provide usage examples

2. **Naming Conventions**
   - Use clear, descriptive function names
   - Follow consistent naming patterns
   - Consider using namespaces for organization

3. **Error Handling**
   - Include input validation
   - Provide meaningful error messages
   - Handle edge cases appropriately

4. **Performance**
   - Consider evaluation context
   - Use appropriate parameter modes (val/expr)
   - Optimize complex calculations

## Contributing

We welcome contributions to this DAX UDF library! Please follow these steps:

1. Fork the repository
2. Create a feature branch
3. Add your UDFs with proper documentation
4. Submit a pull request

## Resources

- [Official Microsoft DAX UDF Documentation](https://learn.microsoft.com/en-us/dax/best-practices/dax-user-defined-functions)
- [Power BI Documentation](https://learn.microsoft.com/en-us/power-bi/)
