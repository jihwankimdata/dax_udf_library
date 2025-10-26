````markdown
# Image.TrendArrowImage

### Summary
Returns a dynamic SVG arrow icon (up/down) with color-coding (good/bad) for KPI trend visualization in Power BI visuals.

### Function Definition
```dax
DEFINE
/// Image.TrendArrowImage
/// Summary: Generates a color-coded SVG arrow (up or down) for KPI trend visualization.
/// Params:
///   _trendDirection (STRING VAL): "up" or "down" - Direction of the arrow
///   _trendMeaning (STRING VAL): "good" or "bad" - Business context for coloring
/// Returns: STRING - SVG image URL for Power BI visuals
/// Model Requirements: None, but output measure must have Data Category = Image URL
/// Example: 
///   Sales Trend Icon = Image.TrendArrowImage(IF([Sales Growth] > 0, "up", "down"), IF([Sales Growth] > Target, "good", "bad"))
/// Notes: 
///   - Returns inline SVG (no external assets needed)
///   - 24x24 viewbox, scalable
///   - Negligible performance impact (string operations only)
FUNCTION Image.TrendArrowImage = (
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

### Model Requirements
- No specific table requirements or relationships needed
- Output measure must have **Data Category** set to **Image URL**
- Power BI visual must support image URL rendering (Tables, Matrices, Cards)

### Performance Best Practices
1. Uses `VAL` mode for parameters as they are scalar values
2. No context manipulation needed
3. Lightweight string operations only - negligible performance impact
4. No table scanning or iteration

### Usage
1. Create a measure that determines trend direction and meaning
2. Pass those values to this function
3. Set the resulting measure's Data Category to Image URL
4. Use in Power BI visuals that support image display

### Example Implementation
```dax
-- Example measure using the function
Sales Trend Indicator = 
VAR Direction = IF([Sales vs LY %] > 0, "up", "down")
VAR Meaning = IF([Sales vs LY %] > [Target Growth %], "good", "bad")
RETURN
    Image.TrendArrowImage(Direction, Meaning)
```
````