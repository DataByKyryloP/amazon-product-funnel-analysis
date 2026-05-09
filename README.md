# Amazon Product Funnel Analysis
### Price, Engagement & Category Performance on Amazon.com

**Live Dashboard →** [Looker Studio Report](https://datastudio.google.com/s/jJJDPLOT3N8)

![Dashboard Overview](visuals/dashboard_final.png)

---

## 🧠 Executive Summary

This project analyzes how product-level attributes (price, category, ratings, and reviews) influence engagement in an Amazon-style marketplace.

The goal is to model a simplified product funnel:

**Visibility → Engagement → Review Accumulation → Demand Proxy**

Key finding: product success is highly concentrated, with a small subset of listings driving the majority of engagement, while pricing effects vary significantly by category rather than following a universal pattern.

## Business Questions

1. How do products drop off across engagement levels — from listed to highly reviewed?
2. Do higher-priced products receive more engagement and reviews?
3. Which categories generate the highest review density and engagement efficiency?
4. What does the distribution of product visibility look like — and where is concentration?
5. Are there identifiable high-performing product tiers, and what separates them?

---

## Data Pipeline

```
Axesso Real-Time Amazon Data API (RapidAPI)
        ↓
Python / Pandas — JSON parsing, cleaning, feature engineering
        ↓
Google BigQuery — SQL funnel queries, aggregations, segmentation
        ↓
Looker Studio — Live interactive dashboard connected to BigQuery
```

Data was extracted on **19/04/2026** across three product categories, yielding **224 unique product listings** from amazon.com. The raw file is preserved untouched; all transformations were applied to versioned copies.

> To ensure reproducibility of the data pipeline, a clean version of the dataset was regenerated directly from the notebook-based transformation workflow. This version was used to validate final data types, feature engineering logic, and discount handling before export for SQL-based analysis.

---

## Tools & Stack

| Layer | Tool | Purpose |
|---|---|---|
| Extraction | Python (requests, pandas) | API pull, JSON parsing, category labelling |
| Cleaning & Engineering | Python (pandas, numpy) | Null handling, type casting, feature creation |
| Storage & Analysis | Google BigQuery (SQL) | Funnel queries, aggregations, segmentation |
| Visualization | Looker Studio | Live interactive dashboard connected to BigQuery |
| Version Control | GitHub | Portfolio structure, reproducibility |

---

## Data Cleaning & Feature Engineering

Key transformations applied before analysis:

**Price cleaning** — API returns `0.0` for missing prices; replaced with `NaN` and recast to float.

**Sales volume normalization** — API returns bucketed strings (`"10K+ bought in past month"`). Parsed to approximate numeric values directionally; comparisons should be treated as proxy rather than exact counts.

**Product rating extraction** — Raw format `"4.8 out of 5 stars"` parsed to `float64`.

**Discount flag** — Derived from `price < retailPrice` comparison, only where both values exist:

```python
df["is_discounted"] = np.where(
    df["retailPrice"].notna() & df["price"].notna(),
    (df["price"] < df["retailPrice"]).astype(int),
    np.nan
)
```

**Engagement features:**

```python
df["has_reviews"]    = (df["countReview"] > 0).astype(int)
df["review_density"] = df["countReview"] / (df["productRating"] + 0.01)
```

**Price tier segmentation** — Tiers defined *within each category* using tertile splits, ensuring a "budget" wireless headphone is cheap relative to other headphones, not relative to coffee beans:

```python
df["price_tier"] = df.groupby("category")["price"].transform(
    lambda x: pd.qcut(x, q=3, labels=["budget", "mid", "premium"], duplicates="drop")
)
```
---

## Notes on Data Integrity

Both the raw and cleaned datasets are preserved in `/data/`. The raw file (`raw_amazon_products.csv`) was never modified after the initial pull. All transformations — null handling, type casting, feature engineering, and derived column creation — were applied to versioned copies and are fully documented in the Jupyter notebooks.

---

## SQL Analytics

Each SQL query is designed to answer a specific business question, moving from funnel structure → category performance → pricing behavior → engagement segmentation.

| Query | Business Question |
|---|---|
| `01_funnel_analysis.sql` | How many products reach each review threshold? Where does drop-off occur? |
| `02_category_performance.sql` | Which categories lead on price, rating, and review engagement? |
| `03_price_tier_by_category_CTE.sql` | How does price tier affect performance within each category? |
| `04_review_density.sql` | Which categories convert visibility into reviews most efficiently? |
| `05_engagement_stage_analysis.sql` | How do products cluster by review maturity — does it predict quality? |

Each query file includes a comment header stating the business question it addresses.

### Funnel Drop-off (Query 1)

```sql
SELECT
  COUNT(*) AS total_products,
  COUNT(productRating) AS products_with_rating,
  COUNTIF(countReview >= 50) AS products_50_reviews,
  COUNTIF(countReview >= 500) AS products_500_reviews,
  COUNTIF(has_reviews = 1) AS products_with_any_reviews
FROM amazonfunnel.products_clean.products_clean;
```

| Stage | Count | Drop-off |
|---|---|---|
| All products | 224 | — |
| 500+ reviews | 179 | −20% |
| 2000+ reviews | 125 | −25% |
| 5000+ reviews | 92 | −15% |

---

## 📊 Key Findings

- Product engagement follows a strong **Pareto distribution** — a small subset of listings drives the majority of reviews and activity.
- Pricing impact is **category-dependent**, not universal (no single optimal price tier).
- Wireless headphones show significantly higher engagement intensity than other categories.
- Ratings remain stable across price tiers, meaning **price affects demand, not perceived quality**.

---

## 💼 Business Use Cases

This analysis can support real-world marketplace and product strategy decisions:

- **Pricing Strategy Optimization**  
  Identify optimal pricing tiers per category based on engagement behavior.

- **Category Expansion Decisions**  
  Evaluate which categories generate higher engagement efficiency before investment.

- **Product Listing Optimization**  
  Detect underperforming products with high visibility but low engagement.

- **Marketing Allocation**  
  Prioritize ad spend toward high engagement-to-review categories (e.g., electronics).

- **Inventory Planning**  
  Use engagement signals as demand proxies where sales data is limited.

---

## Dashboard

**Live →** [Looker Studio Report](https://datastudio.google.com/s/jJJDPLOT3N8)

Connected directly to BigQuery. Includes five analytical panels covering the full funnel, category comparison, price tier segmentation, engagement efficiency, and lifecycle segmentation. A category filter control allows all panels to be sliced simultaneously.

![Category Panels](visuals/category_analysis.png)

![Price Tier and Demand Distribution](visuals/price_tier_view.png)

---

## Limitations

- **Single point-in-time snapshot** — Amazon pricing, review counts, and rankings fluctuate continuously; findings reflect one moment rather than a trend.
- **BSR (Best Seller Rank) unavailable** — this would have been the strongest direct demand signal; the product-level lookup endpoint would be required to retrieve it.
- **salesVolume is bucketed, not exact** — `"1K+ bought in past month"` was parsed to a numeric approximation (1000), meaning demand comparisons are directional rather than precise.
- **Discount analysis is limited to 78/224 products** where both price and retail price were available. As a result, discount-related insights represent a partial but reliable subset of pricing behavior rather than a full-market view.
- **Free API tier capped category depth** — wireless headphones returned only 32 products vs 96 per other category, meaning headphone metrics carry less statistical weight.
- **manufacturer and series fields fully empty** — likely gated behind a paid API tier; brand-level analysis was not possible.
- **Aggregated dashboard only** — all visuals reflect category or tier averages; product-level outliers are not visible at this view.

👉 This analysis reflects correlation, not causation — results should be interpreted as behavioral signals rather than direct causal effects.
---

## What Could Be Explored Further

- **Weekly pulls of the same ASINs** to track BSR, price, and review count over time — converting a snapshot into a time-series trend analysis.
- **Product-level lookup endpoint** to enrich each ASIN with full BSR and category rank, replacing the review-count proxy with a direct demand signal.
- **Expand to 10+ categories** for genuine cross-category benchmarking and to test whether the behavioral patterns observed here hold at scale.
- **Predictive engagement model** — given a product's price tier and category, estimate the review threshold trajectory using historical Amazon data.
- **Seller type segmentation** — does brand vs third-party seller status affect review accumulation rates and pricing strategy?

---

## 🚀 Final Reflection

This project demonstrates how raw marketplace data can be transformed into structured business insights using Python and SQL.

The key takeaway is that product performance is not evenly distributed — success is concentrated, and effective strategy depends on category-specific behavior rather than global assumptions.

---

## Repository Structure

```
amazon-product-funnel-analysis/
│
├── data/
│   ├── raw/
│   │   └── raw_amazon_products.csv
│   │       # Original API pull — never modified
│   │
│   └── processed/
│       ├── cleanest_amazon_products.csv
│       ├── amazon_products_clean_v1.csv
│       ├── amazon_products_clean_v2.csv
│       ├── amazon_products_clean_v3.csv
│       ├── amazon_products_clean_v4(final).csv
│       └── cleanest_amazon_products_reproducible_May9.csv
│           # Final reproducible notebook output used for validation
│
├── notebooks/
│   ├── 01_data_pipeline_amazon_funnel.ipynb
│   ├── 02_data_final_cleaning.ipynb
│   ├── archive/
│   │   ├── 01_data_pipeline_amazon_funnel_raw_version.ipynb
│   │   └── 02_data_final_cleaning_raw_version.ipynb
│
├── queries/
│   ├── 01_funnel/
│   │   └── funnel_analysis.sql
│   │
│   ├── 02_category/
│   │   └── 02_category_performance.sql
│   │
│   ├── 03_price/
│   │   └── 03_price_tier_by_category.sql
│   │
│   ├── 04_review/
│   │   └── 04_review_density.sql
│   │
│   ├── 05_engagement/
│   │   └── 05_engagement_stage_analysis.sql
│   │
│   └── optional/
│       ├── (06_query)_discounting influence on engagement and perceived product quality.sql
│       └── 03_price_tier_by_category(CTE version).sql
├── visuals/
│   ├── dashboard_final.png
│   ├── category_analysis.png
│   └── price_tier_view.png
│
└── README.md
```


*Data pulled: 19/04/2026 · Source: Axesso Real-Time Amazon Data API via RapidAPI · Dataset: 224 products · Categories: board games, coffee beans, wireless headphones*

