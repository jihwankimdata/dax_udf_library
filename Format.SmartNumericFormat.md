# Format.SmartNumericFormat — Dynamic Number Format String (Auto-Scale, % and Fixed)

**TL;DR**  
Returns a **format string** for the current measure that can **auto-scale (K/M/B/T)**, switch to **percentage**, or use **fixed decimals**, controlled by a single-value slicer. Designed for wide-ranging values and standardized presentation across visuals.

---

## What it does
- **Auto-scales** based on magnitude with suffixes **K / M / B / T**  
- **Adapts precision** (0–2 decimals) by range  
- Supports **percentage** formatting  
- Allows **fixed** formats (0, 1, 2 decimals, or %)  
- Centralizes formatting logic for **consistent** UX

**Result:** A **STRING** format pattern to be used as a **Format String Expression** for measures (context-aware via `SELECTEDMEASURE()`).

---

## PBI Data Model Requirements

### Required table
- `__apply_dynamic_format_string`
  - Column: `[Dynamic Format String]` with values **"Yes"** or **"No"**
  - Must be **single-select** (one value in filter context)

### Measure requirements
- The target measure must return **numeric** values and handle **BLANK()** appropriately.
- This UDF uses `SELECTEDMEASURE()`; it’s intended for **Format String Expressions** (or measures that assemble such strings for display).

### Parameter validation
- `_dynamicFormat` (INT): **1=auto-scale**, **2=percentage**
- `_fixedFormat` (INT): **3=whole**, **4=1 decimal**, **5=2 decimals**, **2=percentage**

---

## Parameters & Return

**Parameters**
- `_dynamicFormat : INT64 VAL` — dynamic mode type (1=auto-scale, 2=%)
- `_fixedFormat : INT64 VAL` — fixed mode type (3=whole, 4=1dp, 5=2dp, 2=%)

**Return**
- **STRING** — DAX format string pattern (e.g., `"#,#0,.0K"` or `"#,#0.00%"`)

---

## Function Definition

```dax
DEFINE
/// Format.SmartNumericFormat
/// Returns a dynamic or fixed number format string for the current measure.
/// - Dynamic: auto-scale with K/M/B/T and adaptive decimals, or percentage
/// - Fixed: whole, 1dp, 2dp, or percentage
///
/// Params:
///   _dynamicFormat (INT64 VAL): 1=auto-scale, 2=percentage
///   _fixedFormat   (INT64 VAL): 3=whole, 4=1 decimal, 5=2 decimals, 2=percentage
///
/// Return: STRING (format string)
FUNCTION Format.SmartNumericFormat = (
    _dynamicFormat : INT64,
    _fixedFormat   : INT64
) =>

VAR _applyDynamic =
    SELECTEDVALUE ( __apply_dynamic_format_string[Dynamic Format String], "Yes" )

VAR _absValue =
    ABS ( SELECTEDMEASURE () )

-- Dynamic format options (magnitude → pattern)
VAR _autoScalePattern =
    SWITCH(
      TRUE(),
      _measure >= 100000000000000, "#,#0,,,,.T",      // Trillions
      _measure >= 10000000000000,  "#,#0,,,,.0T",     // Tenths of trillions
      _measure >= 1000000000000,   "#,#0,,,,.00T",    // Hundredths of trillions
      _measure >= 100000000000,    "#,#0,,,.B",       // Billions
      _measure >= 10000000000,     "#,#0,,,.0B",      // Tenths of billions
      _measure >= 1000000000,      "#,#0,,,.00B",     // Hundredths of billions
      _measure >= 100000000,       "#,#0,,.M",        // Millions
      _measure >= 10000000,        "#,#0,,.0M",       // Tenths of millions
      _measure >= 1000000,         "#,#0,,.00M",      // Hundredths of millions
      _measure >= 100000,          "#,#0,.K",         // Thousands
      _measure >= 10000,           "#,#0,.0K",        // Tenths of thousands
      _measure >= 1000,            "#,#0,.00K",       // Hundredths of thousands
      _measure >= 0.01,            "#0,.00",          // Standard 2 decimals
      _measure >= 0.001,           "#0,.000"          // Small values 3 decimals
   )

-- Fixed format options
VAR _fmtWhole = "#,#0"
VAR _fmt1dp   = "#,#0.0"
VAR _fmt2dp   = "#,#0.00"
VAR _fmtPct   = "#,#0.00%"

RETURN
    SWITCH (
        TRUE (),

        -- Dynamic mode
        _applyDynamic = "Yes" && _dynamicFormat = 1, _autoScalePattern,
        _applyDynamic = "Yes" && _dynamicFormat = 2, _fmtPct,

        -- Fixed mode
        _applyDynamic = "No"  && _fixedFormat = 3, _fmtWhole,
        _applyDynamic = "No"  && _fixedFormat = 4, _fmt1dp,
        _applyDynamic = "No"  && _fixedFormat = 5, _fmt2dp,
        _applyDynamic = "No"  && _fixedFormat = 2, _fmtPct,

        -- Fallback
        _fmt2dp
    )
```


### Implementation Notes

The function implements several key strategies for flexible number formatting:

1. **Dynamic Scaling Logic**
   - Evaluates measure magnitude using power-of-three thresholds
   - Applies appropriate suffix (K, M, B, T) based on value size
   - Adjusts decimal precision inversely to magnitude
   - Handles values from trillions down to thousandths

2. **Format Pattern Structure**
   - Uses comma for thousand separators (#,#)
   - Employs scaling through comma elimination (,,,)
   - Maintains consistent decimal precision per range
   - Applies suffixes for readability (K, M, B, T)

3. **Mode Selection**
   - Dynamic mode for auto-scaling presentation
   - Fixed mode for consistent formatting
   - Parameter-driven format selection
   - Default fallback to two decimals

4. **Error Prevention**
   - Uses ABS() to handle negative values
   - Provides default format for unhandled cases
   - Validates parameter combinations
   - Maintains data type consistency

### Performance Considerations

1. **Execution Efficiency**
   - Minimal calculation overhead
   - Single SELECTEDMEASURE() call
   - Efficient SWITCH statement evaluation
   - No table scanning or iteration

2. **Memory Impact**
   - String pattern storage is negligible
   - No temporary table creation
   - Static format pattern definitions
   - Constant memory footprint

3. **Optimization Tips**
   - Cache formatted columns where possible
   - Use appropriate parameter values
   - Consider measure magnitude ranges
   - Group similar formats together

4. **Usage Guidelines**
   - Apply to individual measures
   - Avoid excessive dynamic switching
   - Match format to data characteristics
   - Consider user experience needs


