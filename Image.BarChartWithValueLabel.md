# Image.BarChartWithValueLabel — Dynamic SVG Bar Chart with Value Labels

**Overview**  
Generates a **responsive SVG bar chart** with automatic value and percentage labels. Designed for Power BI visuals requiring compact data visualization with detailed value information.
Returns an **Image URL string** compatible with *Data Category → Image URL*.

---

## What it does
- Creates a **horizontal bar chart** with dynamic scaling
- Shows both **absolute values** and **percentages** as labels
- Automatically **sorts** data in descending order
- Provides **configurable styling** (colors, fonts, sizes)
- Handles **empty states** gracefully
- Maintains **consistent spacing** and layout

---

## Result
**Type:** `STRING`  
**Output Format:** `data:image/svg+xml;utf8,...`  
**Visual Properties:**
- 600×800 viewbox (configurable)
- Responsive scaling
- White background
- Category labels on left
- Value and percentage labels above bars
- Automatically sorted bars (descending)

---

## PBI Data Model Requirements

### Required Parameters
- `_measure`: Any numeric measure (EXPR mode for context awareness)
- `_categoryColumn`: Column reference for categories

### Visual Configuration
- Output measure must have **Data Category** set to **Image URL**
- Visual must support image URL rendering:
  - Cards
  - Images
  - Tables
  - Matrices

---

## Function Definition
```dax
DEFINE
/// Image.BarChartWithValueLabel
/// Summary: Creates a horizontal bar chart showing values and percentages for categories.
/// Params:
///   _measure (EXPR): Numeric measure to visualize
///   _categoryColumn (ANYREF): Column containing category names
/// Returns: STRING - SVG image URL for Power BI visuals
/// Model Requirements: None specific, works with any measure and category column
/// Example:
///   Sales by Category Chart = Image.BarChartWithValueLabel([Total Sales], 'Product'[Category])
/// Notes:
///   - Sorts categories by measure value (descending)
///   - Shows both absolute value and percentage
///   - Configurable through layout variables
///   - Handles empty states with placeholder message
FUNCTION Image.BarChartWithValueLabel = (
    _measure : EXPR,
    _categoryColumn : ANYREF
) =>

/* ===== Layout Configuration ===== */
VAR _w  = 600        -- Canvas width (narrower improves text size in cards)
VAR _h  = 800        -- Canvas height for vertical space
VAR _mT = 36         -- Top margin
VAR _mR = 36         -- Right margin
VAR _mB = 36         -- Bottom margin
VAR _mL = 36         -- Left margin
VAR _cw = _w - _mL - _mR    -- Content width
VAR _ch = _h - _mT - _mB    -- Content height

/* ===== Visual Styling ===== */
VAR _barColor   = "Green"    -- Bar fill color
VAR _catColor   = "Black"    -- Category text color
VAR _labelColor = "Blue"     -- Value label color
VAR _catFont    = 16         -- Category text size
VAR _labFont    = 16         -- Label text size
VAR _barH       = 18         -- Bar height
VAR _rowGap     = 33         -- Vertical gap between rows
VAR _scaleFactor = 0.5       -- Bar length scaling factor

/* ===== Data Processing ===== */
-- Calculate total across all categories (ignoring category filters)
VAR _TotalAllCategories =
    CALCULATE ( _measure, ALL ( _categoryColumn ) )

-- Get filtered categories (respecting other filters)
VAR _Categories =
    VALUES ( _categoryColumn )

-- Add measure values and percentages
VAR _DataWithMetrics =
    ADDCOLUMNS (
        _Categories,
        "__Val", _measure,
        "__Pct", DIVIDE( _measure, _TotalAllCategories )
    )

-- Remove blanks and sort by value
VAR _SortedData =
    TOPN ( 
        1000, 
        FILTER ( _DataWithMetrics, NOT ISBLANK([__Val]) ),
        [__Val],
        DESC 
    )

-- Count rows for empty state check
VAR _RowCount = COUNTROWS ( _SortedData )

-- Calculate value range for scaling
VAR _MinValue = 0
VAR _MaxValue = MAXX ( _SortedData, [__Val] )
VAR _ValueRange = MAX ( 1, _MaxValue - _MinValue )

/* ===== Row Generation ===== */
-- Calculate row height including gap
VAR _RowHeight = _barH + _rowGap

-- Generate SVG elements for each row
VAR _RowsSvg =
    VAR _WithIndex =
        ADDCOLUMNS ( 
            _SortedData, 
            "__Idx", 
            RANKX ( _SortedData, [__Val], , DESC, DENSE ) - 1 
        )
    RETURN
        CONCATENATEX (
            _WithIndex,
            VAR _idx   = [__Idx]
            VAR _name  = _categoryColumn
            VAR _val   = [__Val]
            VAR _pct   = [__Pct]
            VAR _yTop  = _mT + _idx * _RowHeight
            VAR _x0    = _mL
            VAR _x1    = _mL + DIVIDE(
                (_val - _MinValue) * _cw * _scaleFactor,
                _ValueRange,
                0
            )
            VAR _barWidth = MAX ( 0.5, _x1 - _x0 )
            
            -- Combine SVG elements
            RETURN
                -- Category label
                "<text x='" & (_mL - 8) & "' y='" & (_yTop + _barH) &
                "' text-anchor='end' font-family='Segoe UI' font-size='" & 
                _catFont & "' fill='" & _catColor & "'>" &
                SUBSTITUTE ( _name, "&", "&amp;" ) & "</text>" &

                -- Bar element
                "<rect x='" & _x0 & "' y='" & _yTop & 
                "' width='" & _barWidth & "' height='" & _barH & 
                "' rx='2' ry='2' fill='" & _barColor & "'/>" &

                -- Value and percentage label
                "<text x='" & _x0 & "' y='" & (_yTop - 4) &
                "' text-anchor='start' font-family='Segoe UI' font-size='" & 
                _labFont & "' fill='" & _labelColor & "'>" &
                FORMAT ( _val, "#,0" ) & " | " & 
                FORMAT ( _pct, "0.0%" ) & "</text>",
            ""
        )

/* ===== Final SVG Assembly ===== */
VAR _CompleteSvg =
    "data:image/svg+xml;utf8," &
    "<svg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 " & _w & " " & _h & "'>" &
    "<rect width='100%' height='100%' fill='white'/>" &
    _RowsSvg &
    "</svg>"

-- Return empty state message if no data
RETURN
    IF ( 
        _RowCount = 0,
        "data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 600 80'>" &
        "<text x='300' y='45' text-anchor='middle' font-family='Segoe UI' " &
        "font-size='14' fill='#777'>No data</text></svg>",
        _CompleteSvg
    )
```

### Implementation Notes

1. **Layout System**
   - Uses configurable margins for flexible spacing
   - Implements responsive scaling for bar lengths
   - Maintains consistent row heights and gaps
   - Supports customizable canvas dimensions

2. **Data Processing**
   - Automatically sorts categories by value
   - Calculates percentages of total
   - Filters out blank/empty values
   - Supports up to 1000 categories

3. **Visual Elements**
   - Left-aligned category labels
   - Horizontal bars with rounded corners
   - Value and percentage labels above bars
   - White background for clarity

4. **Error Handling**
   - Handles empty datasets gracefully
   - Escapes special characters in labels
   - Ensures minimum bar width visibility
   - Prevents division by zero in scaling

### Performance Considerations

1. **Data Efficiency**
   - Uses EXPR mode for proper context handling
   - Minimizes measure evaluations
   - Processes data in sets, not rows
   - Caches intermediate calculations in variables

2. **Memory Management**
   - Limits to 1000 categories maximum
   - Removes blank values early
   - Uses efficient string concatenation
   - Minimizes variable scope

3. **Visual Optimization**
   - Vector-based for sharp rendering
   - Minimal DOM elements per row
   - Efficient text positioning
   - Optimized SVG structure

4. **Usage Guidelines**
   - Best for 5-50 categories
   - Works with any numeric measure
   - Supports hierarchical category columns
   - Compatible with most Power BI filters