# Stats.FYTDailySlope — Financial Year–to–Date Daily Trend (LINESTX)

**TL;DR**  
Returns a single **daily slope** (trend) of a measure over the **financial year** determined by your date table. Defaults to the **latest valid fact date**; remains filter-aware for non-date filters but evaluates across the **full financial year**.

---

## What it does
Computes the **slope per day** of an input measure over the target financial year using `LINESTX`.  
- Positive = upward trend; Negative = downward.  
- Uses the **latest available fact date** (excluding a configurable sentinel) to find that date’s **financial-year start**, then evaluates the measure for all dates in that FY and extracts `Slope1` from `LINESTX`.

---

## Result
A **scalar** numeric value = *Δ measure / day* for the selected FY (or latest FY when no FY filter is applied).

---

## Model Requirements
- **Date table** (e.g., `Date`)
  - Columns: `DateKey` *(Int, YYYYMMDD)*, `Date` *(Date)*, `FinYearStart` *(Date)*
  - One row per calendar date; `FinYearStart` maps every date to its FY start date.
- **Fact table** (e.g., `FactSales`)
  - Column `DateKey` relating to `Date[DateKey]`.
  - Your measure(s) to evaluate (e.g., `[Sales Amount]`).
- **Relationship**
  - `Fact[DateKey]` (many) → `Date[DateKey]` (one), single-direction is typical.
- **Data quality**
  - Consistent date keys/coverage; avoid placeholder dates (configurable below).

---

## Function Definition

```dax
DEFINE
/// Calculates the daily trend (slope) for the selected or latest full financial year.
///
/// @param _facttable : ANYREF - Reference to the fact table containing transactions.
/// @param _factdatekey : ANYREF - Column reference to the fact table date key (YYYYMMDD int).
/// @param _dimdatetable : ANYREF - Reference to the date dimension table.
/// @param _dimdatekey : ANYREF - Column reference to the date key in the date dimension (YYYYMMDD int).
/// @param _dimdate : ANYREF - Column reference to the date (date data type) in the date dimension.
/// @param _dimdatefirstdatefinyear : ANYREF - Column reference that indicates the first date of the financial year for each date.
/// @param _Measure : EXPR - The measure expression to evaluate (use EXPR for context sensitivity).
///
/// @returns Numeric slope representing daily trend for the financial year period.
FUNCTION _fn_LastYTDTrendLINESTX = (
        _facttable : ANYREF,
        _factdatekey : ANYREF,
        _dimdatetable : ANYREF,
        _dimdatekey : ANYREF,
        _dimdate : ANYREF,
        _dimdatefirstdatefinyear : ANYREF,
        _Measure : EXPR
) =>

        // Find the last valid date in the fact table (exclude placeholder date 99990101)
        VAR _lastDateNumber =
                MAXX(
                        FILTER(_facttable, _factdatekey <> 99990101),
                        _factdatekey
                )

        // Resolve the financial-year-first-date for the last date
        VAR _lastFinancialYearFirstDate =
                MAXX(
                        FILTER(_dimdatetable, _dimdatekey = _lastDateNumber),
                        _dimdatefirstdatefinyear
                )

        // Build a table with one row per date in the target financial year and the evaluated measure
        VAR _t =
                ADDCOLUMNS(
                        FILTER(
                                CALCULATETABLE(_dimdatetable, REMOVEFILTERS()),
                                _dimdatefirstdatefinyear = _lastFinancialYearFirstDate
                        ),
                        "@Measure", _Measure
                )

        // Compute LINESTX on the constructed table using the date column as the x variable
        VAR _linestx = LINESTX(_t, [@Measure], _dimdate)

        // Extract the slope (Slope1) from LINESTX result
        VAR _slope = MAXX(_linestx, [Slope1])

        RETURN
                _slope
```


### Implementation Notes

- The function uses `EXPR` for `_Measure` so that the measure is evaluated in the proper filter context
    when the function builds the per-day table.
- `REMOVEFILTERS()` on the date table is used to ensure the function calculates across the complete
    financial year determined by `_dimdatefirstdatefinyear`.
- LINESTX returns a table of coefficients; `Slope1` is used here to represent the slope for the first
    order term (daily slope).
- The function ignores a sentinel date value (99990101) often used as a placeholder; adjust if your
    model uses a different sentinel.


### Performance Considerations

- Use this UDF sparingly in visuals that iterate over many filter combinations — LINESTX and
    building per-day tables are non-trivial operations.
- Prefer evaluating this function in measures that are cached or used infrequently rather than inside
    high-cardinality row contexts.
- Pre-aggregate or restrict the date range if you only need a subset of the financial year for
    interactive scenarios.
- Ensure the date dimension and fact table have appropriate cardinalities and that the model
    relationships are optimized for single-directional filtering where possible.


