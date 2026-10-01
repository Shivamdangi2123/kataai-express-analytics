# Kataai Express — Agriculture Machinery & Harvesting Analytics
## Project Documentation — Work Completed So Far

---

## 1. Business Context

**Company (fictional, portfolio project):** Kataai Express
**What it does:** Connects farmers across Madhya Pradesh, Rajasthan and Maharashtra with nearby harvesting-machine operators (combine harvesters, tractors, rotavators, reapers, threshers, straw balers).

**Data scale:**
| Table | Rows (raw) |
|---|---|
| locations | 136 |
| farmers | 20,200 |
| operators | 2,020 |
| machines | 2,500 |
| bookings | 109,080 |
| payments | 104,636 |
| reviews | 62,179 |

---

## 2. ✅ Data Cleaning (Python / pandas — done by hand)

All 7 raw Excel files were cleaned manually in Python (not Excel), table by table.

### Cleaning done per table

**locations**
- Removed duplicate `location_id` rows
- Standardized `district` (trim + title case)
- `state` had no messy variants in this table
- `zone`: 32 NULLs left as genuine missing data (not filled)

**farmers** (20,200 → 20,000 after dedup)
- Removed 200 duplicate `farmer_id` rows
- 91 negative `age` values → set to NULL
- `state` standardized (MP / M.P. / madhya pradesh / lowercase → proper case: "Madhya Pradesh", "Rajasthan", "Maharashtra")
- `district` standardized (trim + title case)
- `farm_size_acres` outliers (>100 acres) flagged in new column `farm_size_outlier` — 49 rows flagged, **not deleted**
- `phone` formatting cleaned (removed trailing `.0` from numeric import) — 424 NULLs left as-is
- `registration_date`: mixed formats (dd/mm/yyyy, yyyy/mm/dd, dd-mm-yyyy, yyyy-mm-dd) parsed into a single date format using a custom `parse_date()` function that tries each format in turn

**operators**
- Removed duplicate `operator_id` rows
- Negative `age` → set to NULL
- `state`, `district`, `status` standardized (casing)
- `phone` formatting cleaned
- `join_date` mixed formats parsed

**machines**
- Removed duplicate `machine_id` rows
- `machine_type` casing standardized ("combine harvester" → "Combine Harvester")
- `status` casing standardized
- Negative `hourly_rate` → set to NULL
- Orphan `operator_id` (the injected `999999` placeholder) flagged in new column `operator_orphan` — **not deleted**

**bookings** (largest table, 109,080 rows)
- Removed duplicate `booking_id` rows (~1,080 removed)
- Orphan `machine_id` (`999999`) flagged in new column `machine_orphan`
- `district`, `status` casing standardized
- `crop` casing standardized + spelling fixed ("Soyabean" → "Soybean")
- `acreage` outliers (>100 acres) flagged in new column `acreage_outlier`
- Negative `price` → set to NULL
- `booking_date` mixed formats parsed
- `farmer_id` and `price` NULLs left as-is (genuine missing data — e.g. cancelled bookings often have no price)

**payments**
- Removed duplicate `payment_id` rows
- Orphan `booking_id` (referencing bookings removed by dedup) flagged in new column `booking_orphan`
- `payment_mode`, `status` casing standardized
- `payment_date` mixed formats parsed

**reviews**
- Removed duplicate `review_id` rows
- Invalid `rating` values (0 or 6, outside valid 1–5 range) → set to NULL
- `review_date` mixed formats parsed

### Cleaning principles followed
- **Genuine missing data is never filled** (phone, price on cancelled bookings, comments, zone) — left as NULL, only counted
- **Outliers and orphan foreign keys are never deleted** — flagged in new boolean columns instead, so downstream SQL/BI analysis can decide whether to include or exclude them
- **Invalid values** (negative age/price/rate, out-of-range ratings) set to NULL rather than guessed or corrected

### Output
7 cleaned files exported: `locations_cleaned.xlsx`, `farmers_cleaned.xlsx`, `operators_cleaned.xlsx`, `machines_cleaned.xlsx`, `bookings_cleaned.xlsx`, `payments_cleaned.xlsx`, `reviews_cleaned.xlsx`

---

## 3. ✅ PostgreSQL Database

- Database created: **`kataai_express`**
- All 7 tables created via `CREATE TABLE` with `PRIMARY KEY` constraints, including the new flag columns added during cleaning (`farm_size_outlier`, `operator_orphan`, `machine_orphan`, `acreage_outlier`, `booking_orphan`)
- FOREIGN KEY constraints intentionally **not** added — orphan IDs (`999999`) exist by design in the raw data, so strict FKs would block loading; flag columns handle this instead
- Cleaned `.xlsx` files exported to CSV, then loaded using `COPY table(...) FROM '<path>' DELIMITER ',' CSV HEADER;`
- Row counts verified after load:

| Table | Rows loaded |
|---|---|
| locations | 136 |
| operators | 2,000 |
| machines | 2,500 |
| farmers | 20,000 |
| reviews | 62,179 |
| payments | 103,600 |
| bookings | 108,000 |

Database is fully queryable — SQL analysis (the 10 core business questions) can be run against it any time.

---

## 4. ✅ Power BI — Data Model

- Connected Power BI Desktop to PostgreSQL (`localhost`, database `kataai_express`)
- All 7 tables imported
- **Relationships built** (Model view):

| From (many side) | To (one side) |
|---|---|
| bookings[farmer_id] | farmers[farmer_id] |
| bookings[machine_id] | machines[machine_id] |
| bookings[operator_id] | operators[operator_id] |
| payments[booking_id] | bookings[booking_id] |
| reviews[booking_id] | bookings[booking_id] |
| machines[operator_id] | operators[operator_id] |

- **Calendar table** created for time intelligence:
```DAX
Calendar = CALENDAR(MIN(bookings[booking_date]), MAX(bookings[booking_date]))
Year = YEAR('Calendar'[Date])
Month No = MONTH('Calendar'[Date])
Month = FORMAT('Calendar'[Date], "MMM")
YearMonth = FORMAT('Calendar'[Date], "YYYY-MM")
Season = SWITCH(TRUE(),
    'Calendar'[Month No] IN {10,11}, "Peak: Oct-Nov",
    'Calendar'[Month No] IN {3,4}, "Peak: Mar-Apr",
    "Off-Season")
```
- `Calendar[Date]` linked to `bookings[booking_date]` (one-to-many)
- `Month` column sorted by `Month No` for correct chronological order in charts
- `Calendar` marked as an official Date table

### DAX measures written (on `bookings` table)
```DAX
Total Bookings = COUNTROWS(bookings)
Completed Bookings = CALCULATE(COUNTROWS(bookings), bookings[status] = "Completed")
Cancelled Bookings = CALCULATE(COUNTROWS(bookings), bookings[status] = "Cancelled")
Pending Bookings = CALCULATE(COUNTROWS(bookings), bookings[status] = "Pending")
Cancellation Rate = DIVIDE([Cancelled Bookings], [Total Bookings])
Completion Rate = DIVIDE([Completed Bookings], [Total Bookings])
GMV = CALCULATE(SUM(bookings[price]), bookings[status] = "Completed")
Avg Booking Value = DIVIDE([GMV], [Completed Bookings])
Avg Rating = AVERAGE(reviews[rating])
Commission Rate = 0.15
Company Revenue = [GMV] * [Commission Rate]
Operator Payout = [GMV] - [Company Revenue]
Booked Hours = CALCULATE(SUM(bookings[duration_hours]), bookings[status] = "Completed")
Available Hours = CALCULATE(SUM(machines[available_hours_per_day]), machines[status] = "Active") * DISTINCTCOUNT('Calendar'[Date])
Machine Utilization % = DIVIDE([Booked Hours], [Available Hours])
Bookings Prev Month = CALCULATE([Total Bookings], DATEADD('Calendar'[Date], -1, MONTH))
Demand Growth % = DIVIDE([Total Bookings] - [Bookings Prev Month], [Bookings Prev Month])
Repeat Farmers = COUNTROWS(FILTER(VALUES(bookings[farmer_id]), CALCULATE(COUNTROWS(bookings)) > 1))
Repeat Farmer % = DIVIDE([Repeat Farmers], DISTINCTCOUNT(bookings[farmer_id]))
Avg Acreage = CALCULATE(AVERAGE(bookings[acreage]), bookings[acreage_outlier] = FALSE())
Active Machines = CALCULATE(COUNTROWS(machines), machines[status] = "Active")
Active Machines in District = CALCULATE([Active Machines], TREATAS(VALUES(bookings[district]), machines[district]))
Bookings per Machine = DIVIDE([Total Bookings], [Active Machines in District])
Payment Success % = DIVIDE(CALCULATE(COUNTROWS(payments), payments[status] = "Success"), COUNTROWS(payments))
Refunded Amount = CALCULATE(SUM(payments[amount]), payments[status] = "Refunded")
```

### Two assumptions documented
1. Dataset has no commission column, so a **15% commission rate is assumed** for `Company Revenue` / `Operator Payout` — easy to change later via the `Commission Rate` measure.
2. `GMV` is calculated only from **Completed** bookings, since cancelled bookings frequently have a NULL price.

### Power BI theme
A custom theme JSON (`kataai_express_theme.json`) was created matching the project's harvest color palette (deep green `#1F4D2C`, harvest gold `#D9A441`, soil brown `#7A5230`) for import via View → Themes → Browse for themes.

---

## 5. ✅ Web-style Dashboard Prototypes (bonus, not part of original plan)

Two standalone HTML dashboards were built (outside Power BI) using real computed numbers from the cleaned data, styled in the harvest color theme with sidebar navigation, KPI cards, and charts — modeled after modern SaaS dashboard layouts. These are exploratory/reference prototypes, not the final deliverable.

**Key real numbers surfaced in this process** (useful for any future report/dashboard):
- Total Bookings: 108,000 → 71.2% Completed, 14.5% Cancelled, 14.3% Pending
- GMV: ₹88.6 Cr | Avg Booking Value: ₹11,749
- Avg Rating: 3.79 / 5
- Farmers by state: Madhya Pradesh 43.0%, Maharashtra 29.7%, Rajasthan 27.3%
- Top districts by bookings: Rajgarh (1,815), Barwani (1,759), Raigad (1,713), Satna (1,658), Ahmednagar (1,588)
- Revenue by machine type: Combine Harvester ₹65.7 Cr (dominant), Tractor ₹18.9 Cr, Reaper ₹9.4 Cr, Rotavator ₹8.0 Cr, Thresher ₹6.2 Cr, Straw Baler ₹5.3 Cr
- Seasonality: Oct–Nov and Mar–Apr bookings run roughly **6× higher** than off-season months
- Machine status: 1,538 Active / 482 Inactive / 480 Under Maintenance
- Payment modes: UPI 33.5%, Bank Transfer 16.9%, Cash 16.9%, Card 16.8%
- Crop mix: Wheat and Soybean lead (~27K bookings each), followed by Maize, Sugarcane, Gram, Cotton (~13K each)

---

## 6. ⏳ Not Done Yet

- [ ] SQL analysis — the 10 core business questions, using JOINs / CTEs / window functions (database is ready and waiting; can be picked up any time)
- [ ] Python EDA notebook (charts, distributions, correlations)
- [ ] Final Power BI report pages/visuals (data model + measures are ready, but the actual report pages/cards/charts were not finalized)
- [ ] Business insights & recommendations write-up
- [ ] GitHub repository structure + README
- [ ] Resume bullet points + interview prep

### Reference — 10 business questions still to answer
1. Which districts have the highest harvesting demand?
2. Which months/season see peak demand?
3. Which machine types are most utilized / most profitable?
4. How is machine utilization (%) trending?
5. How are operators performing (completions, cancellations, ratings)?
6. Why are bookings getting cancelled? *(note: dataset has no cancellation-reason column, so this can only be answered indirectly via cancellation rate by machine type/crop/season)*
7. What is the average amount farmers pay per booking?
8. Where does company revenue come from (district / machine type / month)?
9. Where is there a supply-demand gap (high demand, low machine availability)?
10. Where should the company expand / add machine supply next?

---

## 7. Files produced so far

```
locations_cleaned.xlsx, farmers_cleaned.xlsx, operators_cleaned.xlsx,
machines_cleaned.xlsx, bookings_cleaned.xlsx, payments_cleaned.xlsx,
reviews_cleaned.xlsx                                  — cleaned data
kataai_express (PostgreSQL database)                  — 7 tables loaded
Kataai Express.pbix                                   — Power BI file (model + measures)
kataai_express_theme.json                             — Power BI color theme
kataai_dashboard.html / kataai_dashboard_v2.html       — HTML dashboard prototypes
```
