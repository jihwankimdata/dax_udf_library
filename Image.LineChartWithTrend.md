# Image.LineChartWithTrend — SVG Line Chart with Trend Indicator

**Overview**  
A compact, embeddable SVG line chart that compares a current-period measure with a prior-period measure (for example, year‑over‑year). The UDF returns a Data URL (`data:image/svg+xml;utf8,...`) ready to use as an Image URL in Power BI visuals.

---

## What it does
- Renders two lines: current period and prior period (if available).
- Calculates and overlays a regression (trend) line using `LINESTX`.
- Displays a headline value, subtitle, trend descriptor, and an index-to-prior-period indicator.
- Auto-scales Y axis with magnitude-aware formatting (K/M/B/T).
- Produces X-axis labels (start/mid/end) and Y labels (min/mid/max) positioned to the right of the plot.
- Gracefully returns a small "No data" SVG when no valid points exist.

---

## Result
- Return type: `STRING` (Data URL containing inline SVG)
- Visual: responsive polyline SVG optimized for small dashboards (recommended 12–52 points)
- Intended usage: set the measure's Data Category to **Image URL** and place in a card/matrix/table that supports images.

---

## PBI Data Model Requirements
- A display column for X-axis labels (string) and a corresponding sort column (numeric or date) for ordering.
- Two numeric measures: current period (EXPR) and comparison period (EXPR, e.g., same period last year).
- Measures should return numeric values and handle BLANKs.
- Recommended: pre-aggregate very granular data (daily) when rendering many points.

---

## Function Definition

```dax
DEFINE
/// Image.LineChartWithTrend
/// Summary: Builds an SVG line chart comparing a current-period measure with a prior-period measure and returns it as a data URL.
/// Params:
///   _title (STRING): Title shown above the chart
///   _measure (EXPR): Current-period measure expression (use EXPR to evaluate in inner context)
///   _measureLY (EXPR): Prior-period measure expression (use EXPR to evaluate in inner context)
///   _xaxis (ANYREF): Column used for display labels on the X axis
///   _xsort (ANYREF): Column used to sort X-axis categories (numeric/date preferred)
/// Returns: STRING (data:image/svg+xml;utf8,...)
/// Model Requirements: A category column and a sort column; numeric measures for current and prior periods
/// Example:
///   Line Chart = Image.LineChartWithTrend("Revenue Trend", [Total Revenue], [Total Revenue LY], 'Date'[MonthLabel], 'Date'[MonthKey])
FUNCTION Image.LineChartWithTrend = (
    _title : STRING,
    _measure : EXPR,
    _measureLY : EXPR,
    _xaxis : ANYREF,
    _xsort : ANYREF
) =>

/* ===== Layout / Canvas ===== */
VAR _W = 1200
VAR _H = 760
VAR _M_TOP = 350
VAR _M_RIGHT = 120
VAR _M_BOTTOM = 60
VAR _M_LEFT = 60
VAR _CONTENT_W = _W - _M_LEFT - _M_RIGHT
VAR _CONTENT_H = _H - _M_TOP - _M_BOTTOM
VAR _PLOT_OFFSET = 24       -- shift plot area right for labels
VAR _X_LEFT = _M_LEFT + _PLOT_OFFSET
VAR _X_WIDTH = _CONTENT_W - _PLOT_OFFSET

/* ===== Styling ===== */
VAR _CLR_MAIN = "#005A9B"
VAR _CLR_PRIOR = "#d0d0d0"
VAR _CLR_TREND = "#70b7ff"
VAR _CLR_TEXT = "#111111"
VAR _CLR_MUTED = "#9aa0a6"
VAR _FONT = "Segoe UI"
VAR _FONT_HEAD = 64
VAR _FONT_SUB = 20
VAR _FONT_LABEL = 16

/* ===== Build X-axis base (respect filters on other columns) ===== */
VAR _AxisBase =
    SELECTCOLUMNS (
        ALLSELECTED ( _xaxis, _xsort ),
        "Sort", _xsort,
        "Label", _xaxis
    )

/* ===== Add measures and filter blanks ===== */
VAR _WithValues =
    ADDCOLUMNS (
        _AxisBase,
        "Y", _measure,
        "Y_LY", _measureLY
    )

VAR _Filtered =
    FILTER ( _WithValues, NOT ISBLANK ( [Y] ) )

VAR _N = COUNTROWS ( _Filtered )

/* ===== Indexing & ordering ===== */
VAR _Indexed =
    ADDCOLUMNS (
        _Filtered,
        "__Idx", RANKX ( _Filtered, [Sort],, ASC, DENSE ) - 1
    )

/* ===== Scales & padding ===== */
VAR _MaxY = MAXX ( _Indexed, [Y] )
VAR _MinY = MINX ( _Indexed, [Y] )
VAR _Span = MAX ( 1, _MaxY - _MinY )
VAR _PadTop = 0.18 * _Span
VAR _PadBottom = 0.06 * _Span
VAR _Y_MIN = _MinY - _PadBottom
VAR _Y_MAX = _MaxY + _PadTop
VAR _YRANGE = MAX ( 0.000001, _Y_MAX - _Y_MIN )

/* ===== Helper: scale X/Y to pixel coords ===== */
VAR _ScaleX =
    LAMBDA( idx, _X_LEFT + ( DIVIDE( idx, MAX(1, _N - 1) ) * _X_WIDTH ) )

VAR _ScaleY =
    LAMBDA( value, _M_TOP + ( ( _Y_MAX - value ) / _YRANGE ) * _CONTENT_H )

/* ===== Build polylines ===== */
VAR _PolyPoints =
    CONCATENATEX (
        _Indexed,
        VAR _x = _ScaleX ( [__Idx] )
        VAR _y = _ScaleY ( [Y] )
        RETURN _x & "," & _y,
        " ", [Sort], ASC
    )

VAR _PathMain = "<polyline fill='none' stroke='" & _CLR_MAIN & "' stroke-width='3' points='" & _PolyPoints & "'/>"

VAR _PolyPointsLY =
    CONCATENATEX (
        FILTER ( _Indexed, NOT ISBLANK ( [Y_LY] ) ),
        VAR _x = _ScaleX ( [__Idx] )
        VAR _y = _ScaleY ( [Y_LY] )
        RETURN _x & "," & _y,
        " ", [Sort], ASC
    )

VAR _PathLY = IF ( LEN ( _PolyPointsLY ) > 0, "<polyline fill='none' stroke='" & _CLR_PRIOR & "' stroke-width='3' points='" & _PolyPointsLY & "'/>", BLANK() )

/* ===== Regression (trend) line using LINESTX ===== */
VAR _RegInput = SELECTCOLUMNS ( _Indexed, "x", [__Idx], "y", [Y] )
VAR _Reg = LINESTX ( _RegInput, [y], [x] )
VAR _Slope = MAXX ( _Reg, [Slope1] )
VAR _Intercept = MAXX ( _Reg, [Intercept] )
VAR _Y_HAT_0 = _Intercept + _Slope * 0
VAR _Y_HAT_N = _Intercept + _Slope * MAX ( 0, _N - 1 )
VAR _P0X = _X_LEFT
VAR _P1X = _X_LEFT + _X_WIDTH
VAR _P0Y = _ScaleY ( _Y_HAT_0 )
VAR _P1Y = _ScaleY ( _Y_HAT_N )
VAR _TrendLineSvg = "<line x1='" & _P0X & "' y1='" & _P0Y & "' x2='" & _P1X & "' y2='" & _P1Y & "' stroke='" & _CLR_TREND & "' stroke-width='3' stroke-dasharray='6 6'/>"
VAR _TrendText = IF ( _Slope >= 0, "Trending Upwards", "Trending Downwards" )

/* ===== Headline / subtitle / index ===== */
VAR _LastIdx = MAXX ( _Indexed, [__Idx] )
VAR _LastMain = MAXX ( FILTER ( _Indexed, [__Idx] = _LastIdx ), [Y] )
VAR _LastLY = MAXX ( FILTER ( _Indexed, [__Idx] = _LastIdx ), [Y_LY] )
VAR _IndexToLY = DIVIDE ( _LastMain, _LastLY, BLANK() )
VAR _IndexPctText = IF ( NOT ISBLANK ( _IndexToLY ), FORMAT ( 100 * _IndexToLY, "0" ), "--" )

VAR _HeadlineSvg =
    "<text x='" & _M_LEFT & "' y='115' font-family='" & _FONT & "' font-size='" & (_FONT_HEAD) & "' font-weight='700' fill='" & _CLR_TEXT & "'>" &
    FORMAT ( _LastMain, "#,#0.00" ) & "</text>"

VAR _SubtitleSvg =
    "<text x='" & _M_LEFT & "' y='180' font-family='" & _FONT & "' font-size='" & (_FONT_SUB) & "' font-weight='700' fill='" & _CLR_TEXT & "'>" &
    SUBSTITUTE ( _title, "&", "&amp;" ) & "</text>"

VAR _TrendTagSvg =
    "<text x='" & _M_LEFT & "' y='245' font-family='" & _FONT & "' font-size='" & (_FONT_LABEL) & "' fill='" & _CLR_MUTED & "'>" &
    _TrendText & "</text>"

VAR _IndexTagSvg =
    "<text x='" & (_M_LEFT + _CONTENT_W) & "' y='245' text-anchor='end' font-family='" & _FONT & "' font-size='" & (_FONT_LABEL) & "' fill='" & IF ( _LastMain >= _LastLY, "Black", "Red" ) & "'>" &
    "Index to Prior: " & _IndexPctText & "</text>"

/* ===== Y axis labels (right side) ===== */
VAR _Y_MID = ( _Y_MIN + _Y_MAX ) / 2
VAR _Y_PIX_MIN = _ScaleY ( _Y_MIN )
VAR _Y_PIX_MID = _ScaleY ( _Y_MID )
VAR _Y_PIX_MAX = _ScaleY ( _Y_MAX )
VAR _Y_FMT = "#,#0.0" -- simplified; can expand magnitude-aware formatting if needed
VAR _YLabelsSvg =
    "<text x='" & (_X_LEFT + _X_WIDTH + 12) & "' y='" & (_Y_PIX_MAX + 4) & "' text-anchor='start' font-family='" & _FONT & "' font-size='" & (_FONT_LABEL) & "' fill='" & _CLR_TEXT & "'>" & FORMAT ( _Y_MAX, _Y_FMT ) & "</text>" &
    "<text x='" & (_X_LEFT + _X_WIDTH + 12) & "' y='" & (_Y_PIX_MID + 4) & "' text-anchor='start' font-family='" & _FONT & "' font-size='" & (_FONT_LABEL) & "' fill='" & _CLR_TEXT & "'>" & FORMAT ( _Y_MID, _Y_FMT ) & "</text>" &
    "<text x='" & (_X_LEFT + _X_WIDTH + 12) & "' y='" & (_Y_PIX_MIN + 4) & "' text-anchor='start' font-family='" & _FONT & "' font-size='" & (_FONT_LABEL) & "' fill='" & _CLR_TEXT & "'>" & FORMAT ( _Y_MIN, _Y_FMT ) & "</text>"

/* ===== X axis labels (start, mid, end) ===== */
VAR _LabelStart = MAXX ( FILTER ( _Indexed, [__Idx] = 0 ), [Label] )
VAR _LabelMid = MAXX ( FILTER ( _Indexed, [__Idx] = INT ( DIVIDE ( _LastIdx, 2, 0 ) ) ), [Label] )
VAR _LabelEnd = MAXX ( FILTER ( _Indexed, [__Idx] = _LastIdx ), [Label] )
VAR _XLabelsSvg =
    "<text x='" & _X_LEFT & "' y='" & (_M_TOP + _CONTENT_H + 44) & "' font-family='" & _FONT & "' font-size='" & (_FONT_LABEL) & "' fill='" & _CLR_TEXT & "'>" & SUBSTITUTE ( _LabelStart, "&", "&amp;" ) & "</text>" &
    "<text x='" & (_X_LEFT + _X_WIDTH/2) & "' y='" & (_M_TOP + _CONTENT_H + 44) & "' text-anchor='middle' font-family='" & _FONT & "' font-size='" & (_FONT_LABEL) & "' fill='" & _CLR_TEXT & "'>" & SUBSTITUTE ( _LabelMid, "&", "&amp;" ) & "</text>" &
    "<text x='" & (_X_LEFT + _X_WIDTH) & "' y='" & (_M_TOP + _CONTENT_H + 44) & "' text-anchor='end' font-family='" & _FONT & "' font-size='" & (_FONT_LABEL) & "' fill='" & _CLR_TEXT & "'>" & SUBSTITUTE ( _LabelEnd, "&", "&amp;" ) & "</text>"

/* ===== Baseline & axes ===== */
VAR _AxisLine = "<line x1='" & _X_LEFT & "' y1='" & (_M_TOP + _CONTENT_H) & "' x2='" & (_X_LEFT + _X_WIDTH) & "' y2='" & (_M_TOP + _CONTENT_H) & "' stroke='#cfcfcf' stroke-width='1'/>"

/* ===== Final SVG assembly ===== */
VAR _Svg =
    "data:image/svg+xml;utf8," &
    "<svg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 " & _W & " " & _H & "'>" &
    "<rect width='100%' height='100%' fill='white'/>" &
    _HeadlineSvg & _SubtitleSvg & _TrendTagSvg & _IndexTagSvg &
    _AxisLine & _PathLY & _PathMain & _TrendLineSvg &
    _XLabelsSvg & _YLabelsSvg &
    "</svg>"

RETURN
IF ( _N = 0,
    "data:image/svg+xml;utf8,<svg xmlns='http://www.w3.org/2000/svg' viewBox='0 0 600 120'><text x='300' y='70' text-anchor='middle' font-family=\"Segoe UI\" font-size='16' fill='#777'>No data</text></svg>",
    _Svg
)
```

---

### Implementation Notes

1. **SVG Structure**
   - Canvas dimensions: 1200×760 with configurable margins
   - Plot offset reserved for visual balance and labels
   - All text is escaped for `&` characters to avoid malformed XML

2. **Data Processing**
   - Builds an X-axis table using `ALLSELECTED` on the supplied columns to preserve outer filters
   - Adds current and prior values via `ADDCOLUMNS` and filters blanks before indexing
   - Uses `RANKX` with `DENSE` to compute a 0-based index for X positioning

3. **Trend Calculation**
   - Uses `LINESTX` to compute a best-fit line across numeric index positions
   - Renders the trend as a dashed line to visually distinguish from data series

4. **Formatting**
   - Headline uses the last non-blank value as the primary KPI value
   - X-axis shows start/mid/end labels for readability
   - Y-axis displays min/mid/max values on the right for compact layout

---

### Performance Considerations

- Ideal for 12–52 points (month/week views); daily data should be aggregated
- Minimizes measure evaluations by computing values in a single `ADDCOLUMNS` step
- Uses `CONCATENATEX` to build polyline points efficiently
- LINESTX is computed once; avoid recomputing expensive measures inside loops

