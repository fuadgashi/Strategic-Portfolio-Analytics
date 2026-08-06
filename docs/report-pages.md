# Report Tour — Analytical Models

4 self-contained portfolio-analysis pages built on the BCG growth-share model and
percentile-segmentation logic in the [DAX catalog](dax-measures.md).

| Page | What it shows |
|---|---|
| **BCG Matrix – Brand** | Scatter plot positioning every brand by relative market share (log) vs. YoY growth index, quadrant-classified into Star / Cash Cow / Question Mark / Dog against a dynamically averaged growth threshold. |
| **BCG Matrix – Product** | The product-grain model — each product benchmarked against the 2nd-5th best sellers in its own brand — plus a brand-level sales comparison (current vs. prior year) chart, and a slicer that filters the matrix to one quadrant at a time. |
| **Product Grouping** | Dynamic ABCD segmentation of products by trailing-12-month sales & store distribution (percentile-based, recomputed on refresh) — scatter, donut, and trend views by group. |
| **Customer Grouping** | The same segmentation pattern applied to customer outlets by trailing-12-month sales & assortment breadth (distinct products purchased). |

Both BCG pages and both grouping pages share the filter panel (date range,
division/brand, product) and navigator shell used across the solution — all
localized through the dynamic title/label system described in
[Dynamic Titles & Localization](dynamic-titles.md).
