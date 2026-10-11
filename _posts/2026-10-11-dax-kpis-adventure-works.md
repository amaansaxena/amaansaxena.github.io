---
layout: post
title: "Building KPIs and Custom DAX Measures in Power BI: An Adventure Works Walkthrough"
date: 2026-10-11
author: Amaan Saxena
permalink: /blog/dax-kpis-adventure-works/
tags: [power-bi, dax, data-visualization, adventure-works]
excerpt: "23 DAX measures, a way to validate them without trusting the visuals, and what six months of bicycle sales actually show."
---

*23 DAX measures, a way to validate them without trusting the visuals, and what six months of bicycle sales actually show.*

**By Amaan Saxena** | Power BI, DAX, Data Visualization

Most Power BI tutorials stop at dragging a column onto a chart. The interesting work starts when the number you need doesn't exist in your data yet. In this post I build a KPI layer on top of the Adventure Works sample dataset: 23 DAX measures, a validation routine that uses DAX queries instead of eyeballing cards, and a dashboard that ties it together.

Along the way I hit three things worth knowing about: a sales table with no sales amount, the difference between counting rows and counting orders, and how DAX behaves when a month has no previous month.

> **The short version:** Adventure Works sold $9.19M between January and June 2022 at a 42.3% profit margin. Bikes are 92% of revenue, mountain bikes lead, and the United States and Australia account for 60% of sales.

## The data

The CSV version of Adventure Works comes as separate files for sales, products, customers, territories and dates, so the first job is deciding which ones matter. I used seven and left out the returns file and the unpivot practice file.

| File | Role in the model |
| --- | --- |
| Sales Data 2022 | Fact table, 29,481 order lines |
| Product Lookup | Product name, cost and price |
| Product Subcategories Lookup | Subcategory for each product |
| Product Categories Lookup | Bikes, Components, Clothing, Accessories |
| Customer Lookup | Customer details |
| Territory Lookup | Region, country and continent |
| Calendar Lookup | Date table |

Power BI keeps the original file names as table names, such as `AdventureWorks Sales Data 2022`. To keep the formulas readable I use short names in this post: `Sales`, `Products`, `Categories`, `Territory` and `Calendar`. In the real file, every reference uses the full table name in single quotes.

Two quirks of this dataset shape everything that follows:

- **It covers six months, not a year.** Order dates in the 2022 sales file run from 1 January to 30 June 2022.
- **There is no SalesAmount column.** The sales file holds only the quantity ordered. Sales value has to be calculated as quantity multiplied by product price, and cost as quantity multiplied by product cost. That is why the core measures use `SUMX` rather than a plain `SUM`.

## Loading the data and building the model

1. In Power BI Desktop, use **Get Data > Text/CSV** for each file and click **Transform Data** instead of Load.
2. In Power Query, set `OrderDate` and `Calendar[Date]` to the Date type, and filter Calendar to 2022.
3. After **Close & Apply**, open Model view and create the relationships below, all many to one.
4. On the Calendar table, add Year, Month Num and Month columns, sort Month by Month Num, and mark Calendar as the date table using the Date column.

| From (many side) | To (one side) |
| --- | --- |
| Sales[OrderDate] | Calendar[Date] |
| Sales[ProductKey] | Products[ProductKey] |
| Sales[CustomerKey] | Customers[CustomerKey] |
| Sales[TerritoryKey] | Territory[SalesTerritoryKey] |
| Products[ProductSubcategoryKey] | Subcategories[ProductSubcategoryKey] |
| Subcategories[ProductCategoryKey] | Categories[ProductCategoryKey] |

The three calculated columns on the Calendar table:

```dax
Year      = YEAR(Calendar[Date])
Month Num = MONTH(Calendar[Date])
Month     = FORMAT(Calendar[Date], "MMM")
```

Sorting Month by Month Num matters: without it, charts order months alphabetically (Apr, Feb, Jan) instead of chronologically. Marking the date table is what lets the time intelligence functions later in the post work reliably.

## The DAX measures

I built 23 measures in three groups: core KPIs, counting, filtering and ranking, and time intelligence.

### Core KPIs

Because the sales file has no amount column, revenue and cost are computed row by row with `SUMX`, which walks each order line and uses `RELATED` to pull the price or cost from the Products table.

```dax
Total Sales = SUMX(Sales, Sales[OrderQuantity] * RELATED(Products[ProductPrice]))

Total Cost = SUMX(Sales, Sales[OrderQuantity] * RELATED(Products[ProductCost]))

Total Profit = [Total Sales] - [Total Cost]

Total Quantity = SUM(Sales[OrderQuantity])

Total Orders = DISTINCTCOUNT(Sales[OrderNumber])

Total Customers = DISTINCTCOUNT(Sales[CustomerKey])

Average Sales = AVERAGEX(Sales, Sales[OrderQuantity] * RELATED(Products[ProductPrice]))

Average Order Value = DIVIDE([Total Sales], [Total Orders])

Average Product Price = AVERAGE(Products[ProductPrice])

Profit Margin % = DIVIDE([Total Profit], [Total Sales]) * 100
```

Two details are easy to get wrong here. `Total Orders` uses `DISTINCTCOUNT` on the order number because one order can hold several lines. Counting rows would report 29,481 orders instead of the real 11,839. And ratios use `DIVIDE` rather than the `/` operator, so a zero or blank denominator returns blank instead of an error.

### Counting, filtering and ranking

```dax
Order Lines = COUNT(Sales[OrderLineItem])

Average Cost = AVERAGE(Products[ProductCost])

Bike Sales = CALCULATE([Total Sales], Categories[CategoryName] = "Bikes")

High Value Sales =
    CALCULATE(
        [Total Sales],
        FILTER(Sales, Sales[OrderQuantity] * RELATED(Products[ProductPrice]) > 1000)
    )

Product Rank  = RANKX(ALL(Products[ProductName]), [Total Sales], , DESC)

Category Rank = RANKX(ALL(Categories[CategoryName]), [Total Sales], , DESC)

Region Rank   = RANKX(ALL(Territory[Country]), [Total Sales], , DESC)

Sales % of Total = DIVIDE([Total Sales], CALCULATE([Total Sales], ALL(Sales)))
```

The ranking measures wrap the dimension column in `ALL`. That removes the filter on that column, so each row is ranked against every product (or category, or country) rather than only against itself, and keeps its rank when a slicer is applied.

### Time intelligence

```dax
Monthly Sales = TOTALMTD([Total Sales], Calendar[Date])

Previous Month Sales = CALCULATE([Total Sales], DATEADD(Calendar[Date], -1, MONTH))

Sales Variance = [Total Sales] - [Previous Month Sales]

Sales Growth % = DIVIDE([Sales Variance], [Previous Month Sales]) * 100

YTD Sales = TOTALYTD([Total Sales], Calendar[Date])
```

## Month-over-month analysis

Monthly sales rose in every month of the period.

| Month | Sales ($) | Previous ($) | Variance ($) | Growth % |
| --- | --- | --- | --- | --- |
| Jan 2022 | 1,274,379 | - | 1,274,379 | - |
| Feb 2022 | 1,339,241 | 1,274,379 | 64,863 | 5.09% |
| Mar 2022 | 1,448,596 | 1,339,241 | 109,355 | 8.17% |
| Apr 2022 | 1,527,814 | 1,448,596 | 79,218 | 5.47% |
| May 2022 | 1,768,433 | 1,527,814 | 240,619 | 15.75% |
| Jun 2022 | 1,826,987 | 1,768,433 | 58,555 | 3.31% |

Look at the January row. The data starts on 1 January 2022, so there is no previous month, and `Previous Month Sales` and `Sales Growth %` come back blank. `Sales Variance` is not blank, though: it equals the full January sales of 1,274,379, because DAX treats a blank as zero in subtraction. If you report variance, decide how you want to handle the first period.

I checked these figures with a DAX query that groups the measures by year and month and returns all four in one result.

```dax
EVALUATE
SUMMARIZECOLUMNS(
    'AdventureWorks Calendar Lookup'[Year],
    'AdventureWorks Calendar Lookup'[Month Num],
    'AdventureWorks Calendar Lookup'[Month],
    "Monthly Sales", [Total Sales],
    "Previous Month Sales", [Previous Month Sales],
    "Sales Variance", [Sales Variance],
    "Sales Growth %", [Sales Growth %]
)
ORDER BY
    'AdventureWorks Calendar Lookup'[Year],
    'AdventureWorks Calendar Lookup'[Month Num]
```

![Monthly sales, previous month sales, variance and growth percentage in the DAX query view]({{ "/assets/images/fig1-monthly-sales-dax-query.png" | relative_url }})

*Figure 1. DAX query for monthly sales, previous month sales, variance and growth percentage.*
{: .caption}

## Validating measures with DAX queries

A measure that returns a plausible number is not the same as a measure that returns the right one. Power BI Desktop has a built-in **DAX query view**, which makes checking cheap. Open it from the left bar, paste a query and click Run.

### Check 1: totals by category

This query returns sales, profit, margin and orders for each product category in one result.

```dax
EVALUATE
SUMMARIZECOLUMNS(
    'AdventureWorks Product Categories Lookup'[CategoryName],
    "Sales", [Total Sales],
    "Profit", [Total Profit],
    "Margin %", [Profit Margin %],
    "Orders", [Total Orders]
)
```

To validate, I added up the categories and compared the total to the dashboard cards. Sales come to $9,185,449 and profit to $3,889,029, identical to the Total Sales and Total Profit cards. Margins are 41.06% for Bikes, 44.28% for Clothing and 62.84% for Accessories.

![DAX query view showing sales, profit, margin and orders by product category]({{ "/assets/images/fig2-category-query.png" | relative_url }})

*Figure 2. DAX query view in Power BI Desktop: the query and its results by product category.*
{: .caption}

### Check 2: every core KPI in one row

A second query returns all the core KPIs as a single row, so they can be compared with the cards side by side.

```dax
EVALUATE
ROW(
    "Total Sales", [Total Sales],
    "Total Cost", [Total Cost],
    "Total Profit", [Total Profit],
    "Profit Margin %", [Profit Margin %],
    "Total Orders", [Total Orders],
    "Total Customers", [Total Customers],
    "Average Order Value", [Average Order Value]
)
```

The results are 9,185,449.45 in sales, 5,296,420.47 in cost, 3,889,028.97 in profit, a 42.34 margin, 11,839 orders, 10,502 customers and an average order value of 775.86. All of them match the cards.

![All core KPI measures returned as a single row in the DAX query view]({{ "/assets/images/fig3-core-kpis-single-row.png" | relative_url }})

*Figure 3. All core KPI measures returned in a single row.*
{: .caption}

### Check 3: top 10 products with RANKX

This query keeps the ten best-selling products and shows the rank from the `Product Rank` measure.

```dax
EVALUATE
TOPN(
    10,
    SUMMARIZECOLUMNS(
        'AdventureWorks Product Lookup'[ProductName],
        "Sales", [Total Sales],
        "Profit", [Total Profit],
        "Rank", [Product Rank]
    ),
    [Sales], DESC
)
ORDER BY [Sales] DESC
```

The six Mountain-200 variants take ranks 1 to 6, led by the Black, 38 at 514,323.65 in sales and 236,765.34 in profit.

![Top 10 products by sales with their RANKX rank]({{ "/assets/images/fig4-top-10-products.png" | relative_url }})

*Figure 4. Top 10 products by sales with their rank.*
{: .caption}

### Check 4: countries ranked by sales

```dax
EVALUATE
SUMMARIZECOLUMNS(
    'AdventureWorks Territory Lookup'[Country],
    "Sales", [Total Sales],
    "Profit", [Total Profit],
    "Rank", [Region Rank]
)
ORDER BY [Sales] DESC
```

The `Region Rank` measure ranks the six countries. The United States is first with 3,130,759 in sales, Australia second with 2,408,105, and Canada sixth with 638,086.

![Sales, profit and rank for each country]({{ "/assets/images/fig5-countries-ranked.png" | relative_url }})

*Figure 5. Sales, profit and rank for each country.*
{: .caption}

## Query performance

To see whether any of this was slow, I opened **View > Performance analyzer** in Report view, clicked **Start recording** and then **Refresh visuals**.

Every visual on the dashboard finished in about a third of a second or less. The slowest was one KPI card at 325 ms. The other five cards took between 167 and 185 ms, the line chart took 151 ms, the product bar chart 150 ms and the category chart 149 ms. The slicers took between 145 and 189 ms, and the map was the fastest at 112 ms.

At roughly 29,000 order lines, none of the measures needed tuning. `SUMX` over a fact table this size is cheap. The picture would change with millions of rows, and that is when the Performance analyzer becomes worth opening regularly.

![Performance analyzer showing the duration of each visual in milliseconds]({{ "/assets/images/fig6-performance-analyzer.png" | relative_url }})

*Figure 6. Performance analyzer showing the duration of each visual in milliseconds.*
{: .caption}

## The dashboard

The report page has a row of six KPI cards on top, four charts underneath, and three slicers.

- **Cards:** Total Sales, Total Profit, Total Orders, Total Customers, Average Order Value and Profit Margin %.
- **Line chart:** `Calendar[Month]` against Total Sales and Previous Month Sales.
- **Bar chart:** `Products[ProductName]` against Total Sales, filtered to the Top 10.
- **Column chart:** `Categories[CategoryName]` against Total Sales and Total Profit.
- **Map or bar chart:** `Territory[Country]` against Total Sales.
- **Slicers:** `Calendar[Year]`, `Categories[CategoryName]` and `Territory[Country]`. I also added a Month slicer, since Year only has one value (2022).

Clicking any slicer or chart element cross-filters the rest of the page, which was the main thing to test.

![The finished Power BI dashboard with KPI cards, charts and slicers]({{ "/assets/images/fig7-dashboard.png" | relative_url }})

*Figure 7. Dashboard with KPI cards, charts and slicers. The visuals come from the Power BI report and are rearranged and recoloured for this post.*
{: .caption}

## What the numbers say

### Headline KPIs

| KPI | Value |
| --- | --- |
| Total Sales | $9,185,449 |
| Total Cost | $5,296,420 |
| Total Profit | $3,889,029 |
| Profit Margin % | 42.34% |
| Total Quantity | 45,314 units |
| Total Orders | 11,839 |
| Total Customers | 10,502 |
| Average Order Value | $775.86 |
| Average Sales (per order line) | $311.57 |
| Average Product Price | $714.44 |

Average Product Price averages the price list across all 293 products, including ones that did not sell, so it sits well above the average sales per line.

### Categories

| Category | Sales ($) | Profit ($) | Margin % | Units sold |
| --- | --- | --- | --- | --- |
| Bikes | 8,468,855 | 3,477,545 | 41.06% | 5,689 |
| Accessories | 507,331 | 318,824 | 62.84% | 32,493 |
| Clothing | 209,264 | 92,660 | 44.28% | 7,132 |

No Components products were sold in this period, so the category does not appear.

### Top-selling products

| Rank | Product | Sales ($) | Units |
| --- | --- | --- | --- |
| 1 | Mountain-200 Black, 38 | 514,324 | 251 |
| 2 | Mountain-200 Silver, 38 | 513,712 | 248 |
| 3 | Mountain-200 Silver, 46 | 505,426 | 244 |
| 4 | Mountain-200 Black, 42 | 504,078 | 246 |
| 5 | Mountain-200 Silver, 42 | 466,069 | 225 |
| 6 | Mountain-200 Black, 46 | 465,145 | 227 |
| 7 | Touring-1000 Blue, 46 | 281,320 | 118 |
| 8 | Touring-1000 Yellow, 46 | 264,632 | 111 |
| 9 | Road-350-W Yellow, 40 | 261,952 | 154 |
| 10 | Road-350-W Yellow, 42 | 258,550 | 152 |

### Countries

| Country | Sales ($) | Profit ($) | Margin % |
| --- | --- | --- | --- |
| United States | 3,130,759 | 1,339,363 | 42.78% |
| Australia | 2,408,105 | 995,203 | 41.33% |
| United Kingdom | 1,119,088 | 472,552 | 42.23% |
| Germany | 1,015,657 | 430,885 | 42.42% |
| France | 873,754 | 370,049 | 42.35% |
| Canada | 638,086 | 280,977 | 44.03% |

### Key insights

- **Bikes drive the business.** They bring in 92.2% of sales and about 89% of profit, from only 5,689 units.
- **One product family dominates.** Mountain bikes lead the bike subcategories at $3.38M, ahead of road bikes at $2.72M and touring bikes at $2.38M. The six Mountain-200 variants take the top six product spots and together account for roughly 32% of all sales.
- **Accessories are volume, not revenue.** They sell the most units (71.7% of everything shipped) but contribute only 5.5% of sales. They also have the best margin at 62.8%, against 41.1% for bikes.
- **Two countries carry the geography.** The United States is the largest at $3.13M (34% of sales), followed by Australia at $2.41M (26%). Australia is the top single region. Within the US, almost all sales come from Southwest ($1.91M) and Northwest ($1.21M). Southeast, Northeast and Central together add under $11,000.
- **Growth was steady.** Sales climbed from $1.27M in January to $1.83M in June, a rise of about 43%. May had the biggest jump at 15.75% over April, and June grew by 3.31%, the slowest of the five month-on-month changes.
- **Margin is flat across countries.** It stays between 41% and 44% everywhere, so differences in country profit come from sales volume, not pricing or cost.
- **High-value lines matter.** Order lines worth over $1,000 make up about 80% of total sales, from 3,936 of the 29,481 lines.

## Takeaways

- **Derive missing columns in measures, not in the source.** When a table has no amount column, `SUMX` with `RELATED` gives you revenue at the right grain without touching the data.
- **Count the thing you mean.** `DISTINCTCOUNT` on the order number gives 11,839 orders. Counting rows gives 29,481, which is order lines.
- **Validate with queries, not with the visuals you built from the same measures.** `SUMMARIZECOLUMNS` and `ROW` in DAX query view let you check totals independently of the report canvas.
- **Wrap ranking dimensions in `ALL`.** Otherwise a slicer changes what each row is ranked against.
- **Mark your date table and sort your month names.** Time intelligence and chart ordering both depend on it.
- **Decide how to handle the first period.** Blank previous-month values turn into zero in subtraction, so the January variance equals the whole month.

## Limitations

The data covers six months of 2022 only, so there is no year-over-year comparison and January has no previous-month figures. Sales and cost are derived from quantity and list prices because the file has no sales amount column, so discounts and price changes over time are not reflected.

Two things look worth a closer look in the business itself: how much revenue leans on one product family, and the near-zero sales in three US regions. Accessories are the obvious place to look for margin, given their volume and 62.8% margin.

---

*Amaan Saxena is a B.Tech AI & ML student at Symbiosis Institute of Technology, Pune.*
