# CLAUDE.md — Olist AI Data Analyst Project

This file documents what has been learned from running agents on the OLIST_ECOMMERCE database.
Agents should read this before starting any new analysis to avoid re-discovering known issues.

---

## Project Overview

**Database:** OLIST_ECOMMERCE (Snowflake)
**Schema:** RAW_DATA
**Connection:** Snowflake MCP server (see `.claude/mcp.json`)
**Data period:** 2016-09-04 to 2018-10-17
**Primary analysis window:** January 2017 – August 2018 (full months only — exclude partial months at both ends)

---

## Output Conventions

| Artifact | Path |
|---|---|
| Cleaned CSVs | `outputs/cleaned_data/<table>_cleaned.csv` |
| Charts (PNG) | `outputs/charts/` |
| Reports (Markdown) | `outputs/reports/` |
| Analysis scripts | `outputs/` |

Always use **cleaned CSVs** as the primary data source. Fall back to Snowflake only if a cleaned version does not exist.

---

## Schema: Tables and Columns

### ORDERS (99,441 rows)
| Column | Type | Notes |
|---|---|---|
| ORDER_ID | VARCHAR | Primary key — no duplicates |
| CUSTOMER_ID | VARCHAR | FK → CUSTOMERS.CUSTOMER_ID |
| ORDER_STATUS | VARCHAR | Values: delivered, shipped, canceled, unavailable, invoiced, processing, created, approved |
| ORDER_PURCHASE_TIMESTAMP | TIMESTAMP | No nulls |
| ORDER_APPROVED_AT | TIMESTAMP | 160 nulls (0.16%) — expected for canceled/created orders |
| ORDER_DELIVERED_CARRIER_DATE | TIMESTAMP | 1,783 nulls (1.79%) — expected for non-shipped orders |
| ORDER_DELIVERED_CUSTOMER_DATE | TIMESTAMP | 2,965 nulls (2.98%) — expected for non-delivered orders |
| ORDER_ESTIMATED_DELIVERY_DATE | TIMESTAMP | No nulls |

**Status breakdown:** delivered 96,478 (97.02%), shipped 1,107, canceled 625, unavailable 609, invoiced 314, processing 301, created 5, approved 2

### ORDER_ITEMS (112,650 rows)
| Column | Type | Notes |
|---|---|---|
| ORDER_ID | VARCHAR | FK → ORDERS.ORDER_ID |
| ORDER_ITEM_ID | NUMBER | Sequential item number within order |
| PRODUCT_ID | VARCHAR | FK → PRODUCTS.PRODUCT_ID |
| SELLER_ID | VARCHAR | FK → SELLERS.SELLER_ID |
| SHIPPING_LIMIT_DATE | TIMESTAMP | No nulls |
| PRICE | FLOAT | Mean R$120.65, std R$183.63, range R$0.85–R$6,735.00 |
| FREIGHT_VALUE | FLOAT | Mean R$19.99, std R$15.81, range R$0.00–R$409.68 |

### PAYMENTS (103,886 rows)
| Column | Type | Notes |
|---|---|---|
| ORDER_ID | VARCHAR | FK → ORDERS.ORDER_ID |
| PAYMENT_SEQUENTIAL | NUMBER | Sequence within order (composite PK with ORDER_ID) |
| PAYMENT_TYPE | VARCHAR | credit_card, boleto, voucher, debit_card, not_defined |
| PAYMENT_INSTALLMENTS | NUMBER | Max 24 — normal for Brazilian parcelamento |
| PAYMENT_VALUE | FLOAT | Mean R$154.10, std R$217.49 |

**Payment type split:** credit_card 73.92%, boleto 19.04%, voucher 5.56%, debit_card 1.47%, not_defined 0.003%

### CUSTOMERS (99,441 rows)
| Column | Type | Notes |
|---|---|---|
| CUSTOMER_ID | VARCHAR | Primary key (order-level, not person-level) |
| CUSTOMER_UNIQUE_ID | VARCHAR | Person-level — 2,997 people placed multiple orders |
| CUSTOMER_ZIP_CODE_PREFIX | VARCHAR | 5 characters, always numeric |
| CUSTOMER_CITY | VARCHAR | Consistently lowercase |
| CUSTOMER_STATE | VARCHAR | All 27 valid Brazilian state abbreviations |

### SELLERS (3,095 rows)
| Column | Type | Notes |
|---|---|---|
| SELLER_ID | VARCHAR | Primary key — no duplicates, no nulls |
| SELLER_ZIP_CODE_PREFIX | VARCHAR | 5 characters |
| SELLER_CITY | VARCHAR | Consistently lowercase |
| SELLER_STATE | VARCHAR | Valid state abbreviations |

### PRODUCTS (32,951 rows)
| Column | Type | Notes |
|---|---|---|
| PRODUCT_ID | VARCHAR | Primary key |
| PRODUCT_CATEGORY_NAME | VARCHAR | 610 nulls (1.85%) — batch import failure |
| PRODUCT_NAME_LENGTH | NUMBER | 610 nulls — same 610 rows as category |
| PRODUCT_DESCRIPTION_LENGTH | NUMBER | 610 nulls |
| PRODUCT_PHOTOS_QTY | NUMBER | 610 nulls |
| PRODUCT_WEIGHT_G | NUMBER | 2 nulls; mean 2,276.5g, max 40,425g |
| PRODUCT_LENGTH_CM | NUMBER | 2 nulls; mean 30.8cm |
| PRODUCT_HEIGHT_CM | NUMBER | 2 nulls; mean 16.9cm |
| PRODUCT_WIDTH_CM | NUMBER | 2 nulls; mean 23.2cm |

### REVIEWS (99,224 raw → 98,410 cleaned rows)
| Column | Type | Notes |
|---|---|---|
| REVIEW_ID | VARCHAR | Should be unique — was reused across ORDER_IDs (see known issues) |
| ORDER_ID | VARCHAR | FK → ORDERS.ORDER_ID |
| REVIEW_SCORE | NUMBER | 1–5, no out-of-range values |
| REVIEW_COMMENT_TITLE | VARCHAR | 87,658 nulls (88.34%) — score-only reviews are normal |
| REVIEW_COMMENT_MESSAGE | VARCHAR | 58,256 nulls (58.71%) — normal |
| REVIEW_CREATION_DATE | TIMESTAMP | No nulls |
| REVIEW_ANSWER_TIMESTAMP | TIMESTAMP | No nulls |

**Score distribution:** 5★ 57.8%, 4★ 19.3%, 3★ 8.2%, 2★ 3.2%, 1★ 11.5% | Overall average: 4.09/5.00

### GEOLOCATION (1,000,163 raw → 738,332 cleaned rows)
| Column | Type | Notes |
|---|---|---|
| GEOLOCATION_ZIP_CODE_PREFIX | VARCHAR | 5 characters |
| GEOLOCATION_LAT | FLOAT | Mean −21.18 |
| GEOLOCATION_LNG | FLOAT | Mean −46.39 |
| GEOLOCATION_CITY | VARCHAR | No nulls |
| GEOLOCATION_STATE | VARCHAR | No nulls |

### CATEGORY_TRANSLATION (72 rows)
| Column | Type |
|---|---|
| PRODUCT_CATEGORY_NAME | VARCHAR |
| PRODUCT_CATEGORY_NAME_ENGLISH | VARCHAR |

**Important:** 2 Portuguese categories have no entry in this table. Always manually map them:
- `portateis_cozinha_e_preparadores_de_alimentos` → "Portable Kitchen Appliances"
- `pc_gamer` → "Gaming PCs"

---

## Table Relationships (Join Guide)

```
CUSTOMERS (1) ──── (N) ORDERS (1) ──── (N) ORDER_ITEMS (N) ──── (1) PRODUCTS
                                │                   │
                                │                   └──── (N) ──── (1) SELLERS
                                │
                           (1) ──── (N) PAYMENTS
                                │
                           (1) ──── (N) REVIEWS
```

**Standard join pattern for revenue analysis:**
```sql
ORDERS
  JOIN ORDER_ITEMS ON ORDERS.ORDER_ID = ORDER_ITEMS.ORDER_ID
  JOIN PRODUCTS    ON ORDER_ITEMS.PRODUCT_ID = PRODUCTS.PRODUCT_ID
  JOIN SELLERS     ON ORDER_ITEMS.SELLER_ID = SELLERS.SELLER_ID
  LEFT JOIN CATEGORY_TRANSLATION
                   ON PRODUCTS.PRODUCT_CATEGORY_NAME = CATEGORY_TRANSLATION.PRODUCT_CATEGORY_NAME
WHERE ORDERS.ORDER_STATUS = 'delivered'
```

**Revenue calculation:** Use `PRICE + FREIGHT_VALUE` from ORDER_ITEMS (not PAYMENT_VALUE from PAYMENTS — slight differences due to rounding/fees).

**Delivery time calculation:** `ORDER_DELIVERED_CUSTOMER_DATE - ORDER_DELIVERED_CARRIER_DATE` in calendar days.

**Always filter:** `WHERE ORDER_STATUS = 'delivered'` for revenue, delivery, and satisfaction analyses.

---

## Known Data Issues (Do Not Re-investigate)

### Critical — Fixed in Cleaned CSVs
| Issue | Table | Detail |
|---|---|---|
| 261,831 exact duplicate rows | GEOLOCATION | 26.2% of raw table. All 5 columns identical. Removed in cleaned CSV. |
| 814 duplicate REVIEW_IDs | REVIEWS | 789 REVIEW_IDs reused across multiple ORDER_IDs — pipeline bug. Earliest kept. |
| Embedded `\n`/`\r` in comments | REVIEWS | REVIEW_COMMENT_MESSAGE had 2,710 rows with newlines. Replaced with space. |
| Whitespace in comment fields | REVIEWS | 23 titles + 4,276 messages had leading/trailing whitespace. Stripped. |

### Flagged — Present in Cleaned CSVs (Flag Columns Added)
| Issue | Table | Column | Flag Column | Count |
|---|---|---|---|---|
| Coordinates outside Brazil | GEOLOCATION | LAT/LNG | GEOLOCATION_COORDS_FLAG | 31 rows |
| Impossible delivery sequence | ORDERS | Delivered timestamps | ORDER_DELIVERY_SEQUENCE_FLAG | 23 rows |
| Price outliers (>3 std, >R$671.56) | ORDER_ITEMS | PRICE | PRICE_OUTLIER_FLAG | 1,966 rows |
| Freight outliers (>3 std, >R$67.41) | ORDER_ITEMS | FREIGHT_VALUE | FREIGHT_OUTLIER_FLAG | 2,041 rows |
| Zero payment value | PAYMENTS | PAYMENT_VALUE | PAYMENT_VALUE_FLAG | 9 rows |
| Invalid installment count (≤0) | PAYMENTS | PAYMENT_INSTALLMENTS | PAYMENT_INSTALLMENTS_FLAG | 2 rows |
| Payment outliers (>R$806.58) | PAYMENTS | PAYMENT_VALUE | PAYMENT_VALUE_OUTLIER_FLAG | 1,803 rows |
| Category not in translation table | PRODUCTS | PRODUCT_CATEGORY_NAME | CATEGORY_IN_TRANSLATION | 13 rows |
| Weight outliers (>15,122.6g) | PRODUCTS | PRODUCT_WEIGHT_G | PRODUCT_WEIGHT_G_OUTLIER_FLAG | 961 rows |
| Length outliers (>81.6cm) | PRODUCTS | PRODUCT_LENGTH_CM | PRODUCT_LENGTH_CM_OUTLIER_FLAG | 616 rows |
| Height outliers (>57.9cm) | PRODUCTS | PRODUCT_HEIGHT_CM | PRODUCT_HEIGHT_CM_OUTLIER_FLAG | 728 rows |
| Width outliers (>59.4cm) | PRODUCTS | PRODUCT_WIDTH_CM | PRODUCT_WIDTH_CM_OUTLIER_FLAG | 574 rows |
| Zero physical dimension | PRODUCTS | Weight/dimensions | ZERO_DIMENSION_FLAG | 4 rows |

### Documented — Expected Business Behavior (No Action Needed)
- **REVIEWS: 88.34% null titles, 58.71% null messages** — score-only reviews are normal on this platform
- **CUSTOMERS: 2,997 repeat CUSTOMER_UNIQUE_IDs** — same person, multiple orders. Expected.
- **ORDERS: 775 orders with no ORDER_ITEMS** — 603 are `unavailable`, 164 `canceled`. Expected for non-fulfilled orders.
- **ORDERS: 1 delivered order with no PAYMENTS record** — single anomaly; exclude from payment analyses.
- **PRODUCTS: 610 rows with null metadata** — batch import failure; real products but no catalogue info.
- **PAYMENTS: 3 rows with `not_defined` type** — all have R$0 value; likely system/test artefacts. Exclude from payment analyses.

### Referential Integrity Status
| Foreign Key | Status |
|---|---|
| ORDERS.CUSTOMER_ID → CUSTOMERS | PASS (0 orphans) |
| ORDER_ITEMS.ORDER_ID → ORDERS | PASS (0 orphans) |
| ORDER_ITEMS.PRODUCT_ID → PRODUCTS | PASS (0 orphans) |
| ORDER_ITEMS.SELLER_ID → SELLERS | PASS (0 orphans) |
| PAYMENTS.ORDER_ID → ORDERS | PASS (0 orphans) |
| REVIEWS.ORDER_ID → ORDERS | PASS (0 orphans) |

---

## Key Business Metrics (Established Benchmarks)

| Metric | Value |
|---|---|
| Total delivered orders | 96,478 |
| Total revenue (Jan 2017–Aug 2018) | R$ 15,375,875.44 |
| Average order value (AOV) | R$ 159.86 |
| Average MoM revenue growth | 15.0% |
| Peak revenue month | Nov 2017 — R$ 1,153,528.05 |
| On-time delivery rate | 91.9% |
| Average actual delivery time | 9.3 days |
| Average estimated delivery time | 20.5 days (conservative) |
| Average delay when late | 9.6 days |
| Platform-wide avg review score | 4.09 / 5.00 |
| Active sellers | 2,970 |
| Top seller revenue | R$ 247,007.06 |

---

## Key Patterns and Discoveries

### Delivery → Satisfaction Link (Most Important Finding)
- Pearson r = **−0.316** between delivery days and review score
- 5-star orders: avg **7.8 days** delivery | 1-star orders: avg **16.4 days** delivery
- An 8.6-day gap separates the best and worst customer experiences
- Every day removed from delivery time measurably shifts review scores upward

### Geographic Concentration
- SP + RJ + MG = **66.5%** of all orders
- São Paulo alone: 40,501 orders, R$ 5,770,266.19
- Brazil's northeast and north are significantly under-served

### Payment Behavior
- Brazilian parcelamento is real: credit card buyers average **3.5 installments**
- Boleto AOV (R$145.03) is lower than credit card (R$163.32) — unbanked segment buys smaller
- Pix did not exist during this dataset's period (launched 2020) — absent by definition

### Review Polarisation
- Reviews are bimodal: 57.8% are 5★, 11.5% are 1★ — thin middle
- Delivery time is the dominant driver of low scores, not product quality

### Seller Quality vs. Volume
- Top-10 sellers by revenue average review score **4.01** — below platform mean of **4.08**
- High revenue and high satisfaction are not correlated at the seller level

### AOV Stability
- AOV ranged only R$ 23.78 across 20 months (R$146.28 – R$170.06)
- Revenue growth is volume-driven, not price/discount-driven

### GEOLOCATION Table — Use with Caution
- The raw table was 26.2% duplicate rows — always use the cleaned CSV
- 31 coordinate pairs have incorrect GPS (point to Europe or USA despite valid Brazilian zip codes)
- Filter on `GEOLOCATION_COORDS_FLAG = 'OK'` for any geographic distance calculation

---

## Analysis Conventions

- **Revenue:** Always `PRICE + FREIGHT_VALUE` from ORDER_ITEMS (not PAYMENTS.PAYMENT_VALUE)
- **Delivered orders only:** Filter `ORDER_STATUS = 'delivered'` for all business analyses
- **Exclude from delivery analysis:** The 23 orders flagged `CUSTOMER_DELIVERED_BEFORE_CARRIER`
- **Date window:** Use January 2017 – August 2018 for trend analyses (full months only)
- **Category names:** Always translate Portuguese → English; manually map the 2 missing categories
- **Currency:** All values in Brazilian Real (R$), formatted with commas (e.g., R$ 1,412,089.53)
- **Monetary formatting:** Use Brazilian Real symbol R$ with comma thousands separator

---

## Agents Run (History)

| Date | Agent | Output |
|---|---|---|
| 2026-03-27 | data-quality-agent | `outputs/reports/data_quality_report.md`, `outputs/clean_olist_data.py`, 9 cleaned CSVs |
| 2026-03-27 | insight-generator-agent | `outputs/reports/insights_report.md`, `outputs/olist_insights_analysis.py`, 8 charts |
| 2026-03-27 | (manual compilation) | `outputs/reports/final_report.md` |
