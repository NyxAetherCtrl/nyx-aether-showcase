# Data Science & Intelligence Portfolio

**Two production data products spanning probabilistic forecasting, NLP, consumer intelligence, experimentation, and data engineering.**

Built and operated end-to-end by **Samuel Choi**. The production source repositories remain private; this public repository is the portfolio layer — architecture, methodology, evaluation discipline, product evidence, and the data-science decisions that matter.

| Project | Data-science focus | Product |
|---|---|---|
| ⚾ **[NYX Aether — Baseball Intelligence](projects/baseball/README.md)** | Predictive ML · probabilistic modeling · temporal validation · calibration · statistical inference · champion–challenger evaluation · MLOps | [Live product](https://baseball.nyx-aether.com) |
| ◇ **[Beauty Intelligence](projects/beauty-intelligence/README.md)** | NLP / information extraction · taxonomy engineering · sentiment & complaint intelligence · human-ground-truth evaluation · time-series analytics · B2B data product | [Live product](https://beauty-intelligence-seven.vercel.app) |

---

## Why these two projects belong together

The domains are intentionally different. Baseball tests whether I can build a **forward-looking probabilistic model** without leaking the future. Beauty Intelligence tests whether I can turn **unstructured consumer language** into defensible product and market signals.

The common workflow is the part I want a hiring team to see:

```text
raw external data
      ↓
data acquisition + quality controls
      ↓
feature / semantic representation
      ↓
model or analytical engine
      ↓
evaluation with explicit denominators
      ↓
versioned production pipeline
      ↓
user-facing decision product
      ↓
monitoring, error analysis, iteration
```

These are not notebook-only demos. Both projects connect data methodology to a production product and both treat **measurement integrity as part of the model**, not as documentation added afterward.

---

## Data Science skill map

| Skill | Baseball Intelligence | Beauty Intelligence |
|---|---|---|
| **Problem framing** | Estimate pregame MLB win probability and explain uncertainty | Convert customer reviews into product, complaint, brand, and market intelligence |
| **Supervised ML** | Binary classification with L2-regularized logistic regression | — |
| **NLP / Information Extraction** | — | Multi-label concept extraction from review text with evidence spans |
| **Feature Engineering** | Rolling form, run differential, season strength, rest, home-field and point-in-time signals | Concept taxonomy, phrase/context rules, sentiment/assertion states, review-level denominators |
| **Temporal Validation** | Walk-forward / out-of-time train → validation → test protocol | Time-indexed review aggregation and trend windows |
| **Leakage Prevention** | Point-in-time reconstruction, immutable pregame records, ingest-time controls | Versioned semantic tables, separate evaluation data, no prediction exposure during blind annotation |
| **Probability Calibration** | Platt scaling, isotonic regression, identity calibration | — |
| **Statistical Inference** | Bootstrap CIs, paired bootstrap, cluster-aware uncertainty, sign testing | Error slices and uncertainty-aware evaluation design rather than unqualified accuracy claims |
| **Experiment / Version Evaluation** | Champion–challenger paired benchmarking: Engine 2.2 vs 3.0 | Side-by-side V2/V3 semantic engine rollout with feature gates and rollback boundaries |
| **Human Ground Truth** | Final game outcomes provide objective labels | Blind human annotation pipeline with representative + challenge cohorts |
| **Evaluation Metrics** | Log loss, Brier, AUC, ECE, accuracy | Concept detection, span IoU/exact match, sentiment/assertion accuracy & macro-F1, confusion matrices, strict end-to-end metrics |
| **Sampling / Cohort Design** | Strict paired-game eligibility and date-aware evaluation cohorts | Deterministic proportional stratification plus targeted challenge-set sampling |
| **Time-Series Analytics** | Rolling historical windows and prospective grading | Monthly concept/sentiment rollups, emerging-signal and complaint trend analysis |
| **Metric Design** | Proper scoring rules and cohort-matched comparison | Explicit review denominators, support thresholds, typed unavailable/low-sample states |
| **Data Engineering** | Multi-source ingestion, append-only history, deterministic paging, reconciliation | Multi-retailer collection, incremental tagging, dirty-month rollups, idempotent recovery |
| **Data Quality** | Point-in-time audits, pagination regression guards, fail-closed publication | Denominator tests, read-failure vs true-empty states, crawler recovery, semantic drift guards |
| **Model / Engine Governance** | Frozen protocols, content-addressed artifacts, test-set access controls | Versioned taxonomy, frozen evaluation identity, blind predictions, read/write feature gates |
| **Production ML / MLOps** | Shadow candidate, official champion, scheduled scoring, monitoring, rollback | Offline tagging + persisted semantic layer + production serving, V2/V3 isolation and rollback |
| **Product Analytics** | Public forecast history and market-context comparison | Brand intelligence, peer benchmarking, product complaint radar, consumer-love metrics |

### Skills demonstrated across the portfolio

**Machine Learning & Statistics**  
Binary Classification · Logistic Regression · L2 Regularization · Probability Calibration · Platt Scaling · Isotonic Regression · Feature Engineering · Model Selection · Bootstrap Confidence Intervals · Paired Model Evaluation · Statistical Significance · Error Analysis

**Experimentation & Evaluation**  
Champion–Challenger Testing · Walk-Forward Validation · Out-of-Time Testing · Prospective Validation · Holdout Governance · Stratified Sampling · Challenge Sets · Human-in-the-Loop Evaluation · Macro-F1 · Confusion Matrices · Span IoU

**NLP & Consumer Intelligence**  
Information Extraction · Multi-Label Concept Tagging · Taxonomy / Ontology Design · Negation & Context Handling · Sentiment Analysis · Assertion Classification · Evidence Extraction · Complaint Intelligence · Trend Detection

**Data Engineering & Analytics Engineering**  
Python · SQL · PostgreSQL · Supabase · ETL / ELT · Incremental Pipelines · Idempotency · Data Lineage · Point-in-Time Data · Data Quality Testing · Metric Semantics · Denominator Design · Time-Series Aggregation

**Production & Product**  
MLOps · Model Versioning · Shadow Deployment · Feature Gating · CI/CD · GitHub Actions · Cloudflare Workers · Vercel · Next.js · TypeScript · Monitoring · Fail-Closed Systems · Reproducible Pipelines

> **Deliberate non-claim:** the Baseball champion–challenger benchmark is not presented as an A/B test because games were not randomly assigned to treatments. Beauty Intelligence does not present semantic “accuracy” as human-validated until the blind ground-truth process supports that claim. Correct experiment labeling is part of the work.

---

# Project 1 — NYX Aether Baseball Intelligence

[**Open the Baseball case study →**](projects/baseball/README.md)

A production MLB analytics and forecasting platform built around one difficult requirement: **a prediction must be reproducible using only information that was available at prediction time.**

### Data-science highlights

- Built a probabilistic binary-classification pipeline for MLB game outcomes.
- Designed **point-in-time feature engineering** and walk-forward validation to prevent look-ahead bias.
- Separated model training from **probability calibration** and compared identity, Platt, and isotonic calibration using validation-only evidence.
- Evaluated predictions with proper scoring rules — **log loss and Brier score** — alongside AUC, ECE, and accuracy.
- Built a **paired champion–challenger benchmark** so Engine 2.2 and Engine 3.0 are compared on the exact same games.
- Quantified uncertainty with bootstrap confidence intervals rather than promoting a model from a point estimate alone.
- Operated prospective prediction capture, grading, versioning, monitoring, rollback, and cloud scheduling.

### One result worth discussing in an interview

The validated strict historical benchmark contained **120 paired games across 9 dates**. Engine 3.0 improved log loss and Brier score directionally, but the paired 95% confidence intervals crossed zero, so the published conclusion remained **“directionally better, not statistically established.”** The benchmark also caught a nondeterministic pagination defect in its own evaluation harness, invalidated the first run, fixed the data-ordering contract, and reran from scratch.

That is a better representation of the project than “my model got a higher accuracy.”

![NYX Aether Model History](assets/screenshots/model-history.png)

**Deep dives:** [Architecture](docs/ARCHITECTURE.md) · [Technical case study](docs/TECHNICAL_CASE_STUDY.md) · [Data engineering](docs/DATA_ENGINEERING.md) · [Model evaluation](docs/MODEL_EVALUATION.md) · [Project evolution](docs/PROJECT_EVOLUTION.md)

---

# Project 2 — Beauty Intelligence

[**Open the Beauty Intelligence case study →**](projects/beauty-intelligence/README.md)

A production consumer-review intelligence platform that turns large volumes of unstructured beauty reviews into structured concepts, sentiment, complaint evidence, product signals, brand comparisons, and longitudinal trends.

### Data-science highlights

- Designed a domain taxonomy and **multi-label concept extraction** layer for beauty-review language.
- Built semantic handling for phrase context, negation, assertion state, and sentiment instead of treating keyword matches as truth.
- Materialized review-level concept mentions and monthly time-series rollups so analytics requests do not rerun NLP on every page load.
- Built complaint ranking, brand/product intelligence, comparable-peer selection, and explicit metric-denominator rules.
- Designed a **blind human-ground-truth evaluation system**: predictions are hidden from annotators, representative and challenge cohorts are separated, and evaluation aligns predicted/human evidence spans before scoring.
- Evaluation supports concept detection, span exactness/IoU, sentiment and assertion accuracy, macro-F1, per-class metrics, confusion matrices, and strict end-to-end correctness.
- Built versioned V2/V3 data paths and feature gates so a new semantic engine can be evaluated without silently replacing the incumbent.

### Why this matters for Data Science

This project demonstrates a different side of DS from Baseball: the hard problem is not predicting a clean binary label. It is defining **what should count**, building a reproducible semantic representation, creating ground truth, protecting denominators, distinguishing “no signal” from “query failed,” and translating noisy language into business decisions without overstating model quality.

---

## Tech stack

**Python** · **SQL** · **PostgreSQL / Supabase** · **pandas** · **scikit-learn** · **pytest** · **TypeScript** · **Next.js / React** · **GitHub Actions** · **Cloudflare Workers** · **Vercel**

The two systems use different subsets of the stack; the project case studies distinguish the modeling and production methods actually used in each one.

---

## Source availability

The full production repositories are private. They include operational configuration, internal runbooks, model/engine internals, raw-data handling, and deployment machinery that are not necessary for portfolio review.

This public showcase intentionally exposes the parts that are useful for technical evaluation:

- system architecture and data flow;
- modeling / semantic methodology;
- evaluation design and limitations;
- statistical reasoning and metric definitions;
- representative production incidents and how they changed the system;
- screenshots and live products;
- evidence of reproducibility, testing, and governance.

This is not intended to be an open-source distribution of either product.

---

## About

Designed, built, and operated by **Samuel Choi**. I am using these projects to demonstrate end-to-end capability across **data science, product analytics, analytics engineering, and ML-adjacent data products** — from raw data and methodology through validation and production delivery.

AI coding tools were used as implementation accelerators. Problem framing, architecture, data definitions, evaluation policy, acceptance criteria, and production decisions are represented here as explicit, testable system contracts rather than as tool-generated claims.
