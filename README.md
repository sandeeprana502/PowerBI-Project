# Olist E-Commerce Analytics — Power BI End-to-End Project

Part of the [Sandeep Rana](https://www.youtube.com/@SandeepSinghRana502/) YouTube series — *Beyond Basic Dashboards.*

A production-style Power BI project built on the public [Olist Brazilian E-Commerce dataset](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce). The focus is on **correct grain, a clean star schema, and DAX-driven logic** (no calculated columns in fact tables, minimal Power Query transformation).

## Dashboard pages

| Page | What it answers |
|---|---|
| **Sales Overview** | Gross sales, freight %, total orders, unique customers, revenue trend, payment mix, top 10 categories |
| **Delivery Overview** | Average review score, late delivery %, on-time %, delivered orders, actual vs estimated delivery time |

Interactive features: dynamic title, Actual / K / M unit conversion, Monthly vs YTD toggle, page navigation buttons, clear-all-slicers bookmark.

![Sales Overview](Assets/sales_overview.png)
![Delivery Overview](Assets/delivery_overview.png)

## Data model

Star schema, single-direction filtering from dimensions to facts, no fact-to-fact relationships.

| Table | Type | Grain |
|---|---|---|
| `FACT_Orders` | Fact | One row per order |
| `FACT_Order_Items` | Fact | One row per order item |
| `FACT_Payment` | Fact | One row per payment record |
| `FACT_Reviews` | Fact | One row per review |
| `DIM_Customer` | Dimension | One row per `customer_id` (`customer_unique_id` identifies a repeat customer) |
| `DIM_Product` | Dimension | One row per product |
| `DIM_Seller` | Dimension | One row per seller |
| `DIM_Date` | Dimension | Continuous calendar |
| `_Measures` | Measure table | DAX measures only |

**Relationships (8)**

- `FACT_Order_Items` → `FACT_Orders` (order_id), `DIM_Product` (product_id), `DIM_Seller` (seller_id)
- `FACT_Payment` → `FACT_Orders` (order_id)
- `FACT_Reviews` → `FACT_Orders` (order_id)
- `FACT_Orders` → `DIM_Customer` (customer_id)
- `FACT_Orders` → `DIM_Date` on `order_purchase_timestamp` (active)
- `FACT_Orders` → `DIM_Date` on `order_delivered_customer_date` (inactive, used via `USERELATIONSHIP`)

## Metric definitions

| Metric | Definition |
|---|---|
| Merchandise Revenue | Sum of item `price` |
| Gross Sales | Merchandise Revenue + Freight Value |
| Average Order Value | Merchandise Revenue ÷ Total Orders (item price only, order-level denominator) |
| Late Delivery % | Delivered orders where delivered date > estimated date ÷ Delivered Orders |
| On-Time % | 1 − Late Delivery % |
| Unique Customers | Distinct `customer_unique_id` |

## Key DAX

```dax
Total Orders = DISTINCTCOUNT ( FACT_Orders[order_id] )

Merchandise Revenue = SUM ( FACT_Order_Items[price] )

Gross Sales = [Merchandise Revenue] + [Freight Value]

Delivered Orders =
CALCULATE ( [Total Orders], FACT_Orders[order_status] = "delivered" )

Late Orders =
COUNTROWS (
    FILTER (
        FACT_Orders,
        NOT ISBLANK ( FACT_Orders[order_delivered_customer_date] )
            && FACT_Orders[order_delivered_customer_date] > FACT_Orders[order_estimated_delivery_date]
    )
)

Late Delivery % = DIVIDE ( [Late Orders], [Delivered Orders] )

Avg Delivery Minutes =
AVERAGEX (
    FILTER ( FACT_Orders, NOT ISBLANK ( FACT_Orders[order_delivered_customer_date] ) ),
    DATEDIFF ( FACT_Orders[order_purchase_timestamp], FACT_Orders[order_delivered_customer_date], MINUTE )
)

Revenue PY =
CALCULATE ( [Merchandise Revenue], SAMEPERIODLASTYEAR ( DIM_Date[Date] ) )

Revenue by Delivery Date =
CALCULATE (
    [Merchandise Revenue],
    USERELATIONSHIP ( DIM_Date[Date], FACT_Orders[order_delivered_customer_date] )
)

Category Rank =
RANKX ( ALLSELECTED ( DIM_Product[Product_Category_English_Name] ), [Merchandise Revenue] )

Unique Customers =
CALCULATE (
    DISTINCTCOUNT ( DIM_Customer[customer_unique_id] ),
    CROSSFILTER ( FACT_Orders[customer_id], DIM_Customer[customer_id], BOTH )
)

Dynamic Title = "Sales/Revenue for " & SELECTEDVALUE ( DIM_Date[Year], "All Years" )
```

## Design decisions

- **No calculated columns in fact tables.** Row-level logic lives in measures. Optionally, heavy logic can later be materialised in a separate aggregate fact table as an optimisation step.
- **Minimal Power Query.** Source CSVs stay untouched in `Dataset/`.
- **Respect grain.** Facts are never merged into one flat table, which avoids silent revenue multiplication.
- **Role-playing date.** Purchase date is the active relationship; delivery date is inactive and activated per measure.

## Repository structure

The project is saved in **Power BI Project (PBIP)** format, so the model and report are stored as readable text files and work well with Git.

```text
.
├── README.md
├── Olist_Project.pbip            # open this in Power BI Desktop
├── Olist_Project.SemanticModel/  # model definition (TMDL): tables, relationships, measures
├── Olist_Project.Report/         # report definition (pages, visuals)
├── Dataset/                      # original Olist CSVs, unchanged
└── Assets/                       # dashboard screenshots and thumbnail
```

## Getting started

1. Clone or download this repository.
2. Download the dataset from [Kaggle](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce), review its licence, and place the CSV files in `Dataset/` (skip if already included).
3. Open `Olist_Project.pbip` in Power BI Desktop. If PBIP is not enabled, turn on **Power BI Project (.pbip) save option** under File → Options → Preview features.
4. Update the source folder path in **Transform data → Data source settings** to point to your local `Dataset/` folder.
5. Refresh.

## Tools

Power BI Desktop · DAX · Power Query · Power BI Modeling MCP Server · Claude Desktop

## Series

Full tutorial series on the [Sandeep Rana](https://www.youtube.com/@SandeepSinghRana502/) YouTube channel.

## Author

**Sandeep** — Power BI Developer / Analytics Engineer. 
**Blogpost** link is: [Power BI Comprehensive Guide](https://sandeeprana502.blogspot.com/2026/11/microsoft-power-bi-comprehensive-guide.html)

## Licence

Code and DAX: All code and dax are written by me. Dataset: see the Kaggle page for Olist licence terms.
