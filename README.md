# 🩺 Healthcare Claims Risk Stratification  
**Snowflake • SQL • Linear (Logistic) Regression**

---

## Overview

This project implements an **end-to-end healthcare risk stratification pipeline** using medical claims data.  
The objective is to identify **members at risk of becoming high-cost in the next 6 months**, with an emphasis on:

- clinical complexity and comorbidities  
- high-cost procedures  
- temporal integrity (no leakage)  
- explainability using linear models  

All feature engineering is performed in **Snowflake using SQL**, and a **baseline logistic regression model** is trained and evaluated using **Snowflake Notebooks**.

This design mirrors how risk models are commonly built in payer and population health analytics environments.

---

## Problem Statement

> Given a rolling history of medical and pharmacy claims, can we predict which members will fall into the **top cost percentile** over the *next* 6 months?

Key constraints:
- Rare outcome (~2–3% prevalence)
- High risk of temporal leakage
- Need for **clinically interpretable features**
- Preference for **linear models** over black-box approaches

---

## Data Model & Grain

All modeling is performed at the **member-month** grain.

| Table | Purpose |
|-----|-----|
| `health_claims` | Cleaned claim-level data (dates, CPT, ICD-10, costs) |
| `member_month_spine` | Complete member × month backbone |
| `features_6m` | Rolling 6-month feature aggregates |
| `labels_6m` | Future 6-month cost labels |
| `model_dataset_split` | Time-based train/valid/test split |

---

## Feature Engineering (SQL in Snowflake)

Features are computed using a **rolling 6-month lookback window** and include:

### Utilization & Cost
- Total allowed and paid amounts
- Claim count
- Service day count
- Provider count
- Emergency department visit proxy
- Pharmacy vs medical spend

### Clinical Complexity
- Diagnosis code count
- CPT code count
- High-cost diagnosis indicators (based on ICD-10 descriptions)
- High-cost procedure indicators (line-level cost threshold)
- Share of spend attributable to high-cost diagnoses and procedures

### Procedure Explainability
To improve interpretability without altering modeling grain:
- Binary flag for presence of high-cost procedures
- **Single representative CPT** per member-month (highest allowed amount)
- Retained primarily for **review and explanation**, not as a core model feature

### Recency
- Days since last claim
- Explicitly encoded for members with no recent utilization

All features are **null-safe**, leakage-aware, and production-oriented.

---

## Label Definition

A binary label is assigned at each member-month:

- **1** = Member falls into the top cost percentile in the *following* 6 months  
- **0** = Otherwise  

This produces a realistic **rare-event classification problem** with ~2.6% positive rate.

---

## Train / Validation / Test Split

A **strict time-based split** is used to prevent leakage:

- **TRAIN:** oldest months  
- **VALID:** intermediate months  
- **TEST:** most recent months  

Random splitting is intentionally avoided.

---

## Modeling Approach

### Model
- **Logistic Regression**
- Class-weighted to handle imbalance
- Numeric features only
- String fields (CPT / diagnosis descriptions) excluded from training

This approach prioritizes:
- interpretability  
- stability  
- clinical trust  

over marginal performance gains from more complex models.

---

## Model Performance

| Metric | Validation | Test |
|-----|-----|-----|
| AUC | ~0.62 | ~0.67 |
| Precision @ 2.6% | ~3.4% | **~8.4%** |
| Lift @ Top Risk | — | **~3.2×** |

**Interpretation:**
- The model is over **3× more precise than random selection**
- Performance is stable across time
- Results are realistic for a linear model on claims data

---

## Explainability & Review

Predictions are written back to Snowflake and enriched with:

- Prior utilization metrics
- High-cost diagnosis and procedure indicators
- Human-readable diagnosis descriptions
- Representative high-cost CPTs (when present)

This enables:
- case-level review  
- care management targeting  
- clear articulation of *why* a member was flagged  

Members with no diagnoses or procedures in the lookback window are retained with explicit context to avoid misinterpretation.

---

## Design Principles

- SQL-first feature engineering  
- Strict temporal alignment  
- No data leakage  
- Linear model transparency  
- Separation of prediction vs explanation  
- Healthcare-realistic assumptions  

---

## What This Project Demonstrates

- Population health modeling judgment  
- Claims-based feature engineering  
- Time-aware ML workflows  
- Explainability without leakage  
- Production-style analytics design  

---

## Tech Stack

- Snowflake (SQL, Snowflake Notebooks)
- Python (pandas, scikit-learn)
