# Retail Fashion Report (Power BI)

An end-to-end Power BI project built from a raw fashion-retail CSV — 2,176 products across 6 categories, 8 brands, covering pricing, markdowns, stock, customer ratings, and returns. This README walks through the process from raw file to final dashboard: what the data actually was, the decisions I made while cleaning and modeling it, and what came out the other end.

## Files

- `Retail_Fashion_Report.pbix` — the dashboard (4 pages: Overview, Return Reason Analysis, Markdown Analysis, Key Insights)
- `fashion_boutique_dataset.csv` — the raw source data
- This README

## 1. Understanding the raw data first

Before touching a single visual, I opened the raw CSV to see what I was actually working with:

- **2,176 rows, 14 columns.** Prices came in as text with currency symbols (`$196.00`), markdown as text percentages (`35%`), and dates as long-form text (`06 August 2025`) — none of it usable for calculation until converted.
- **Product ID was unique on every single row.** That one check shaped a lot of what came after: this isn't a log of individual purchases, it's a snapshot of the product catalog — one row per product, with its current price, stock on hand, and (if applicable) a return outcome. There's no repeat-purchase or transaction-count field anywhere in the data.
- **Purchase Date was heavily concentrated**: 1,641 of 2,176 rows (75.4%) share the exact same date. The remaining rows are spread thinly across the rest of the year. Read literally, "Purchase Date" behaves more like a stock-load date than a real, evenly-distributed transaction date.
- **Returns**: 320 of 2,176 products (14.7%) were marked returned, split across six reasons (Size Issue, Quality Issue, Wrong Item, Color Mismatch, Damaged, Changed Mind).
- **Pricing**: current price ranges from $7 to $249, averaging $85.

These four observations directly shaped how I named metrics and structured the model later — I wanted the dashboard's language to match what the data could actually support, not what a retail dashboard "usually" says.

## 2. Cleaning & transforming (Power Query)

Using Power Query, I turned the raw text fields into usable types:

- Stripped currency symbols and converted `Original Price` / `Current Price` to numeric (rounded down / up respectively, to keep whole-currency values consistent).
- Converted `Markdown Percentage` from a text percentage to a true decimal percentage field.
- Parsed `Purchase Date` from long-form text into a proper Date type.
- Verified row uniqueness and checked for duplicate records (none found) — kept a `Table.Distinct` step as a safeguard regardless.
- Renamed all fields to clean, consistent, human-readable names (`Product Id`, `Category`, `Original Price`, etc.) instead of the raw file's inconsistent casing.

Final query: 16 clear, purposeful steps — no leftover clutter from exploratory clicking.

## 3. Modeling the data

I built a proper **star-schema-style model** rather than working off a single flat table:

- A dedicated `Date` table, generated as a calendar spanning the actual range of `Purchase Date` in the data — deliberately *not* padded to a full calendar year. Time-intelligence functions like `DATESMTD`/`DATESYTD` evaluate against the last date present in the filtered date table; padding to a full year would mean that "last date" often lands on a period with no underlying data, silently returning blank results. Scoping the calendar to the real data range avoids that.
- A single active relationship: `Products[Purchase Date]` → `Date[Date]`, many-to-one, so slicers and time-based measures filter correctly across the model.
- The `Date` table marked explicitly as a date table, so DAX time intelligence resolves correctly.

## 4. Defining the metrics

Given that every row represents a distinct product rather than a transaction, I named the core value metric **Inventory Value** (`Current Price × Stock Quantity`) — the value of stock currently on hand, not completed revenue. I built the DAX consistently around that definition rather than labeling it "Sales," which the data doesn't actually support (there's no unit-sold or transaction-count field to calculate real sales from).

Key measures:

```DAX
Total Inventory Value = SUM('Products'[Inventory Value])

Average Inventory value = AVERAGE('Products'[Inventory Value])

Inventory Value Added YTD =
CALCULATE([Total Inventory Value], DATESYTD('Date'[Date]))

Return Rate =
DIVIDE(
    CALCULATE(COUNTROWS('Products'), 'Products'[Is Returned] = TRUE),
    COUNTROWS('Products')
)

% of Total Inventory Value =
DIVIDE([Total Inventory Value], CALCULATE([Total Inventory Value], ALL('Products')))

Brand Rank by Value =
RANKX(ALL('Products'[Brand]), [Total Inventory Value], , DESC)
```

`DIVIDE` throughout to guard against divide-by-zero, `ALL` to compute share-of-total independent of filter context, `RANKX` for brand ranking.

## 5. Building the report

Four pages, each with a specific job:

- **Overview** — product mix by category/brand, stock by size, pricing range, return reasons at a glance.
- **Markdown Analysis** — markdown percentage against customer rating by category, price vs. stock by brand.
- **Return Reason Analysis** — return reasons by category/color/season, stock quantity by return status, inventory value trend by month.
- **Key Insights** — the analysis written in plain language (below).

Every visual has a specific, descriptive title — no default "Sum of X by Y" labels left in the final version.

## 6. What the data actually shows

- **Returns are driven by fit and quality, not price.** After "Changed Mind," *Size Issue* and *Quality Issue* are the next-largest return drivers — ahead of Wrong Item, Color Mismatch, and Damaged. Sizing accuracy and QA look like the higher-leverage fix compared to pricing adjustments.
- **Category mix is balanced.** All six categories sit within a 10–16% share of total product count — no single category dominates inventory.
- **Pricing spans a wide range** ($7–$249, average $85), which argues for tiered promotional rules by price band rather than one flat markdown percentage.
- **Purchase Date is concentrated on one date (75.4% of records)** — most likely a bulk data-load artifact rather than real transaction timing. Any month/quarter/year view in this report should be read as directional, not as a confirmed trend, until transaction-level dates are available.
- **This is inventory data, not completed sales.** Every product row is unique — "Inventory Value" represents potential revenue if fully sold, not revenue already booked. A future version of this analysis would benefit from real transaction/order-level data to report true sales performance.

## Tech used

Power BI Desktop, Power Query (M), DAX (time intelligence, `DIVIDE`, `RANKX`, `ALL`).

---

*The `.pbix` data source path is set locally. To refresh, point it at your own copy of the CSV via Transform Data → Data Source Settings.*
