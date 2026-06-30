# Retail Sales Analysis — Power BI Report

An end-to-end business intelligence project analysing retail sales performance for a company operating across **Germany, the Czech Republic, and Denmark** over **fiscal years 2014 to 2017**. The project covers the complete analytics lifecycle: extracting data from multiple disconnected sources, cleaning and transforming it, modelling it into a coherent schema, building the calculation layer, and presenting the findings through an interactive nine-page report.

---

## Project Objective

Retail businesses generate data across many disconnected systems — transactional records in flat files, product catalogues in databases, reference data on the web. On their own, these fragments answer very little. The objective of this project is to consolidate those scattered sources into a single, trustworthy analytical model and turn them into clear answers to the questions leadership actually asks: where is growth coming from, which markets and products drive profit, and who is delivering it.

Concretely, the project sets out to:

- Integrate data from a PostgreSQL database, a dynamic folder of CSV files, a web-scraped table, and several flat files into one model.
- Build a refresh-resilient pipeline that does not break when source files are added or removed.
- Establish a clean, well-related data model suitable for reliable aggregation and time-intelligence analysis.
- Quantify revenue, cost, and gross profit performance, and surface the drivers behind them.
- Deliver the analysis through an interactive report that a non-technical stakeholder can navigate and act on.

---

## Table of Contents

- [Business Context](#business-context)
- [Business Questions](#business-questions)
- [Headline Results](#headline-results)
- [Data Sources](#data-sources)
- [Data Cleaning and Transformation](#data-cleaning-and-transformation)
- [Data Modeling](#data-modeling)
- [Time Intelligence: The Calendar Table](#time-intelligence-the-calendar-table)
- [Calculated Columns and Measures](#calculated-columns-and-measures)
- [Outputs](#outputs)
- [Insights](#insights)
- [Recommendations](#recommendations)
- [Repository Structure](#repository-structure)
- [Getting Started](#getting-started)
- [Tools and Technologies](#tools-and-technologies)

---

## Business Context

The company sells a catalogue of products through a team of sales representatives across three European markets. Sales are recorded continuously and stored as yearly transaction files, while supporting reference data — products, categories, sales staff, and geography — lives in separate systems maintained by different parts of the business.

Leadership needs a consolidated view that answers strategic questions about growth, market strength, product mix, and team performance. Because the underlying data is deliberately spread across heterogeneous sources, the project also serves as a demonstration of building a robust, production-style data pipeline rather than a single-file exercise.

---

## Business Questions

The report is organised around four core questions. Each maps to one or more pages in the final report.

| Theme | Question | Where it is answered |
|---|---|---|
| **Growth Velocity** | Where and when did revenue experience its most significant Month-over-Month and Quarter-over-Quarter inflection points? | MoM and QoQ Growth |
| **Market Dominance and Volume** | How does the top-performing region compare to emerging territories in revenue, volume, and transaction efficiency? | Revenue by Country/Year, Country/Town Performance |
| **Portfolio Mix Analysis** | Which product categories and price tiers (Low, Medium, High Value) yield the highest relative gross margin? | Category KPIs, Price Range KPIs |
| **Human Capital Performance** | Who are the core sales representatives anchoring the company's bottom-line profitability? | Top Sales Representatives |

> The full problem statement and objectives are documented in [`business-questions/`](business-questions/).

---

## Headline Results

| Metric | Value |
|---|---|
| Total Revenue | $126.01M |
| Total Cost | $39.13M |
| Gross Profit | $86.89M |
| Gross Profit Margin | approximately 68.95% |
| Period Covered | FY 2014 to FY 2017 |
| Markets | Germany, Czech Republic, Denmark |

---

## Data Sources

A defining feature of this project is that the data was deliberately kept in diverse locations to reflect a realistic enterprise environment. Each source was connected through its own appropriate connector in Power Query.

| Table | Source | Connector / Method |
|---|---|---|
| **DimProducts** | PostgreSQL database | Direct database connection (`PostgreSQL.Database`) |
| **FactSales** | A folder of yearly sales CSV files | Folder connector (`Folder.Files`), combining all files dynamically |
| **DimGeography** | A table published on a website | Web scraping (`Web` connector) |
| **DimCategory** | Flat file | Excel workbook |
| **DimSubCategory** | Flat file | Excel workbook |
| **DimSalesRep** | Flat file | Excel workbook |

The sales data deserves a special mention. Rather than binding to specific file names, the FactSales query points at the **folder** itself. This means the pipeline is refresh-resilient: a new yearly sales file can be dropped into the folder, or an existing one removed, and the next refresh simply re-combines whatever files are present without raising an error. This is the kind of robustness expected of a production pipeline.

---

## Data Cleaning and Transformation

Because the sources were independently maintained, each arrived with its own quirks. The following transformations were applied in Power Query to bring every table to a clean, model-ready state. Expand each table below to see what was done and why.

<details>
<summary><strong>DimProducts</strong> (from PostgreSQL)</summary>

<br>

- Connected to the `public.products` table in the PostgreSQL database.
- Applied **Remove Duplicates** (`Table.Distinct`) — the source contained duplicate product records (for example, ProductID 7 and 10 appeared more than once), which would have broken the one-to-many relationship to the fact table.
- Renamed the database's lowercase column names into clean, consistent names (`productid` to `ProductID`, `color` to `Colour`, `productname` to `ProductName`, and so on).

</details>

<details>
<summary><strong>FactSales</strong> (from a CSV folder)</summary>

<br>

- Used the **Folder** connector and filtered out hidden system files.
- Invoked the sample **Transform File** function to parse and expand every CSV into a single combined table.
- Renamed the primary key from `fSalesPrimaryKey` to `SalesPrimaryKey` and set key columns to numeric types.
- The source carried location as a single `Location` field formatted as `Country;Town`. This was **split on the semicolon** into separate Country and Town columns.
- Performed a **left outer merge** against DimGeography on Country and Town, then expanded only the `GeoKey` back into the fact. The redundant Country and Town columns were then removed, since the surrogate key now carries that context.

This merge-based lookup is how the fact table acquires its geography surrogate key rather than storing geography text directly.

</details>

<details>
<summary><strong>DimGeography</strong> (web-scraped)</summary>

<br>

- Promoted the first row to headers, producing Country, Town, and a reference link column.
- Added an **index column** starting at 1 and renamed it `GeoKey`, creating a surrogate key for the dimension.
- Reordered the key to the first position. This `GeoKey` is the column the fact table later joins to.

</details>

<details>
<summary><strong>DimSubCategory</strong> (flat file)</summary>

<br>

- The foreign key `CategoryKey` arrived as inconsistent text such as `"ID -1"` and `"ID - 2"`, with irregular spacing.
- Applied two **Replace Value** steps to strip the `"ID -"` and `"ID - "` prefixes, then converted the cleaned result to a whole number.
- Without this step, the relationship from SubCategory up to Category would not have formed, and the entire category branch of the model would have failed to filter.

</details>

<details>
<summary><strong>DimSalesRep</strong> (flat file)</summary>

<br>

- The `SalesRepID` carried the same `"ID - "` text prefix seen in SubCategory.
- Stripped the prefix and converted the column to a whole number so it could relate cleanly to the fact table.

</details>

<details>
<summary><strong>DimCategory</strong> (flat file)</summary>

<br>

- Promoted headers, set `CategoryKey` to a whole number and the category label to text.
- Renamed the descriptive column to `CategoryName` for clarity. This table was the cleanest of the sources and needed minimal work.

</details>

---

## Data Modeling

The cleaned tables were related into a **hybrid star and snowflake schema** with `FactSales` at the centre. The model uses six relationships, each **one-to-many** and **single cross-filter direction**, flowing from the dimension down to the fact.

| From (one side) | To (many side) | Key |
|---|---|---|
| DimSalesRep | FactSales | SalesRepID |
| DimGeography | FactSales | GeoKey |
| DimProducts | FactSales | ProductID |
| CalendarMaster | FactSales | Date |
| DimSubCategory | DimProducts | SubCategoryKey |
| DimCategory | DimSubCategory | CategoryKey |

SalesRep, Geography, Products, and the calendar relate directly to the fact, forming the star portion of the model. Category is intentionally not attached to the fact directly; instead it reaches the fact through a chain — `DimCategory` to `DimSubCategory` to `DimProducts` to `FactSales` — which forms the snowflake arm. This is precisely why the cleanup of the SubCategory key was critical: that key is the link that allows category-level filters to propagate all the way down to individual sales rows.

<details>
<summary><strong>View data model</strong></summary>

<br>

![Data Model](data-model/data_model.png)

</details>

---

## Time Intelligence: The Calendar Table

A dedicated date table, **CalendarMaster**, was created to drive all time-based analysis:

```DAX
CalendarMaster = CALENDAR(MIN(FactSales[Date]), MAX(FactSales[Date]))
```

The range is derived dynamically from the fact table's own earliest and latest dates. This keeps the calendar in step with the resilient folder design: if a new sales file extends the date range, the calendar expands automatically on the next refresh, with no manual edit required.

The table was then enriched with calculated columns to support both display and correct sorting:

- **Date parts:** Year, Month, MonthName, Quarter, Week No, Week DayName, Week Day
- **Display labels:** Month-Year (for example, "Jan 2014") and Quarter-Year (for example, "Q1 2014")
- **Sort helpers:** MonthYearSort and QuarterYearSort (for example, 201401), which ensure month and quarter labels appear in true chronological order on report axes rather than alphabetically

Marking this as the model's official date table is what allows the Month-over-Month and Quarter-over-Quarter measures to calculate correctly.

---

## Calculated Columns and Measures

The calculation layer is deliberately split between **calculated columns** (computed row by row, used for attributes and per-transaction values) and **measures** (aggregations evaluated in the context of the report). Expand the sections below for detail.

<details>
<summary><strong>Calculated columns</strong></summary>

<br>

**On FactSales** — built from product price and cost brought across with `RELATED`:

- **Total Revenue** — derived from the product retail price and units sold, net of the revenue discount on the transaction.
- **Total Cost** — derived from the product standard cost, the percentage of standard cost on the transaction, and units sold.
- **Gross Profit** — Total Revenue minus Total Cost, at the row level.

**On DimProducts:**

- **Price Range** — classifies each product into Low Value, Medium Value, or High Value based on its retail price. This sits on the product dimension because price tier is a product attribute, and it is what drives the price-range analysis page.

</details>

<details>
<summary><strong>Measures</strong></summary>

<br>

**Core aggregations:**

- **Total Gross Profit** — the sum of row-level gross profit; the figure behind the headline profit card.
- **Gross Profit Margin %** — Total Gross Profit divided by total revenue.
- **Revenue per Unit** — total revenue divided by total units; used in the per-unit comparison visuals.
- **Total Transactions** — a count of sales records by primary key.
- **Avg Discount** — the average revenue discount across transactions.

**Time-intelligence stack** — built as chained helper measures rather than one monolithic formula, which keeps the logic readable and reusable:

- **Prev Mth** and **Prev Qtr** — anchor the prior period over the calendar table.
- **Previous Month Gross Profit** — pulls the prior period value forward.
- **MoM Growth** and **QoQ Growth** — express the period-over-period change as a ratio of current to previous, powering the growth trend page.

</details>

---

## Outputs

The final report spans nine pages, moving from a high-level introduction through detailed drill-downs to a closing summary. Expand each page to view its screenshot.

<details>
<summary><strong>1. Introduction</strong> — project scope, business questions, and data coverage</summary>

<br>

![Introduction](outputs/01-introduction.png)

</details>

<details>
<summary><strong>2. Overall Stats</strong> — revenue, cost, and gross profit KPI cards</summary>

<br>

![Overall Stats](outputs/02-overall-stats.png)

</details>

<details>
<summary><strong>3. Revenue by Country/Year</strong> — revenue waterfall with country, year, and month slicers</summary>

<br>

![Revenue by Country/Year](outputs/03-revenue-by-country-year.png)

</details>

<details>
<summary><strong>4. MoM and QoQ Growth</strong> — Month-over-Month and Quarter-over-Quarter trend lines</summary>

<br>

![MoM and QoQ Growth](outputs/04-mom-qoq-growth.png)

</details>

<details>
<summary><strong>5. Top Sales Representatives</strong> — leading representatives by gross profit</summary>

<br>

![Top Sales Representatives](outputs/05-top-sales-reps.png)

</details>

<details>
<summary><strong>6. Country/Town-wise Performance</strong> — revenue, units, transactions, and profit by country</summary>

<br>

![Country/Town Performance](outputs/06-country-town-performance.png)

</details>

<details>
<summary><strong>7. Category / Sub-Category KPIs</strong> — performance split across the Special and General categories</summary>

<br>

![Category KPIs](outputs/07-category-kpi.png)

</details>

<details>
<summary><strong>8. Product Price Range KPIs</strong> — performance split across Low, Medium, and High Value tiers</summary>

<br>

![Price Range KPIs](outputs/08-price-range-kpi.png)

</details>

<details>
<summary><strong>9. Conclusion</strong> — strategic review and recommendations</summary>

<br>

![Conclusion](outputs/09-conclusion.png)

</details>

---

## Insights

The analysis surfaced four clear findings:

- **Germany is the anchor market.** It accounts for roughly 69 percent of global revenue (about $87M) and 2.79M units sold, far ahead of the Czech Republic and Denmark.
- **The Special category drives profitability.** It contributes 64.53 percent of overall gross profit (about $56.07M), compared with the General category's $30.81M.
- **Medium Value is the most effective price tier.** It generates 41.22 percent of gross profit (about $35.81M) and captures 45.54 percent of all units sold, making it the strongest combination of volume and margin.
- **Revenue is concentrated in top talent.** A small group of representatives, led by Ellen Woody (about $23.7M) and John White (about $18.8M), anchors the company's frontline profitability.

A fuller write-up of trends, results, and their business relevance is available in [`conclusion/`](conclusion/).

---

## Recommendations

- **Replicate Germany's playbook in smaller territories.** Germany's high-volume operating model is the clearest profit driver; applying its approach to the Czech Republic and Denmark represents the largest growth opportunity.
- **Prioritise the Medium Value, Special category segment.** Concentrating marketing spend and inventory on this combination targets the point of peak margin efficiency.
- **Use the growth trend lines to manage seasonality.** The Month-over-Month and Quarter-over-Quarter views make seasonal revenue swings visible, allowing inventory and staffing to be planned ahead of predictable peaks and troughs.
- **Recognise the concentration of revenue in key staff.** With profitability heavily reliant on a few representatives, it is worth strengthening retention of top performers and sharing their practices across the wider team.

---

## Repository Structure

```
Retail_Sales_Analysis_Power_BI/
├── README.md
├── business-questions/
│   └── Business_Questions.pdf
├── data/
│   ├── sales/
│   │   ├── Sales 2014.csv
│   │   ├── sales 2015.csv
│   │   ├── sales 2016.csv
│   │   └── sales 2017.csv
│   ├── Categories.xlsx
│   ├── Geography.xlsx
│   ├── Product.csv
│   ├── SalesRep.xlsx
│   └── SubCategories.xlsx
├── data-model/
│   └── data_model.png
├── report/
│   └── RetailSalesAnalysis.pbix
├── outputs/
│   ├── 01-introduction.png
│   ├── 02-overall-stats.png
│   ├── 03-revenue-by-country-year.png
│   ├── 04-mom-qoq-growth.png
│   ├── 05-top-sales-reps.png
│   ├── 06-country-town-performance.png
│   ├── 07-category-kpi.png
│   ├── 08-price-range-kpi.png
│   └── 09-conclusion.png
└── conclusion/
    └── Insights_and_Analysis.md
```

---

## Getting Started

1. Install [Power BI Desktop](https://powerbi.microsoft.com/desktop/) (free).
2. Clone the repository:
```bash
   git clone https://github.com/deadlineZeus/Retail_Sales_Analysis_Power_BI.git
```
3. Open `report/RetailSalesAnalysis.pbix` in Power BI Desktop.
4. The raw source files are provided under `data/` so the model can be inspected and refreshed. If you reproduce the pipeline, update the source paths under **Transform data, Data source settings** to point at your local copies, and provide your own PostgreSQL connection for the products table.

---

## Tools and Technologies

- **Microsoft Power BI** — data modelling, DAX, and report building
- **Power Query (M)** — extraction and transformation
- **DAX** — calculated columns, measures, and time intelligence
- **PostgreSQL** — source database for the product catalogue
- **Web data extraction** — for the geography reference table

---

*If you find this project useful, consider starring the repository.*
