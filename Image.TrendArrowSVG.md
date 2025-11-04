# Image.TrendArrowSVG — SVG Trend Indicator for KPI Direction

**Overview**  
Generates a **dynamic SVG arrow icon** (up or down) in Power BI visuals, with color-coded meaning (good/bad) defined by business logic.  
Returns an **Image URL string** compatible with *Data Category → Image URL*.

---

## What it does
- Creates a **vector-based arrow icon** that visually indicates trend direction
- Applies **color-coding** based on business context:
  - Green (#00FF00) for positive/good trends
  - Red (#FF0000) for negative/bad trends
  - Gray (#E6E6E6) for neutral/default cases
- Generates **inline SVG** - no external image assets needed
- Returns a **complete image URL** ready for Power BI visuals

---

## Result
**Type:** `STRING`  
**Output Format:** `data:image/svg+xml;utf8,...`  
**Visual Properties:**
- 24x24 viewbox (scalable)
- Vector-based (sharp at any size)
- Directional (up/down arrow)
- Color-coded (green/red/gray)

---

## PBI Data Model Requirements

### Required Configuration
- Output measure must have **Data Category** set to **Image URL**
- Visual must support image URL rendering:
  - Tables
  - Matrices
  - Cards
  - Other image-capable visuals

### Parameter Values
- `_trendDirection`: "up" or "down"
- `_trendMeaning`: "good" or "bad"

---

## Function Definition
```dax
DEFINE
/// Image.TrendArrowImage
/// Summary: Generates a color-coded SVG arrow (up or down) for KPI trend visualization.
/// Params:
///   _trendDirection (STRING VAL): "up" or "down" - Direction of the arrow
///   _trendMeaning (STRING VAL): "good" or "bad" - Business context for coloring
/// Returns: STRING - SVG image URL for Power BI visuals
/// Model Requirements: None, but output measure must have Data Category = Image URL
FUNCTION Image.TrendArrowSVG = (
    _trendDirection : STRING,
    _trendMeaning : STRING
) =>

VAR _ColorHex =
    SWITCH (
        TRUE (),
        _trendMeaning = "bad",  "#FF0000",  -- Red
        _trendMeaning = "good", "#00FF00",  -- Green
        "#E6E6E6"                           -- Neutral gray (default)
    )

VAR _ArrowUp =
    "<svg xmlns='http://www.w3.org/2000/svg' height='24' viewBox='0 -960 960 960' width='24' fill='" & _ColorHex & "'>" &
    "<path d='m136-240-56-56 296-298 160 160 208-206H640v-80h240v240h-80v-104L536-320 376-480 136-240Z'/>" &
    "</svg>"

VAR _ArrowDown =
    "<svg xmlns='http://www.w3.org/2000/svg' height='24' viewBox='0 -960 960 960' width='24' fill='" & _ColorHex & "'>" &
    "<path d='M640-240v-80h104L536-526 376-366 80-664l56-56 240 240 160-160 264 264v-104h80v240H640Z'/>" &
    "</svg>"

VAR _ArrowSVG =
    SWITCH (
        TRUE (),
        _trendDirection = "up",   _ArrowUp,
        _trendDirection = "down", _ArrowDown
    )

RETURN
    "data:image/svg+xml;utf8," & _ArrowSVG
```

### Implementation Notes

1. **SVG Structure**
   - Uses Material Design icon paths
   - Implements viewBox for consistent scaling
   - Maintains aspect ratio automatically
   - Supports all modern browsers

2. **Color Logic**
   - Green (#00FF00) indicates positive/good
   - Red (#FF0000) indicates negative/bad
   - Gray (#E6E6E6) for undefined states
   - Colors can be customized in code

3. **Usage Strategy**
   - Pass trend direction based on data comparison
   - Define meaning based on business rules
   - Set Data Category to Image URL
   - Apply to visual that supports images

4. **Error Prevention**
   - Falls back to gray if meaning undefined
   - Validates direction and meaning parameters
   - Uses safe SVG encoding
   - Handles special characters properly

### Performance Considerations

1. **Execution Efficiency**
   - Minimal calculation overhead
   - Simple string operations only
   - No table scanning or iteration
   - No external resource loading

2. **Memory Impact**
   - Small string footprint
   - No temporary storage needed
   - Constant memory usage
   - No caching required

3. **Optimization Benefits**
   - Vector-based (scales without quality loss)
   - Inline SVG (no external dependencies)
   - Browser-native rendering
   - Instant visual updates

4. **Usage Guidelines**
   - Suitable for large tables/matrices
   - Works well with frequent refreshes
   - Supports high-DPI displays
   - Compatible with mobile viewing
