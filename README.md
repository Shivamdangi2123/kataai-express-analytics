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

## 💡 Key Business Insights

- **Seasonal demand is highly concentrated**: bookings in the Oct–Nov and Mar–Apr harvesting windows run roughly **6× higher** than off-season months — a clear signal for proactive machine-supply planning ahead of peak season.
- **Strong platform retention**: **97.3% of farmers are repeat customers** (booked more than once), indicating farmers return for multiple harvesting cycles rather than one-off use.
- **Combine Harvesters dominate revenue**, contributing the largest share of GMV among all 6 machine types despite being a smaller share of the total fleet.
- Clear district-level demand concentration exists, highlighting where additional machine supply would have the highest impact.

---

## 🔍 What's Next

- SQL-based deep-dive analysis (window functions, CTEs) answering 10 core business questions
- Python EDA notebook with statistical visualizations
- Expanded Power BI pages (District-level drill-down)

---

## 👤 About

Built as a personal data analytics portfolio project to demonstrate the full workflow: data cleaning → database design → dashboarding → business storytelling.
