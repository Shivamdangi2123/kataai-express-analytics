# 🌾 Kataai Express — Agriculture Machinery Booking Analytics

An end-to-end data analytics project simulating a platform that connects farmers across Madhya Pradesh, Rajasthan, and Maharashtra with nearby harvesting-machine operators (combine harvesters, tractors, rotavators, reapers, threshers, straw balers).

This project covers the full analyst workflow: **raw data → Python cleaning → PostgreSQL → Power BI dashboard → business insights.**

---

## 📊 Dashboard Preview

*(Add a screenshot of your Power BI dashboard here — drag an image into this README on GitHub, or use the syntax below once uploaded)*

```md
![Dashboard Overview](docs/screenshots/dashboard_overview.png)
```

---

## 🧰 Tools & Tech Stack

`Python (pandas)` · `PostgreSQL` · `SQL` · `Power BI` · `DAX`

---

## 📁 Repository Structure

```
├── data/
│   ├── raw/            # Original, intentionally messy datasets (7 tables)
│   └── cleaned/         # Cleaned datasets after Python processing
├── python/
│   └── data_clean.ipynb # Data cleaning notebook (pandas)
├── powerbi/
│   └── katai_express_dashboards.pbix   # Power BI dashboard file
├── docs/
│   └── PROJECT_DOCUMENTATION.md        # Full project documentation
└── README.md
```

---

## 📦 About the Data

7 relational tables, ~300K+ rows total, simulating a real agri-tech marketplace:

| Table | Description | Rows (raw) |
|---|---|---|
| `locations` | District/state/zone reference data | 136 |
| `farmers` | Registered farmers | 20,200 |
| `operators` | Machine operators | 2,020 |
| `machines` | Harvesting machines (6 types) | 2,500 |
| `bookings` | Core transaction table | 109,080 |
| `payments` | Payment records | 104,636 |
| `reviews` | Farmer ratings/reviews | 62,179 |

The raw data was **intentionally messy** — duplicate rows, inconsistent text casing, mixed date formats, negative/invalid values, orphan foreign keys, and outliers — to simulate a realistic, imperfect real-world dataset.

---

## 🧹 Step 1: Data Cleaning (Python / pandas)

All 7 tables were cleaned using Python — see [`python/data_clean.ipynb`](python/data_clean.ipynb).

**Cleaning performed:**
- Removed duplicate records (by primary key)
- Standardized inconsistent text casing (state names, machine types, crop names, status values)
- Parsed 4 different mixed date formats into a single standard format
- Fixed invalid values: negative ages, negative prices/rates, out-of-range ratings → set to NULL
- Flagged (not deleted) outliers and orphan foreign keys in new boolean columns, preserving data for downstream analysis to decide inclusion/exclusion
- Left genuine missing data (phone numbers, cancelled-booking prices, review comments) as NULL — never artificially filled

Full breakdown of every cleaning decision is documented in [`docs/PROJECT_DOCUMENTATION.md`](docs/PROJECT_DOCUMENTATION.md).

---

## 🗄️ Step 2: PostgreSQL Database

Cleaned data was loaded into a normalized PostgreSQL database (`kataai_express`) — 7 tables with primary key constraints, bulk-loaded via `COPY`.

### SQL Analysis — status

✅ **Done:** Database is fully set up and query-ready. Initial validation queries confirmed zero negative/invalid values post-cleaning, and exploratory queries (top districts by demand, monthly booking trends, revenue by machine type) have been run directly against the database.

🔜 **In progress:** A dedicated `sql/` folder with `.sql` files answering the 10 core business questions (see below) using JOINs, CASE WHEN, CTEs, and window functions (ROW_NUMBER, RANK, LAG/LEAD, running totals) is being added — check back for the full query set, or see [`docs/PROJECT_DOCUMENTATION.md`](docs/PROJECT_DOCUMENTATION.md) for the current status.

**The 10 business questions being answered:**
1. Which districts have the highest harvesting demand?
2. Which months/season see peak demand?
3. Which machine types are most utilized / most profitable?
4. How is machine utilization (%) trending?
5. How are operators performing (completions, cancellations, ratings)?
6. Why are bookings getting cancelled?
7. What is the average amount farmers pay per booking?
8. Where does company revenue come from (district / machine type / month)?
9. Where is there a supply-demand gap (high demand, low machine availability)?
10. Where should the company expand / add machine supply next?

---

## 📈 Step 3: Power BI Dashboard

An interactive Power BI dashboard ([`powerbi/katai_express_dashboards.pbix`](powerbi/katai_express_dashboards.pbix)) with multiple report pages:

- **Dashboard (Overview)** — Total Bookings, GMV, Cancellation Rate, monthly demand trend, revenue by machine type
- **Bookings** — Seasonal demand analysis, crop mix, booking status breakdown
- **Machines** — Machine utilization %, revenue by machine type, supply coverage
- **Operators** — Operator performance, completion/cancellation rates, ratings
- **Farmers** — Farmer registration trends, repeat-booking behavior, state-wise distribution

Built with a custom Calendar table for time intelligence and 20+ DAX measures (cancellation rate, machine utilization %, demand growth %, repeat farmer %, company revenue, and more).

---

## 🗃️ SQL Analysis

Business questions were answered by querying the PostgreSQL database directly. Analysis is in progress — 3 of 10 core questions completed so far, using aggregation, JOINs, and GROUP BY.

### 1. Which districts have the highest demand?
```sql
SELECT district, COUNT(*) AS total_bookings
FROM bookings
GROUP BY district
ORDER BY total_bookings DESC
LIMIT 10;
```
| District | Total Bookings |
|---|---|
| Rajgarh | 1,815 |
| Barwani | 1,759 |
| Raigad | 1,713 |
| Satna | 1,658 |
| Ahmednagar | 1,588 |
| Niwari | 1,526 |
| Chandrapur | 1,521 |
| Nashik | 1,520 |

### 2. Which months see peak demand?
```sql
SELECT EXTRACT(MONTH FROM booking_date) AS month, COUNT(*) AS total_bookings
FROM bookings
GROUP BY month
ORDER BY month;
```
| Month | Bookings |
|---|---|
| Jan | 3,498 |
| Feb | 3,617 |
| Mar | 16,799 |
| Apr | 16,654 |
| May | 3,545 |
| Oct | 21,954 |
| Nov | 22,059 |
| Dec | 3,535 |

Confirms a clear seasonal pattern: bookings in Oct–Nov and Mar–Apr are roughly **5–6× higher** than off-season months.

### 3. Which machine types are most profitable?
```sql
SELECT m.machine_type, SUM(b.price) AS total_revenue, COUNT(*) AS total_bookings
FROM bookings b
JOIN machines m ON b.machine_id = m.machine_id
WHERE b.status = 'Completed'
GROUP BY m.machine_type
ORDER BY total_revenue DESC;
```
| Machine Type | Total Revenue (₹) | Total Bookings |
|---|---|---|
| Combine Harvester | 50,58,58,402 | 24,677 |
| Tractor | 14,48,34,937 | 18,135 |
| Reaper | 7,25,06,659 | 10,683 |
| Rotavator | 6,10,81,520 | 10,896 |
| Thresher | 4,74,33,178 | 7,494 |

Combine Harvesters generate the largest share of revenue by a wide margin, despite not having the most bookings — reflecting their higher per-booking price.

**Remaining questions** (operator performance, cancellation drivers, supply-demand gap, revenue trends using window functions/CTEs) are in progress — the database is fully set up and query-ready for this.

---

## 💡 Key Business Insights

- **Seasonal demand is highly concentrated**: bookings in the Oct–Nov and Mar–Apr harvesting windows run roughly **6× higher** than off-season months — a clear signal for proactive machine-supply planning ahead of peak season.
- **Strong platform retention**: **97.3% of farmers are repeat customers** (booked more than once), indicating farmers return for multiple harvesting cycles rather than one-off use.
- **Combine Harvesters dominate revenue**, contributing the largest share of GMV among all 6 machine types despite being a smaller share of the total fleet.
- Clear district-level demand concentration exists, highlighting where additional machine supply would have the highest impact.

---

## 🗃️ SQL Analysis

Business questions were answered directly against the PostgreSQL database using aggregation, JOINs, CTEs, and window functions (`RANK`, `LAG`).

**Top districts by demand**
```sql
SELECT district, COUNT(*) AS total_bookings
FROM bookings GROUP BY district ORDER BY total_bookings DESC LIMIT 10;
```
→ Rajgarh leads with 1,815 bookings, followed by Barwani (1,759) and Raigad (1,713).

**Seasonal demand pattern**
```sql
SELECT EXTRACT(MONTH FROM booking_date) AS month, COUNT(*) AS total_bookings
FROM bookings GROUP BY month ORDER BY month;
```
→ Confirms the harvest-season spike: March (16,799) and April (16,654) bookings are ~4.7× higher than a typical off-season month (~3,500); October (21,954) and November (22,059) peak even higher, at ~6× off-season levels.

**Revenue by machine type**
```sql
SELECT m.machine_type, SUM(b.price) AS total_revenue, COUNT(*) AS total_bookings
FROM bookings b JOIN machines m ON b.machine_id = m.machine_id
WHERE b.status = 'Completed' GROUP BY m.machine_type ORDER BY total_revenue DESC;
```
→ Combine Harvesters generate ₹50.6 Cr — by far the largest share of total completed-booking revenue (₹87.2 Cr), despite Tractors having a comparable booking count.

**Operator performance** (completion rate, cancellation rate, avg rating — min. 20 bookings)
```sql
SELECT o.name, COUNT(b.booking_id) AS total_bookings,
    ROUND(COUNT(*) FILTER (WHERE b.status='Completed')*100.0/COUNT(*),2) AS completion_rate,
    ROUND(COUNT(*) FILTER (WHERE b.status='Cancelled')*100.0/COUNT(*),2) AS cancellation_rate,
    ROUND(AVG(r.rating),2) AS avg_rating
FROM bookings b
JOIN operators o ON b.operator_id = o.operator_id
LEFT JOIN reviews r ON r.booking_id = b.booking_id
GROUP BY o.name HAVING COUNT(b.booking_id) >= 20
ORDER BY avg_rating DESC LIMIT 15;
```
→ Top-rated operators sit around 4.2–4.4★, with completion rates varying widely (60–78%) — cancellation rate is not strongly tied to rating, suggesting other factors (demand timing, machine availability) drive cancellations more than operator quality.

**Cancellation rate by machine type**
```sql
SELECT m.machine_type,
    ROUND(COUNT(*) FILTER (WHERE b.status='Cancelled')*100.0/COUNT(*),2) AS cancellation_rate
FROM bookings b JOIN machines m ON b.machine_id = m.machine_id
GROUP BY m.machine_type ORDER BY cancellation_rate DESC;
```
→ Cancellation rate is fairly uniform across machine types (14.1%–14.8%) — no single machine type is disproportionately driving cancellations.

**Average booking value**
```sql
SELECT ROUND(AVG(price),2) AS avg_booking_value FROM bookings WHERE status = 'Completed';
```
→ ₹11,748.55 average per completed booking.

**Revenue by district — using a CTE**
```sql
WITH monthly_revenue AS (
    SELECT district, EXTRACT(MONTH FROM booking_date) AS month, SUM(price) AS revenue
    FROM bookings WHERE status = 'Completed' GROUP BY district, month
)
SELECT district, SUM(revenue) AS total_revenue
FROM monthly_revenue GROUP BY district ORDER BY total_revenue DESC LIMIT 10;
```
→ Nashik tops total revenue (₹1.62 Cr) despite not being the #1 district by booking count — indicating higher-value bookings (likely more Combine Harvester usage) rather than just volume.

**District ranking — window function (`RANK`)**
```sql
SELECT district, SUM(price) AS revenue,
    RANK() OVER (ORDER BY SUM(price) DESC) AS revenue_rank
FROM bookings WHERE status = 'Completed' GROUP BY district LIMIT 15;
```

**Month-over-month growth — window function (`LAG`)**
```sql
WITH monthly AS (
    SELECT EXTRACT(MONTH FROM booking_date) AS month, COUNT(*) AS bookings
    FROM bookings GROUP BY month ORDER BY month
)
SELECT month, bookings,
    LAG(bookings) OVER (ORDER BY month) AS prev_month,
    ROUND((bookings - LAG(bookings) OVER (ORDER BY month))*100.0/LAG(bookings) OVER (ORDER BY month),2) AS growth_pct
FROM monthly;
```
→ Quantifies the seasonal swing precisely: bookings jump **+364% month-over-month into March** and **+511% month-over-month into October** — the sharpest demand inflections in the calendar, critical for supply-planning timing.

**Supply-demand gap — bookings per active machine, by district**
```sql
SELECT b.district,
    COUNT(*) AS total_bookings,
    (SELECT COUNT(*) FROM machines m WHERE m.district = b.district AND m.status = 'Active') AS active_machines,
    ROUND(COUNT(*)::numeric / NULLIF((SELECT COUNT(*) FROM machines m WHERE m.district = b.district AND m.status = 'Active'), 0), 2) AS bookings_per_machine
FROM bookings b
GROUP BY b.district
ORDER BY bookings_per_machine DESC
LIMIT 10;
```
→ Identifies the clearest expansion targets: **Jalgaon** (183 bookings per active machine) and **Amravati** (155 per machine) are the most supply-constrained districts — far above the healthier ~95–110 range seen elsewhere — making them the top priority for adding machine supply ahead of peak season.

---

## 🔍 What's Next

- Python EDA notebook with statistical visualizations
- Expanded Power BI pages (District-level drill-down)

---

## 👤 About

Built as a personal data analytics portfolio project to demonstrate the full workflow: data cleaning → database design → dashboarding → business storytelling.
