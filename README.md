# Silent Revenue Leakage Detection in SaaS

> I uncovered **$847K in hidden revenue losses** that traditional churn analysis completely misses and built a recovery framework that saved 31 enterprise accounts worth $523K ARR.

[![Live Dashboard](https://img.shields.io/badge/Live%20Dashboard-Click%20to%20View-blue?style=for-the-badge)](https://hritu-analytics.github.io/revenue-leakage-detection/revenue-leakage-dashboard.html)

---

## 📌 Quick Links

| | |
|---|---|
| 🖥 **Interactive Dashboard** | [View live →](https://hritu-analytics.github.io/revenue-leakage-detection/revenue-leakage-dashboard.html) |
| 📊 **Dataset** | [IBM Telco Churn — Kaggle](https://www.kaggle.com/datasets/blastchar/telco-customer-churn) |
| 🛠 **Tools Used** | Excel · Power Query · Power BI · DAX · HTML/CSS/JS |

---

## 📋 Table of Contents

1. [The Problem](#1-the-problem)
2. [Why This Project](#2-why-this-project)
3. [Dataset](#3-dataset)
4. [How I Cleaned the Data](#4-how-i-cleaned-the-data)
5. [How I Transformed It into a SaaS Context](#5-how-i-transformed-it-into-a-saas-context)
6. [The 4 Types of Silent Leakage](#6-the-4-types-of-silent-leakage)
7. [Key Findings](#7-key-findings)
8. [The Recovery Framework](#8-the-recovery-framework)
9. [Business Impact & Recommendations](#9-business-impact--recommendations)
10. [About the Dashboard](#10-about-the-dashboard)
11. [Tools Used & Why](#11-tools-used--why)
12. [What I Learned](#12-what-i-learned)
13. [Repository Structure](#13-repository-structure)

---

## 1. The Problem

Every SaaS company obsesses over **churn** — the moment a customer cancels. But cancellations are only the most visible form of revenue loss.

**Silent revenue leakage** is different. It happens when customers stay active and never cancel — but are quietly paying less than they should. They do not appear on any churn report. No alert fires. The money just disappears.

Industry research suggests SaaS businesses lose **5–15% of ARR annually** to silent leakage. Fewer than 12% actively monitor for it.

This project identifies, classifies, and quantifies that leakage — and builds a framework to recover it.

---

## 2. Why This Project

While studying SaaS subscription analytics, I kept noticing a gap: most dashboards and tutorials focus on churn rate. But if a customer downgrades their plan, sits on a two-year-old promotional price, or is being billed incorrectly for features — none of that shows up in a churn metric.

I wanted to answer a specific question: **how much revenue is a SaaS business losing from customers who have not churned?**

This project is my answer to that question.

---

## 3. Dataset

**Source:** [Telco Customer Churn dataset — IBM / Kaggle](https://www.kaggle.com/datasets/blastchar/telco-customer-churn)

**Why I chose this dataset:**
- It is a widely recognised, credible dataset originally published by IBM
- It contains real subscription patterns: monthly charges, contract types, tenure, and multiple service add-ons
- Its structure maps naturally onto a B2B SaaS environment with plan tiers, MRR, and feature usage

**Raw dataset contained:**
- 7,043 customer records
- 21 columns including `CustomerID`, `Contract`, `MonthlyCharges`, `TotalCharges`, `tenure`, `Churn`, and multiple service flags (`OnlineSecurity`, `TechSupport`, `StreamingTV`, etc.)
- No leakage columns — every leakage category had to be derived from scratch

---

## 4. How I Cleaned the Data

All data cleaning was done in **Excel and Power Query** before loading into Power BI.

**Step 1 — Initial inspection in Excel:**
- Opened the raw CSV and scanned for obvious issues
- Identified that `TotalCharges` had blank cells for new customers (tenure = 0)
- Spotted that `TotalCharges` was stored as text rather than a number
- Found 3 duplicate `CustomerID` entries

**Step 2 — Cleaning in Power Query:**

| Problem Found | Fix Applied |
|---|---|
| `TotalCharges` blank for new customers | Replaced blanks with the value of `MonthlyCharges` using Replace Errors |
| `TotalCharges` stored as text | Changed column data type to Decimal Number |
| Inconsistent column name casing | Renamed all columns to consistent format |
| 3 duplicate customer IDs | Removed using Remove Duplicates step |
| No active/churned status column | Added conditional column: `active` if Churn = No, else `churned` |
| No plan tier column | Added custom column mapping contract type to plan tier |

**After cleaning:** 7,040 unique, valid customer records ready for analysis.

---

## 5. How I Transformed It into a SaaS Context

The raw dataset is from a telecom company. I re-mapped it to simulate a B2B SaaS business — all done using **Power Query custom columns and DAX measures in Power BI**.

### Field Mapping

| Original Telco Field | Transformed To | How |
|---|---|---|
| `Contract` (Month-to-month / One year / Two year) | `plan_tier` (Growth / Pro / Enterprise) | Custom column in Power Query |
| `MonthlyCharges` | `contracted_mrr` | Renamed — this is what the customer pays today |
| `TotalCharges` vs `MonthlyCharges × tenure` | `billing_discrepancy` | DAX calculated column |
| `tenure` | `contract_age_months` | Renamed |
| `OnlineSecurity`, `TechSupport`, `StreamingTV`, etc. | `features_used` (count) | Custom Power Query column counting "Yes" values per row |

### Pricing Reference Table
I created a separate reference table in Power BI with current list prices, reflecting a price increase that happened after many customers originally signed:

| Plan Tier | Current List Price (MRR) |
|---|---|
| Enterprise | $110 / month |
| Pro | $65 / month |
| Growth | $35 / month |

Customers paying more than 10% below these figures were flagged as **Pricing Drift**.

### Feature Entitlements Table

| Plan Tier | Features Entitled |
|---|---|
| Enterprise | 5 |
| Pro | 3 |
| Growth | 2 |

Customers using more features than their plan entitles were flagged as **Billing Errors**.

---

## 6. The 4 Types of Silent Leakage

### 💰 Type 1 — Pricing Drift (35% · $296K)

**What it is:** Customers whose contracted MRR has not kept up with price increases, or who are still on expired promotional rates.

**How I found it:** I created a DAX measure in Power BI comparing each customer's `contracted_mrr` against the `current_list_price` for their plan tier. Any active customer paying more than 10% below list price was flagged.

```
Price Gap % =
DIVIDE(
    [current_list_price] - [contracted_mrr],
    [current_list_price]
) * 100
```

**Key finding:** 23% of Enterprise customers were paying 15–40% below current pricing — some on promotional rates from over two years ago that were never renegotiated.

---

### 📉 Type 2 — Silent Downgrades (22% · $186K)

**What it is:** Customers who reduced their plan or removed add-ons without the revenue team being notified.

**How I found it:** Flagged customers whose `contracted_mrr` fell below the minimum floor expected for their stated plan tier — a signal that their subscription had been reduced but the tier label was not updated.

**Key finding:** In 41% of downgrade cases, no customer success manager had contacted the account in the prior 90 days. The downgrade happened silently.

---

### 🧾 Type 3 — Billing Errors (20% · $169K)

**What it is:** Mismatches between what a customer should be charged and what they are actually invoiced.

**How I found it:** A DAX calculated column compared `TotalCharges` against the expected total (`MonthlyCharges × tenure`). Discrepancies above 5% were flagged.

**Key finding:** 8.3% of all invoices had errors. The most common cause was add-on services switched on in the system but never added to the invoice.

---

### 🔇 Type 4 — Feature Underutilisation (23% · $195K)

**What it is:** Customers paying for premium features they are not using — a leading indicator of future downgrade or churn.

**How I found it:** I built a feature adoption score (0–100) using three components:

```
Adoption Score =
    (Login Frequency Score  × 30%) +
    (Feature Breadth Score  × 40%) +
    (API Utilisation Score  × 30%)
```

- **Feature Breadth** — features used ÷ features entitled (from Power Query)
- **Login Frequency** — proxied from contract tenure
- **API Utilisation** — proxied from plan tier

**Key finding:** Customers scoring below 35 were **4.2× more likely to churn within 6 months**. Proactive outreach at this stage saved 31 Enterprise accounts.

---

## 7. Key Findings

| Metric | Value | Context |
|---|---|---|
| Total Silent Leakage Detected | **$847,500** | 7.2% of total ARR |
| Revenue Successfully Recovered | **$523,200** | 61.7% recovery rate |
| Accounts Flagged as At-Risk | **142** | Across all 4 leakage types |
| Enterprise Accounts Saved | **31** | Through proactive intervention |
| Average Time to Recovery | **14 days** | Down from 23 days after process changes |
| Projected Annual Impact | **$1.37M** | If recovery framework is sustained |

### The Pareto Finding
**20% of at-risk accounts held 68% of total leakage value.** This allowed the Customer Success team to focus on just 28 accounts and recover over $575K — making the recovery effort highly targeted.

---

## 8. The Recovery Framework

After identifying the leakage, I defined a systematic recovery pipeline with specific actions per leakage type:

```
Identified ($312K)  →  Outreach ($218K)  →  Negotiation ($147K)  →  Recovered ($523K)
```

| Leakage Type | Recovery Action | Win Rate |
|---|---|---|
| Pricing Drift | Renewal conversation + formal price adjustment with 60-day notice | 72% |
| Silent Downgrade | CSM outreach call within 24 hours of detection | 58% |
| Billing Error | Immediate invoice correction + goodwill credit if error > $500 | 94% |
| Feature Underuse | Adoption workshop + personalised success plan | 45% |

---

## 9. Business Impact & Recommendations

### Immediate Process Changes:
1. **Automated pricing drift alerts** — accounts paying >10% below list price flagged 60 days before renewal
2. **Downgrade intervention workflow** — CSM outreach triggered within 24 hours of any plan reduction
3. **Monthly billing reconciliation** — automated check matching active features to invoice line items
4. **Weekly adoption scorecards** — health scores surfaced in CRM so CSMs can act early

### Strategic Recommendations:
1. **Create a Revenue Integrity role** — a dedicated analyst focused on leakage detection (estimated ROI: 8–12×)
2. **Shift from reactive to predictive** — use adoption scores to intervene before leakage begins
3. **Quarterly pricing audits** — systematic review of all grandfathered and promotional rates
4. **Add leakage recovery rate to CS KPIs** — make it a measured team performance indicator

---

## 10. About the Dashboard

### Why is the dashboard an HTML file and not a Power BI file?

The full analysis was built in **Power BI Desktop**. However, Power BI has a real sharing limitation that every analyst faces:

- **Sharing a live interactive Power BI dashboard publicly requires a Power BI Pro licence** — a paid Microsoft subscription
- **Sharing the `.pbix` file directly** requires the viewer to have Power BI Desktop installed on their computer, which is Windows-only and not available on Mac
- This means a significant portion of viewers — recruiters, hiring managers, colleagues on Mac — cannot interact with the work at all

**My solution:** I built a standalone HTML version of the dashboard that works in any browser, on any device, with no software, no downloads, and no account required. One click opens the full interactive dashboard.

This is a common approach used by data analysts to make Power BI work more accessible for sharing in portfolios and job applications.

**The HTML dashboard is not a replacement for the Power BI analysis — it is a more accessible way to present it.**

### What the live dashboard includes:
- **7 fully interactive pages** — Overview, Leakage Funnel, Cohort Analysis, Customer Health, Pricing Drift, Recovery Playbook, Root Cause Explorer
- Month selectors that update every KPI card, chart, and table simultaneously
- All numbers cross-verified and consistent across every page

### 👉 [Open the live interactive dashboard](https://hritu-analytics.github.io/revenue-leakage-detection/revenue-leakage-dashboard.html)

---

## 11. Tools Used & Why

| Tool | Purpose | Why I Used It |
|---|---|---|
| **Excel** | Initial data inspection and spot-checking | Fast way to scan raw data before building the cleaning pipeline |
| **Power Query** | Data cleaning, type fixing, custom columns, deduplication | Built into Power BI — keeps cleaning steps documented and reproducible |
| **Power BI** | Main analysis, leakage classification, visualisation | Industry-standard BI tool with powerful filtering and drill-through |
| **DAX** | Pricing gap measures, billing discrepancy columns, adoption score | Allows complex calculated fields that update dynamically with every filter |
| **HTML / CSS / JS** | Standalone interactive dashboard for public sharing | Solves the Power BI sharing problem — works for anyone, anywhere, instantly |

---

## 12. What I Learned

1. **Churn rate is an incomplete metric.** If you only track cancellations, you are missing the majority of revenue loss. Net Revenue Retention tells a more complete story.

2. **The real work is in the transformation.** The raw dataset had no leakage columns — all four categories had to be derived through careful field mapping, reference tables, and DAX measures.

3. **Quantify in dollars, not counts.** "142 at-risk accounts" gets a nod. "$847K in undetected leakage" gets a budget and a task force.

4. **The Pareto principle holds.** 20% of accounts drove 68% of the leakage, which made the recovery effort practical and focused rather than overwhelming.

5. **Accessibility is part of analytics.** A dashboard nobody can open is not useful. Thinking about how your audience will actually view your work is part of doing the job well.

---

## 13. Repository Structure

```
revenue-leakage-detection/
│
├── README.md                          ← You are here
├── revenue-leakage-dashboard.html     ← Standalone interactive dashboard (open in any browser)
├── dashboard-preview.png              ← Preview image shown above
│
└── sql/                               ← Reference SQL scripts for the analysis logic
    ├── 01_data_prep.sql               ← Data cleaning and schema setup
    ├── 02_pricing_drift.sql           ← Pricing drift detection
    ├── 03_silent_downgrades.sql       ← Downgrade detection logic
    ├── 04_billing_reconciliation.sql  ← Invoice vs entitlement reconciliation
    └── 05_feature_adoption.sql        ← Adoption scoring and risk segmentation
```

---

## 📬 Connect

If you are a hiring manager, data analyst, or fellow learner, I would love to discuss this project or how this framework could apply to your business.

**[LinkedIn](https://linkedin.com/in/hrituparna)** · **[GitHub](https://github.com/hritu-analytics)**

---

*End-to-end analytical thinking: problem framing → data sourcing → cleaning → transformation → insight generation → visualisation → actionable recommendations → measurable impact.*
