# Food retail Power BI report: dataset audit and build guide

Prepared from `food_realistic_dataset.zip`, inspected in full on September 25, 2026. The source archive was not modified. This guide is an implementation specification, not an already built or tested PBIX file.

## 1. Decision before building

Build a **six-page demonstration report**, with one hidden order drillthrough page and one hidden tooltip page: **eight pages in the PBIX, six in normal navigation**. The page count is a design recommendation, not a Power BI requirement.

Use a visible subtitle: **Demonstration dataset · January 1–28, 2025 · Unreconciled amounts**. Do not present these figures as company performance. The dataset contains extensive placeholder content and cross-table contradictions. Corrected source data is required for a production report.

Do not create one report page for each CSV. Organize pages around decisions. Do not invent targets, currency, historical costs, discounts, ratings or valid statuses to fill gaps.

## 2. What the audit found

All 15 CSVs contain 1,200 rows and 10 columns: 18,000 rows and 150 source columns in total. There are no blank cells, exact duplicate rows or duplicate primary IDs in the inspected files. The 18 tested foreign-key relationships have zero missing parent keys. However, every tested foreign-key column also has 1,200 distinct values, and each row's ID fields equal that row's primary ID. This produces an artificial one-to-one pattern rather than realistic repeat business activity.

All date-shaped fields parse as dates and range from January 1 through January 28, 2025. Inventory.LastUpdated is not a date: it contains names. Only one partial month is available. Annual, quarterly, year-over-year and month-over-month performance comparisons are unsupported.

| Check | Result | Meaning |
|---|---:|---|
| Order header amount differs from Quantity × line UnitPrice | 1,200 / 1,200 | No order reconciles to its gross line calculation; valid discounts/taxes are unavailable |
| Payment amount differs from order header amount | 1,200 / 1,200 | Investigate settlement rules; differences alone need not be errors, but cannot be reconciled here |
| Return quantity exceeds quantity sold on the matching order/product | 589 / 1,200 | Return quantities need correction or explanation |
| Return date precedes order date | 577 / 1,200 | Invalid chronology for the stated transaction relationship |
| Payment date precedes order date | 570 / 1,200 | Requires explanation; prepayments are possible |
| Shipment arrival precedes shipment date | 596 / 1,200 | Invalid transit interval |
| Promotion end precedes start | 571 / 1,200 | Invalid promotion interval |
| Order precedes store opening | 563 / 1,200 | Requires source investigation |
| Order precedes employee hiring | 576 / 1,200 | Requires source investigation |
| Product cost exceeds product list price | 597 / 1,200 | Exception to investigate, not an automatic correction |
| Line unit price differs from product list price | 1,200 / 1,200 | Transaction and list prices differ; not inherently an error |
| Line expiry date precedes order date | 556 / 1,200 | Potential expired sale or incorrect dates |
| Products.Category matches Categories.CategoryName | 0 / 1,200 | Categories.csv cannot be used as the product category lookup |

Every value in such fields as Orders.Tax, Orders.Discount, OrderDetails.LineTotal, Reviews.Rating, Waste.CostLoss and Inventory.ReorderLevel fails numeric interpretation. Nonblank does not mean valid.

There are 1,200 ProductIDs but only 10 distinct product names and 6 category labels. Use ProductID in entity-level displays. Do not deduplicate different IDs just because their names match. Product/category assignments also require a business mapping review: for example, the first source row describes Olive Oil as Dairy.

### Raw control totals

These are arithmetic controls for checking your import, **not validated KPIs**. Amounts are in unspecified source units; currency is not established. Totals include all statuses.

| Control | Expected unfiltered value |
|---|---:|
| Orders | 1,200 |
| Order detail rows | 1,200 |
| Sum Orders.TotalAmount | 3,056,468.89 |
| Sum OrderDetails.Quantity | 122,657 |
| Sum Quantity × OrderDetails.UnitPrice | 310,224,418.29 |
| Sum Payments.Amount | 3,073,344.95 |
| Sum Returns.Qty | 121,783 |
| Sum Returns.RefundAmount | 3,018,159.67 |
| Sum Inventory.StockQty | 119,375 |
| Sum Waste.Qty | 122,705 |

The gross line calculation is about 101.5 times the order header amount. Do not choose one and silently rename it Revenue. Counts, sums and ratios may be mathematically computable while their business meaning remains unreliable.

## 3. Business definitions to establish

Before production, record the report owner, intended users, refresh schedule, currency, transaction timezone, quantity units, meaning of each status, and the exact sales recognition policy. Establish whether Discount and Tax are amounts or rates, how returns affect sales and costs, and whether payments are settlements, attempts, partial payments or refunds. Identify the authoritative financial total.

For this demonstration, use explicit names such as **Recorded Order Amount**, **Calculated Gross Line Value**, **Recorded Payment Amount** and **Recorded Return Units**. All include all source records unless a visual explicitly states otherwise. Do not call any of these net revenue, collected cash or profit.

## 4. Import and staging: follow this order

1. Extract the ZIP to a dedicated source folder. Preserve an unchanged copy of the archive.
2. Open Power BI Desktop and save a new PBIX. Use Import storage for this small CSV dataset.
3. In Power Query, create a Text parameter `pDataFolder` containing the extracted folder path, without a trailing separator.
4. Import each CSV separately using Get Data > Text/CSV > Transform Data, or use the helper function below. The 15 files have different schemas: do not Combine Files as one table.
5. Name the raw queries `stg_Products`, `stg_Orders`, etc. Use the same pattern for all 15 files. Disable Enable load on these staging queries.
6. Initially preserve all source values as text. Remove premature automatic Changed Type steps that create errors before you can inspect them.
7. Turn on Column quality, Column distribution and Column profile. Change profiling from the default first 1,000 rows to the **entire dataset**, since each file has 1,200 rows. [Microsoft: profiling tools](https://learn.microsoft.com/en-us/power-query/data-profiling-tools)
8. Create reference queries from staging for cleaning, then model tables from the cleaned queries. Disable load for intermediate cleaning queries too.
9. Preserve key IDs, source row number or primary ID, original problematic values and validation flags in audit queries. Only validated or clearly labeled demonstration fields should be exposed to report authors.
10. Create a refresh timestamp separately from the source data date. A September 2026 refresh does not make January 2025 transactions current.

### Reusable CSV loader (Power Query M)

Create a Blank Query, open Advanced Editor, paste this and name it `fxReadCsv`:

```powerquery
(FileName as text) as table =>
let
    Source = Csv.Document(
        File.Contents(pDataFolder & "/" & FileName),
        [Delimiter=",", Encoding=65001, QuoteStyle=QuoteStyle.Csv]
    ),
    Headers = Table.PromoteHeaders(Source, [PromoteAllScalars=true]),
    TextValues = Table.TransformColumnTypes(
        Headers,
        List.Transform(Table.ColumnNames(Headers), each {_, type text})
    )
in
    TextValues
```

Each staging query then uses, for example, `= fxReadCsv("Orders.csv")`.

## 5. Cleaning rules and transformations

Apply Text.Trim and Text.Clean to descriptive fields. Normalize case using a mapping where business labels differ; do not change identifiers or free-form text unnecessarily. Keep raw values available for audit. Set IDs consistently to Whole Number in this dataset, dates to Date, quantities to Whole Number, and monetary fields to Fixed Decimal Number. Use Decimal Number for rates only once their units are confirmed.

Parse dates explicitly: the source uses variable-width dates such as `2025-01-9`. Splitting year, month and day avoids locale ambiguity. Do not replace conversion errors with zero. Create a parsed nullable value and a separate invalid flag, then expose the issue count. Preserve whole records when only a nonessential attribute is invalid.

| Source | Usable for a labeled demonstration | Required corrections / exclusions |
|---|---|---|
| Products | ProductID, ProductName, Category, Brand, UnitPrice, CostPrice, SupplierID | ShelfLifeDays and Calories contain names; review product/category and brand mappings, costs and Status vocabulary |
| Suppliers | SupplierID, SupplierName; Country and City as unvalidated labels | Phone, Email, Rating, LeadTimeDays and PaymentTerms contain names; validate country/city pairs and Status |
| Customers | CustomerID, CustomerName; City as an unvalidated label | State is not a dependable state field; Phone, Email, CustomerType, LoyaltyPoints, Gender and Age are invalid for their intended uses |
| Orders | OrderID, OrderDate, CustomerID, StoreID, TotalAmount, PaymentMode, EmployeeID | Tax, Discount and OrderStatus contain names; amount reconciliation and chronology need investigation |
| OrderDetails | OrderDetailID, OrderID, ProductID, Quantity, UnitPrice, ExpiryDate | Discount, Tax, LineTotal and BatchNo are unusable as intended; calculate a separately named gross value |
| Inventory | InventoryID, ProductID, StoreID, StockQty, ReorderQty, ExpiryDate | ReorderLevel, LastUpdated, Warehouse and BatchNo contain names; no reliable snapshot date or reorder trigger |
| Payments | PaymentID, OrderID, PaymentDate, Amount, TransactionID; raw Status label | Mode, Bank, Currency and Remarks contain names; confirm settlement status and amounts |
| Stores | StoreID; City as an unvalidated label; OpenDate subject to chronology checks | StoreName is a person-name placeholder; State, StoreType, Phone and AreaSqFt need correction; confirm Manager and Status |
| Employees | EmployeeID, EmployeeName, StoreID, HireDate | Role, Department, Phone and Email contain names; Salary has no period/currency definition; review Status |
| Categories | CategoryID and CreatedDate only for audit | CategoryName contains person names; Department, Description, TaxRate, MarginPercent and other content are not dependable category attributes |
| Shipments | ShipmentID, SupplierID, StoreID, ShipDate, ArrivalDate; raw Status | FreightCost is nonnumeric; Carrier and TrackingNo are names; confirm ReceivedBy; flag negative transit intervals |
| Returns | ReturnID, OrderID, ProductID, ReturnDate, Qty, RefundAmount; raw Status | Reason and Remarks are names; validate ApprovedBy, quantity, date and refund rules |
| Promotions | PromoID, Category, StartDate, EndDate; raw Status | PromoName, DiscountPercent, Channel and Budget are placeholders; no promotion-to-sale attribution |
| Reviews | ReviewID, CustomerID, ProductID, ReviewDate | Rating, ReviewText, Platform, Verified and Likes contain names; cannot analyze satisfaction or verified reviews |
| Waste | WasteID, ProductID, StoreID, WasteDate, Qty; raw Status | Reason and CostLoss are invalid; confirm HandledBy and what each status means |

Dates and numbers being parseable is not evidence that they describe real events. Retain these limitations in the report's methodology page.

### Example: parse invalid tax without hiding the error

Create `fxDateYMD`:

```powerquery
(Value as nullable text) as nullable date =>
let
    Parts = if Value = null then {} else Text.Split(Text.Trim(Value), "-"),
    Parsed = try #date(
        Number.FromText(Parts{0}, "en-US"),
        Number.FromText(Parts{1}, "en-US"),
        Number.FromText(Parts{2}, "en-US")
    ) otherwise null
in
    Parsed
```

An Orders cleaning pattern is:

```powerquery
let
    Source = stg_Orders,
    Trimmed = Table.TransformColumns(Source,
        List.Transform(Table.ColumnNames(Source),
            each {_, (v) => if v = null then null else Text.Trim(Text.Clean(v)), type text})),
    PreserveRaw = Table.RenameColumns(Trimmed,
        {{"Tax", "TaxRaw"}, {"OrderDate", "OrderDateRaw"}}),
    Typed = Table.TransformColumnTypes(PreserveRaw,
        {{"OrderID", Int64.Type}, {"CustomerID", Int64.Type},
         {"StoreID", Int64.Type}, {"EmployeeID", Int64.Type},
         {"TotalAmount", Currency.Type}}, "en-US"),
    AddDate = Table.AddColumn(Typed, "OrderDate",
        each fxDateYMD([OrderDateRaw]), type date),
    AddTax = Table.AddColumn(AddDate, "TaxParsed",
        each try Number.FromText([TaxRaw], "en-US") otherwise null,
        type nullable number),
    TaxFlag = Table.AddColumn(AddTax, "TaxInvalid",
        each [TaxParsed] = null and [TaxRaw] <> null and [TaxRaw] <> "", type logical),
    DateFlag = Table.AddColumn(TaxFlag, "OrderDateInvalid",
        each [OrderDate] = null, type logical)
in
    DateFlag
```

This example trusts only the ID and TotalAmount conversions already verified in this extract. Apply the same try/flag pattern to them when handling future unvalidated deliveries. A column with 100% invalid values must remain unavailable until corrected. Never infer tax or discount from the gap between contradictory totals.

### Exception queries

Create an audit table with `SourceTable`, `RecordID`, `Field`, `Rule`, `RawValue` and `Severity`. One row represents one failed rule, so one source record may create multiple issue rows. Keep issue count distinct from affected-record count. A record key for cross-table counting must include SourceTable plus RecordID.

Check primary key uniqueness, missing keys with Left Anti joins, numeric ranges, invalid dates, return quantities, date chronology and reconciliation differences. Separate definite failures from business exceptions: negative shipment duration is a failure, but a below-cost sale or advance payment requires context.

Do not remove duplicates on ProductName, CustomerName or StoreName. There are no duplicate primary keys now. In future deliveries, conflicting rows with the same primary key require a documented resolution, not arbitrary Remove Duplicates.

## 6. Model design

Use several fact tables sharing dimensions. A dimension describes who/what/where; a fact records an event or balance. Keep a consistent grain in each fact. This implements Microsoft's star-schema principles. [Microsoft: star schema](https://learn.microsoft.com/en-us/power-bi/guidance/star-schema)

### Dimensions

| Model table | Source / construction | Unique key |
|---|---|---|
| DimDate | Continuous calendar, full 2025 for this extract | Date |
| DimProduct | Clean Products; retain Category as an attribute | ProductID |
| DimCustomer | Clean Customers; omit invalid demographic/contact fields | CustomerID |
| DimStore | Clean Stores; use a display label such as Store 0001 until names are corrected | StoreID |
| DimEmployee | Clean Employees, minimal relevant attributes | EmployeeID |
| DimSupplier | Clean Suppliers, minimal relevant attributes | SupplierID |
| DimOrder | Reference clean Orders and retain distinct OrderID only | OrderID |

Add ProductLabel = ProductName & " (" & ProductID & ")" for entity-level charts. Do not manufacture business store names. Keep invalid source names in audit only. Do not relate Categories.csv to Products by row index or equal numeric IDs: Products has no CategoryID, and its category labels do not match CategoryName.

### Fact tables and exact grain

| Table | Grain | Construction |
|---|---|---|
| FactOrders | One row per OrderID | Clean Orders; owns recorded header TotalAmount |
| FactSales | One row per OrderDetailID | OrderDetails left-joined to selected Orders attributes on OrderID |
| FactPayments | One row per PaymentID | Payments left-joined to Orders for CustomerID and StoreID |
| FactReturns | One row per ReturnID | Returns left-joined to Orders for CustomerID, StoreID and original OrderDate |
| FactInventory | One supplied inventory record per InventoryID | Preserve StockQty and ExpiryDate; as-of date is unknown |
| FactWaste | One row per WasteID | Clean Waste, retaining recorded quantity |
| FactShipments | One row per ShipmentID | Clean Shipments; calculate duration only for valid intervals |

For FactSales, expand only OrderDate, CustomerID, StoreID, EmployeeID and PaymentMode from Orders. Do **not** expand TotalAmount as a summable line value. One order can have many lines in realistic data, which would duplicate the header total.

Optionally merge Products.SupplierID into FactSales, FactInventory, FactReturns and FactWaste. Label it **Current product supplier**, because the product master does not prove the historical supplier for a transaction. Supplier shipment reporting uses the actual Shipments.SupplierID directly.

Do not merge Payments and Returns directly into sales lines: multiple child rows would multiply values. Build reconciliation separately by grouping each source to OrderID first, then merging those grouped tables into a one-row-per-order QA table.

For return validation, aggregate sold Quantity by OrderID + ProductID and aggregate returned Qty at that same grain before comparing. The current data has one row per order, but this method also works when future orders have multiple lines or multiple returns.

Reviews and Promotions should remain staging/audit queries for this first release. Add their business pages only after the missing content is repaired. Employees is a dimension, not a payroll fact; do not sum Salary and call it a monthly payroll expense.

### Relationships to create manually

Every relationship below is **one-to-many (1:*)**, **single direction from dimension to fact**, and active unless otherwise stated. Set these deliberately even though this sample's foreign keys are currently unique. Disable or remove inappropriate automatically detected relationships.

| Dimension key | Fact columns |
|---|---|
| DimDate.Date | FactOrders.OrderDate; FactSales.OrderDate; FactPayments.PaymentDate; FactReturns.ReturnDate; FactWaste.WasteDate; FactShipments.ShipDate |
| DimProduct.ProductID | FactSales.ProductID; FactReturns.ProductID; FactInventory.ProductID; FactWaste.ProductID |
| DimCustomer.CustomerID | FactOrders.CustomerID; FactSales.CustomerID; FactPayments.CustomerID; FactReturns.CustomerID |
| DimStore.StoreID | FactOrders.StoreID; FactSales.StoreID; FactPayments.StoreID; FactReturns.StoreID; FactInventory.StoreID; FactWaste.StoreID; FactShipments.StoreID |
| DimEmployee.EmployeeID | FactOrders.EmployeeID; FactSales.EmployeeID |
| DimSupplier.SupplierID | FactShipments.SupplierID; optional enriched SupplierID columns described above |
| DimOrder.OrderID | FactOrders.OrderID; FactSales.OrderID; FactPayments.OrderID; FactReturns.OrderID |

Do not additionally relate DimStore to DimEmployee or DimSupplier to DimProduct when those dimensions already independently filter the same facts. Do not relate fact tables to each other. This avoids competing filter paths.

DimOrder exists so the hidden order detail page can use the same OrderID filter across lines, headers, payments and returns. Main product and category visuals should use measures from FactSales. A Product filter does not filter FactOrders or FactPayments in this design; do not place their unfiltered measures in product-level matrices.

Leave inventory unrelated to DimDate while its as-of date is unknown. ExpiryDate is **not** a snapshot date. If expiry-date filtering is needed later, use a separate DimExpiryDate and label it clearly. Never show stock balances as daily sales activity.

For arrival-date reporting, add an inactive DimDate.Date → FactShipments.ArrivalDate relationship and use USERELATIONSHIP in a dedicated measure. If simultaneous ship-date and arrival-date slicers are needed, use separate role-playing date dimensions. [Microsoft: date relationship roles](https://learn.microsoft.com/en-us/power-bi/guidance/relationships-active-inactive)

### Date table

Create with Modeling > New table:

```dax
DimDate =
ADDCOLUMNS (
    CALENDAR ( DATE ( 2025, 1, 1 ), DATE ( 2025, 12, 31 ) ),
    "Year", YEAR ( [Date] ),
    "MonthNo", MONTH ( [Date] ),
    "Month", FORMAT ( [Date], "MMM" ),
    "YearMonth", FORMAT ( [Date], "yyyy-MM" ),
    "YearMonthSort", YEAR ( [Date] ) * 100 + MONTH ( [Date] ),
    "DayOfWeekNo", WEEKDAY ( [Date], 2 ),
    "DayOfWeek", FORMAT ( [Date], "ddd" )
)
```

Mark DimDate as the date table using Date. Sort Month by MonthNo, YearMonth by YearMonthSort, and DayOfWeek by DayOfWeekNo. Disable Auto date/time for this file. Restrict initial report selection to the actual January 1–28 coverage; the rest of the calendar does not imply zero business activity. Extend the calendar to complete years as future source periods are added. [Microsoft: date tables](https://learn.microsoft.com/en-us/power-bi/transform-model/desktop-date-tables)

### Finish the semantic model

Hide foreign keys and raw numeric columns from casual report authoring; retain needed keys for drillthrough. Set identifiers to Don't summarize. Put measures into display folders such as Sales, Payments, Inventory and Quality. Add descriptions containing source, grain, units, status scope and date basis. Avoid bidirectional filtering as a shortcut for missing relationships.

## 7. DAX measures

Create each measure separately using New measure. Names assume the tables above. These are demonstration definitions, not approved financial metrics. Format amounts to two decimals with **no currency symbol**, units/counts as integers, and ratios as percentages. DAX has not been executed in Power BI during this audit; validate it in your model using the controls below.

```dax
Recorded Orders = COUNTROWS ( FactOrders )

Recorded Order Amount = SUM ( FactOrders[TotalAmount] )

Recorded Units Sold = SUM ( FactSales[Quantity] )

Calculated Gross Line Value =
SUMX ( FactSales, FactSales[Quantity] * FactSales[UnitPrice] )

Orders With Selected Products = DISTINCTCOUNT ( FactSales[OrderID] )

Purchasing Customers = DISTINCTCOUNT ( FactSales[CustomerID] )

Stores With Sales = DISTINCTCOUNT ( FactSales[StoreID] )

Gross Value Per Selected Order =
DIVIDE ( [Calculated Gross Line Value], [Orders With Selected Products] )

Weighted Line Unit Price =
DIVIDE ( [Calculated Gross Line Value], [Recorded Units Sold] )

Recorded Payment Amount = SUM ( FactPayments[Amount] )

Recorded Payment Records = COUNTROWS ( FactPayments )

Recorded Return Units = SUM ( FactReturns[Qty] )

Recorded Refund Amount = SUM ( FactReturns[RefundAmount] )

Recorded Waste Units = SUM ( FactWaste[Qty] )

Recorded Stock Units = SUM ( FactInventory[StockQty] )

Recorded Shipments = COUNTROWS ( FactShipments )

Invalid Shipment Intervals =
COUNTROWS (
    FILTER (
        FactShipments,
        NOT ISBLANK ( FactShipments[ShipDate] ) &&
        NOT ISBLANK ( FactShipments[ArrivalDate] ) &&
        FactShipments[ArrivalDate] < FactShipments[ShipDate]
    )
)

Valid Transit Days Average =
AVERAGEX (
    FILTER (
        FactShipments,
        NOT ISBLANK ( FactShipments[ShipDate] ) &&
        NOT ISBLANK ( FactShipments[ArrivalDate] ) &&
        FactShipments[ArrivalDate] >= FactShipments[ShipDate]
    ),
    DATEDIFF ( FactShipments[ShipDate], FactShipments[ArrivalDate], DAY )
)

Shipments By Arrival Date =
CALCULATE (
    [Recorded Shipments],
    USERELATIONSHIP ( DimDate[Date], FactShipments[ArrivalDate] )
)
```

The valid-transit average is a data-check statistic for observed intervals, not an on-time-delivery KPI. Display the excluded interval count beside it. A delivery SLA or promised arrival date is missing. The sample has no missing shipment dates; if future data does, count and disclose those separately as well.

Gross Value Per Selected Order becomes the selected products' value per order containing those products when a product filter is applied; it is not the full basket value. Counts such as distinct orders and customers are non-additive: matrix totals may not equal the sum of visible rows in realistic data.

### Quality measures after creating FactOrderQA

Build FactOrderQA at one row per OrderID by aggregating source lines and payments before merging. Include `HeaderAmount`, `GrossLineValue`, `PaymentAmount`, `HasLines` and `HasPayments`; preserve null when a child is missing. Optionally include order date/customer/store for diagnostic slicing. Relate DimOrder to it with a single 1:* relationship.

```dax
Orders With Amount Difference =
COUNTROWS (
    FILTER (
        FactOrderQA,
        FactOrderQA[HasLines] = TRUE () &&
        ABS ( FactOrderQA[HeaderAmount] - FactOrderQA[GrossLineValue] ) > 0.01
    )
)
```

The comparison flags a difference; it does not prove a net-to-gross accounting error until tax and discount rules are known. Add separate missing-line and missing-payment counts. Do not treat missing child amounts as zeros in reconciliation.

### Measures to defer

| Metric | Missing requirement |
|---|---|
| Net sales | Reliable discounts, refunds, tax convention, currency and recognition/status rules |
| Gross profit / margin | Reconciled net sales and transaction-time cost, including returned cost treatment |
| Collected cash / outstanding balance | Valid payment outcomes and allocation/settlement rules |
| Inventory turnover / days of stock | Dated inventory snapshots, consistent units and matched sales/cost periods |
| Stockout / reorder alerts | Valid ReorderLevel and trustworthy snapshot date |
| Waste cost / waste rate | Valid CostLoss or transaction cost and a defined input-volume denominator |
| Customer repeat / retention | Multiple orders per customer and sufficient history |
| Customer satisfaction | Numeric ratings and valid review content |
| Promotion ROI / uplift | Valid spend, discounts and defensible transaction attribution/baseline |
| On-time shipment rate | Promised delivery date or approved SLA, and valid completion status |
| YoY / MoM growth | Comparable prior periods and coverage |
| Sales vs target | Approved targets at a compatible date/entity grain |

Do not sum unit prices, average row-level margins, or divide returns by same-period sales and call the result a cohort return rate. Returns use return dates, while sales use order dates; production return rates need a defined order cohort and observation window.

## 8. Six-page report specification

Use a consistent 16:9 canvas, one title line, a narrow filter strip, a row of 3–5 cards, and a limited set of supporting visuals. Choose navy headings, a neutral background, one primary accent and amber/red for exceptions. Avoid decorative gauges without targets, crowded pie charts, and unsupported maps.

### Page 1 — Executive overview

Question: What does the extract contain, and what can management trust?

- Cards: Recorded Orders, Calculated Gross Line Value, Recorded Units Sold, Purchasing Customers.
- Line chart: DimDate.Date and Calculated Gross Line Value; display only actual coverage.
- Bar chart: DimProduct.Category and Calculated Gross Line Value, labeled source category pending validation.
- Bar chart: DimStore display label, top 10 by gross line value; include ID to avoid repeated placeholder names.
- Small quality notice with the 1,200 reconciliation differences and a button to Page 6.
- Filters: Order date, Store, Category. Use Orders With Selected Products instead of Recorded Orders when a category filter is active, or simply use that measure for the card consistently and name it Orders with selected products.

Do not mix the order-header amount into category visuals. Product filters do not reach FactOrders.

### Page 2 — Sales and product mix

Question: Which source product IDs/categories contribute to recorded line activity?

- Cards: gross line value, units, orders with selected products, weighted line unit price.
- Top 10 ProductLabel bar chart by gross line value.
- Daily value trend and category contribution chart.
- Matrix: rows Category > ProductLabel; values gross line value, units, orders with selected products, weighted line unit price.
- Slicers: Order date, Store, Category, Brand. Include a product mapping limitation in the tooltip.

Avoid claims of profitable products or best-performing promotions.

### Page 3 — Stores and customers

Question: How is recorded activity distributed across source stores and customers?

- Cards: distinct stores with sales, purchasing customers, gross line value, gross value per selected order.
- Store ranking using StoreID/display label.
- Customer activity table showing CustomerID, name, orders and gross value.
- Matrix: store display label as rows; gross value, units, distinct orders and purchasing customers as values.
- Slicers: Order date and Store; city is optional and explicitly unvalidated.

This source has one order per customer and store, so repeat purchasing and store productivity insights are not meaningful. Do not use Age, Gender, LoyaltyPoints or AreaSqFt.

### Page 4 — Inventory, waste and returns

Question: What inventory records and stock-related exceptions are available?

- Separate a **stock extract — snapshot date unavailable** panel from date-filtered event panels.
- Stock card: Recorded Stock Units. It does not respond to the event date slicer; state that beside it.
- Event cards: Recorded Waste Units and Recorded Return Units, clearly labeled as unvalidated quantities.
- Bar charts: waste units by product and returns by product.
- Inventory matrix: Store > ProductLabel; stock quantity and earliest recorded expiry date. For this simple snapshot, use MIN(ExpiryDate) only as a display summary, not a full expiry distribution.
- Exceptions table: returns greater than sold quantities, returns before orders, expired sale candidates.
- Slicers: Store and Product for the page. Event date filters only waste and returns, using their own event dates.

Do not subtract waste and returns from the stock snapshot: the extract is not a stock movement ledger. Do not compare these quantities as shares of one common denominator. Do not derive current expiry risk using TODAY() against this old undated inventory. An as-of analysis requires an explicitly selected reference date and a verified snapshot.

### Page 5 — Payments and supplier shipments

Question: Which recorded financial and shipment events need investigation?

- Payments panel: Recorded Payment Amount, payment record count, daily payment amount, raw status distribution and order-level reconciliation table.
- Shipments panel: Recorded Shipments, invalid interval count, valid transit average and supplier shipment count chart.
- Date selection means PaymentDate for payments and ShipDate for shipments; label this. Store can filter both panels.
- Supplier slicer controls shipment visuals only. Do not let an unaffected payment total appear to be supplier-filtered.
- No product/category slicer here: neither payment transactions nor shipment headers have a reliable product allocation.

For a later production report, split this page into Cash and Collections and Supplier Delivery when valid data supports richer analysis. Never label all payment rows as successful collections.

### Page 6 — Data quality and definitions

Question: Which defects prevent business interpretation, and how are metrics defined?

- Cards: source rows, failed rule count, distinct affected records, order reconciliation differences.
- Issue count by source table and rule, with severity labels.
- Matrix: SourceTable > Field > Rule, issue count and affected-record count.
- Exception detail table: source key, raw value, rule and follow-up owner if supplied.
- Definitions panel: grain, status scope, date basis, currency unavailable, source coverage and refresh timestamp.
- Slicers: Table, Rule, Severity. Do not synchronize business slicers onto this page unless the audit model explicitly supports them.

### Hidden page 7 — Order detail drillthrough

Use **DimOrder.OrderID** in the Drillthrough field well. Include that dimension field in the source order table so the context can be passed. Display header information, sales lines, payments, returns and reconciliation side by side; do not merge them into a multiplied table. Add a Back button.

For a complete order history, turn off Keep all filters so an incoming order-date selection does not suppress payments or returns that occurred later. Test this behavior. [Microsoft: drillthrough](https://learn.microsoft.com/en-us/power-bi/create-reports/desktop-drillthrough)

### Hidden page 8 — Sales tooltip

Set the page type to Tooltip. Show product label, units, gross line value, orders with selected products, weighted line price and the gross-value definition. Keep it short and assign it only to relevant sales visuals.

## 9. Matrix and interaction setup

1. Insert a Matrix visual. Drag the dimension hierarchy into Rows, the explicit measures into Values, and Date/YearMonth into Columns only when multiple periods exist. For this single partial month, omit redundant month columns.
2. Enable expand/collapse where useful. Format counts, amounts and percentages separately. Use subtotals at meaningful levels, not every nested field.
3. Apply data bars to comparable amount columns and conditional formatting to genuine exceptions. A high raw return quantity is not automatically a high return rate.
4. Keep identifiers with nonunique names. Use dimension fields for slicers and categories.
5. Test each total: distinct counts and weighted prices recalculate at the total level and should not be manually summed.
6. Set Edit interactions intentionally. Select a category and store to verify every responding visual has a valid relationship path.
7. Synchronize Store only across relevant pages. Date can be synchronized where event-date semantics are clear; inventory is the documented exception. Never synchronize Category to payment/shipment visuals that lack product attribution.
8. Add Page navigation and Reset filters buttons. Save a reset bookmark with the intended default filter state. A visible page title should always identify the active page.
9. Add descriptive titles, alt text, sensible tab order and color contrast; don't rely only on red/green. [Microsoft: accessibility](https://learn.microsoft.com/en-us/power-bi/create-reports/desktop-accessibility-creating-reports)

## 10. Validation before presentation

- Compare unfiltered totals with Section 2. Fixed-decimal sums should agree to the chosen 0.01 reporting precision.
- Confirm all 15 imports retain 1,200 rows before any justified exclusions. A 1,000-row preview is not a full validation.
- Confirm the dimensions have unique keys and Left Anti joins still return zero missing parent keys.
- Check row counts before and after every merge. Expanding a many-row child into a fact unexpectedly is a double-counting risk.
- Trace OrderID 1: recorded header amount is 2,954.60, while its line has 155 units at 3,367.06, giving 521,894.30 gross line value. Payment is 4,299.74. Keep all three visibly distinct.
- Check an order, product, customer and store filter separately. Make sure product filtering never makes an unchanged header/payment total look product-specific.
- Verify the invalid shipment count is 596 and orders with amount differences is 1,200 without filters.
- Test empty selections: do not replace missing data with an apparent zero achievement or zero defect rate.
- Verify inventory remains labeled undated and that event-date filtering does not imply historical stock availability.
- Check drillthrough and Back navigation, tooltip context, reset behavior, mobile layout if needed, and exported presentation views.
- Run Performance Analyzer after the design is complete. At 18,000 source rows, fix unnecessary visuals, ambiguous relationships and inefficient measures before adding complex optimizations.

## 11. Publish and handover

Only publish real business KPIs after the source owner resolves the critical defects and agrees on definitions. A clearly labeled demonstration can be presented sooner as a model/design exercise.

1. Save a versioned PBIX and retain its data dictionary, relationship specification and measure definitions.
2. Publish to the appropriate company workspace with approved access. Do not use public Publish to web for company data.
3. Configure source credentials and refresh. CSV files on a local PC generally require an available gateway for scheduled service refresh; alternatively use an approved managed source and matching connector. Test refresh in the service, not just Desktop. [Microsoft: refresh](https://learn.microsoft.com/en-us/power-bi/connect-data/refresh-data)
4. Configure store-based row-level security only if the company needs it and supplies user-to-store access rules. Test with View as and real viewer accounts. Hiding a page or column is not security. RLS does not restrict workspace Admin, Member or Contributor roles in the same way it restricts Viewers. [Microsoft: RLS](https://learn.microsoft.com/en-us/fabric/security/service-admin-row-level-security)
5. Review disconnected audit tables and other dimensions for sensitive detail before distribution; securing facts does not automatically secure every disconnected table. Keep contact details and salaries out of this management report unless specifically needed.
6. Deliver the PBIX, source/update instructions, metric definitions, quality issue log, access owner and refresh owner. Record known limitations and remediation ownership.

## 12. Practical build sequence

**First:** complete imports and the quality page. **Second:** build clean dimensions and FactOrders/FactSales, then reconcile. **Third:** add payments, returns, waste, inventory and shipments at their own grains. **Fourth:** implement and test measures. **Fifth:** build overview, sales, stores/customers, operations and payment/shipment pages. **Last:** configure navigation, drillthrough, accessibility, refresh and access, then rehearse the presentation.

For the company presentation, introduce the business questions, explain the model grain and relationships, show the report interaction, and disclose the source limitations before discussing numbers. This demonstrates analytical judgment as well as Power BI design skill.
