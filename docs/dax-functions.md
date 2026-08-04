# DAX Functions Used in This Module

Analytical Models draws on 3 functions from the solution's shared DAX
time-intelligence library — the same reusable engine used across every module. Only
the functions this module actually calls are documented below; the current/prior-year
sales figures the BCG measures consume are themselves built from `YearToDate` and
`PriorYear` in the [Sales Analytics](https://github.com/fuadgashi/Sales-Analytics)
repository's measure catalog.

```dax
/// Year-to-date: evaluates the measure for the current calendar year, dates up to
/// and including today. KEEPFILTERS preserves any existing date filters.
function YearToDate =
    (Measure: anyref expr) =>
        CALCULATE(
            Measure,
            KEEPFILTERS(Calendar[Year] = YEAR(TODAY())),
            KEEPFILTERS(Calendar[Date] <= TODAY())
        )

/// Prior year: shifts the measure back one year using DATEADD and restricts to the
/// previous calendar year. Used for year-over-year comparisons.
function PriorYear =
    (Measure: anyref expr) =>
        CALCULATE(
            Measure,
            DATEADD(Calendar[Date], -1, YEAR),
            Calendar[Year] = YEAR(TODAY()) - 1
        )

/// Rolling 12 months: evaluates the measure for dates from 12 months before today
/// onward, respecting the current filter context. The window every segmentation
/// calculated column in this module ranks products/outlets over.
function Trailing12Months =
    (Measure: anyref expr) =>
        VAR Last12Months = EDATE(TODAY(), -12)
        RETURN
            CALCULATE(Measure, KEEPFILTERS(Calendar[Date] >= Last12Months))
```

These 3 are drawn from a larger 23-function library covering the full solution's
time-intelligence needs — see the [Sales Analytics](https://github.com/fuadgashi/Sales-Analytics)
repository for the complete set.
