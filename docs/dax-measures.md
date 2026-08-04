# DAX Measure Catalog

Formula logic is reproduced exactly as authored; only identifiers are translated to
English. Two families: the BCG growth-share matrix (run twice, at brand and product
grain — see the [design note](data-model.md#design-note-two-bcg-models-at-two-different-grains)),
and the dynamic ABCD segmentation.

## Brand-level BCG matrix

Classic BCG framing: each brand's share of the *whole portfolio*, benchmarked against
the 3rd-largest brand, crossed with YoY growth benchmarked against a threshold that's
computed from the portfolio's own average — not a hard-coded "10% growth = good"
rule.

```dax
-- Hidden helper: this brand's share of total portfolio sales
Brand Share of Total Sales =
    DIVIDE(
        [Net Sales Value, Current Year YTD],
        CALCULATE([Net Sales Value, Current Year YTD], ALL(Products[BrandId])),
        0
    )

-- Highest single-brand share across the whole portfolio
Max Brand Share =
    CALCULATE(
        MAXX(ALL(Products[BrandId]), [Brand Share of Total Sales]),
        ALL(Products[BrandId])
    )

-- This brand's share relative to the 3rd-largest brand, with a minimum-sales noise
-- filter so a brand with negligible volume doesn't get an inflated relative share
Brand Relative Share =
    VAR CurrentShare = [Brand Share of Total Sales]
    VAR MinimumSalesThreshold = 1000
    VAR RankedBrands =
        ADDCOLUMNS(
            FILTER(ALL(Products[BrandId]), [Brand Share of Total Sales] > 0),
            "BrandShare", [Brand Share of Total Sales]
        )
    VAR ThirdLargestShare =
        MINX(TOPN(4, RankedBrands, [BrandShare], DESC), [BrandShare])
    RETURN
        IF(
            CurrentShare <= 0
                || ISBLANK(CurrentShare)
                || [Net Sales Value, Current Year YTD] < MinimumSalesThreshold,
            BLANK(),
            DIVIDE(CurrentShare, ThirdLargestShare)
        )

-- Log-scaled so the share axis reads intuitively: 0 = same share as the benchmark,
-- positive = ahead, negative = behind — instead of a raw ratio that skews wildly
Brand Relative Share (Log) =
    VAR Share = [Brand Relative Share]
    RETURN
        IF(Share <= 0 || ISBLANK(Share), BLANK(), LOG10(Share))

-- YoY growth, with a guard so a brand new to the portfolio reads as "100% growth"
-- rather than a divide-by-zero blank
Brand Growth Index, YTD =
    VAR PriorYearSales  = [Net Sales Value, Prior Year YTD]
    VAR CurrentYearSales = [Net Sales Value, Current Year YTD]
    RETURN
        IF(
            CurrentYearSales <= 0 || ISBLANK(CurrentYearSales),
            BLANK(),                                              -- no current-year sales = excluded from the matrix
            IF(
                ISBLANK(PriorYearSales) || PriorYearSales <= 0,
                1,                                                -- new brand = treated as 100% growth
                DIVIDE(CurrentYearSales - PriorYearSales, PriorYearSales)
            )
        )

-- The growth threshold isn't hard-coded — it's the average growth rate across every
-- brand that actually has valid growth data, so the "good vs. bad growth" line moves
-- with the portfolio's own performance each period
Brand Growth Threshold (Average) =
    AVERAGEX(
        FILTER(
            ALL(Products[BrandId]),
            [Brand Growth Index, YTD] <> BLANK() && [Net Sales Value, Current Year YTD] > 0
        ),
        [Brand Growth Index, YTD]
    )

-- The quadrant classification itself
Brand BCG Quadrant =
    VAR Share           = [Brand Relative Share (Log)]
    VAR Growth          = [Brand Growth Index, YTD]
    VAR GrowthThreshold = [Brand Growth Threshold (Average)]
    VAR ShareThreshold  = 0.1
    RETURN
        IF(
            ISBLANK(Share) || ISBLANK(Growth),
            BLANK(),
            SWITCH(
                TRUE(),
                Share >= ShareThreshold && Growth >= GrowthThreshold, "⭐ Star",
                Share >= ShareThreshold && Growth <  GrowthThreshold, "🐄 Cash Cow",
                Share <  ShareThreshold && Growth >= GrowthThreshold, "❓ Question Mark",
                Share <  ShareThreshold && Growth <  GrowthThreshold, "🐕 Dog"
            )
        )
```

## Product-level BCG matrix

The product model uses a **different competitive set on purpose**: a product's
relevant rivals are the other SKUs in its own brand's lineup, not the entire catalog
— so share is measured against the *average of the 2nd-through-5th best sellers in
that same brand*, and the quadrant is delivered through a slicer-driven selector
pattern (built for an interactive scatter matrix) rather than a static text label.

```dax
-- Guards out zero/blank sales so a discontinued or not-yet-launched product doesn't
-- show as a valid comparison point on the matrix
Current Year Sales (Positive Only) =
    VAR CurrentYearSales = [Net Sales Value, Current Year YTD]
    RETURN
        IF(ISBLANK(CurrentYearSales) || CurrentYearSales <= 0, BLANK(), CurrentYearSales)

Prior Year Sales (Positive Only) =
    VAR PriorYearSales = [Net Sales Value, Prior Year YTD]
    RETURN
        IF(ISBLANK(PriorYearSales) || PriorYearSales <= 0, BLANK(), PriorYearSales)

-- Product growth, with explicit -1/+1 signals for discontinued vs. newly launched
-- products rather than letting DIVIDE silently return blank for either
Product Growth Index =
    VAR CurrentSales  = [Current Year Sales (Positive Only)]
    VAR PriorSales    = [Prior Year Sales (Positive Only)]
    RETURN
        SWITCH(
            TRUE(),
            (CurrentSales <= 0 || ISBLANK(CurrentSales)) && PriorSales > 0, -1,   -- discontinued
            (PriorSales <= 0 || ISBLANK(PriorSales)) && CurrentSales > 0,   1,   -- newly launched
            DIVIDE(CurrentSales - PriorSales, PriorSales)                        -- standard growth
        )

-- Log-scaled share of THIS product vs. the average of the 2nd-5th best sellers in
-- its own brand — deliberately excludes the #1 seller so one dominant SKU doesn't
-- drag every other product in the brand into the "low share" quadrant
Product Share of Brand (Log) =
    VAR CurrentProductSales = [Current Year Sales (Positive Only)]
    VAR AllProductsInBrand =
        CALCULATETABLE(Products, ALLEXCEPT(Products, Products[BrandId]))
    VAR Top5 =
        TOPN(5, AllProductsInBrand, [Current Year Sales (Positive Only)], DESC)
    VAR Top1 =
        TOPN(1, AllProductsInBrand, [Current Year Sales (Positive Only)], DESC)
    VAR Top2Through5 =
        EXCEPT(Top5, Top1)
    VAR AverageTop2Through5Sales =
        AVERAGEX(Top2Through5, [Current Year Sales (Positive Only)])
    VAR RelativeShare =
        DIVIDE(CurrentProductSales, AverageTop2Through5Sales)
    RETURN
        IF(RelativeShare > 0, LOG10(RelativeShare), BLANK())

-- Slicer-driven quadrant selector: the BCG Growth-Share Matrix table's `Sort` column
-- (1=Star, 2=Cash Cow, 3=Question Mark, 4=Dog) drives which single quadrant's
-- products are shown, letting one scatter chart filter to one quadrant at a time
Product BCG Quadrant Selector =
    VAR Sales    = [Current Year Sales (Positive Only)]
    VAR LogShare = [Product Share of Brand (Log)]
    VAR Growth   = [Product Growth Index]
    VAR QuadrantSort =
        IF(
            ISBLANK(LogShare),
            BLANK(),
            SWITCH(
                TRUE(),
                LogShare >= 0 && Growth <  0.05, 1,   -- high share, low growth
                LogShare >= 0 && Growth >= 0.05, 2,   -- high share, high growth
                LogShare <  0 && Growth >= 0.05, 3,   -- low share, high growth
                4                                     -- low share, low growth
            )
        )
    VAR SelectedSort = INT(SELECTEDVALUE('BCG Growth-Share Matrix'[Sort]))
    RETURN
        IF(ISBLANK(SelectedSort) || QuadrantSort = SelectedSort, QuadrantSort, BLANK())
```

## Dynamic ABCD segmentation

Both the Products and Customer Outlets tables carry a percentile-based grouping
**calculated column** — segments recompute automatically on every refresh, with no
manually maintained tiering list to go stale.

```dax
-- Products: tiered by trailing-12-month sales AND the number of distinct stores
-- that bought it (store distribution) — a product needs BOTH high sales and broad
-- reach to land in the top tier, not just one or the other
Product Group =
    VAR ActiveProducts =
        FILTER(
            ALL(Products),
            NOT ISBLANK([Sales, Trailing 12 Months]) && NOT ISBLANK([Store Distribution, Trailing 12 Months])
        )
    VAR SalesP75 = PERCENTILEX.EXC(ActiveProducts, [Sales, Trailing 12 Months], 0.75)
    VAR SalesP50 = PERCENTILEX.EXC(ActiveProducts, [Sales, Trailing 12 Months], 0.50)
    VAR SalesP25 = PERCENTILEX.EXC(ActiveProducts, [Sales, Trailing 12 Months], 0.25)
    VAR DistP75  = PERCENTILEX.EXC(ActiveProducts, [Store Distribution, Trailing 12 Months], 0.75)
    VAR DistP50  = PERCENTILEX.EXC(ActiveProducts, [Store Distribution, Trailing 12 Months], 0.50)
    VAR DistP25  = PERCENTILEX.EXC(ActiveProducts, [Store Distribution, Trailing 12 Months], 0.25)
    VAR Sales = [Sales, Trailing 12 Months]
    VAR Dist  = [Store Distribution, Trailing 12 Months]
    RETURN
        IF(
            ISBLANK(Sales) || ISBLANK(Dist),
            BLANK(),
            SWITCH(
                TRUE(),
                Sales >= SalesP75 && Dist >= DistP75, "Group D",   -- top performers
                Sales >= SalesP50 && Dist >= DistP50, "Group C",
                Sales >= SalesP25 && Dist >= DistP25, "Group B",
                Sales <  SalesP25 && Dist <  DistP25, "Group A",   -- long tail
                BLANK()
            )
        )

-- Supporting measures the column above ranks against — sales and reach, each with
-- a floor that filters out noise (e.g. a single test transaction) before ranking
Sales, Trailing 12 Months =
    VAR SalesLast12Months = Trailing12Months([Net Sales Value])
    RETURN
        IF(ISBLANK(SalesLast12Months) || SalesLast12Months <= 10, BLANK(), SalesLast12Months)

Store Distribution, Trailing 12 Months =
    VAR DistributionLast12Months = Trailing12Months(DISTINCTCOUNT('Sales Invoice Lines'[OutletId]))
    RETURN
        IF(ISBLANK(DistributionLast12Months) || DistributionLast12Months <= 5, BLANK(), DistributionLast12Months)

-- Customer Outlets: the identical percentile pattern, tiered by trailing-12-month
-- sales AND assortment breadth (distinct products purchased) instead of store reach
Outlet Group =
    VAR ActiveOutlets =
        FILTER(
            ALL('Customer Outlets'),
            NOT ISBLANK([Sales, Trailing 12 Months]) && NOT ISBLANK([Assortment Breadth, Trailing 12 Months])
        )
    VAR SalesP75 = PERCENTILEX.EXC(ActiveOutlets, [Sales, Trailing 12 Months], 0.75)
    VAR SalesP50 = PERCENTILEX.EXC(ActiveOutlets, [Sales, Trailing 12 Months], 0.50)
    VAR SalesP25 = PERCENTILEX.EXC(ActiveOutlets, [Sales, Trailing 12 Months], 0.25)
    VAR AssortP75 = PERCENTILEX.EXC(ActiveOutlets, [Assortment Breadth, Trailing 12 Months], 0.75)
    VAR AssortP50 = PERCENTILEX.EXC(ActiveOutlets, [Assortment Breadth, Trailing 12 Months], 0.50)
    VAR AssortP25 = PERCENTILEX.EXC(ActiveOutlets, [Assortment Breadth, Trailing 12 Months], 0.25)
    VAR Sales   = [Sales, Trailing 12 Months]
    VAR Assort  = [Assortment Breadth, Trailing 12 Months]
    RETURN
        IF(
            ISBLANK(Sales) || ISBLANK(Assort),
            BLANK(),
            SWITCH(
                TRUE(),
                Sales >= SalesP75 && Assort >= AssortP75, "Group A",   -- top-tier accounts
                Sales >= SalesP50 && Assort >= AssortP50, "Group B",
                Sales >= SalesP25 && Assort >= AssortP25, "Group C",
                Sales <  SalesP25 && Assort <  AssortP25, "Group D",  -- long tail
                BLANK()
            )
        )

Assortment Breadth, Trailing 12 Months =
    VAR ProductsLast12Months = Trailing12Months(DISTINCTCOUNT('Sales Invoice Lines'[ProductId]))
    RETURN
        IF(ISBLANK(ProductsLast12Months) || ProductsLast12Months <= 1, BLANK(), ProductsLast12Months)
```

> **Note on labeling.** The Products and Customer Outlets segmentations intentionally
> assign the letters in opposite directions — `Group D` is the top tier for products,
> `Group A` is the top tier for outlets — reflecting how each was originally
> designed (an outlet's "A-list" vs. a product catalog ranked toward its most
> established, "further-along" tier). Reproduced faithfully as-authored.
