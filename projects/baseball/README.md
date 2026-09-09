# NYX Aether — Baseball Intelligence

**Predictive ML · Probabilistic Forecasting · Temporal Validation · Statistical Inference · Production ML**

[← Portfolio home](../../README.md) · [Live product](https://baseball.nyx-aether.com) · [Architecture](../../docs/ARCHITECTURE.md) · [Model evaluation](../../docs/MODEL_EVALUATION.md)

NYX Aether is a production MLB analytics and forecasting system. The core DS requirement is strict: **estimate win probability using only information available before first pitch, preserve the forecast, and evaluate it later without rewriting history.**

---

## 30-second hiring scan

**Best-fit roles:** Data Scientist · Applied Data Scientist · Product Data Scientist · ML-adjacent Data Scientist

**Core skills demonstrated:**

`Binary Classification` · `Logistic Regression` · `L2 Regularization` · `Feature Engineering` · `Walk-Forward Validation` · `Point-in-Time Data` · `Probability Calibration` · `Bootstrap Inference` · `Champion–Challenger Evaluation` · `MLOps`

**What makes it more than a modeling demo:** the model lives inside a production system with historical backtesting, prospective capture, immutable forecast records, cloud scheduling, model versioning, monitoring, and rollback.

---

## End-to-end DS workflow

```text
MLB / Statcast / contextual data
              ↓
point-in-time feature engineering
              ↓
model training + standardization
              ↓
probability calibration
              ↓
chronological validation
              ↓
champion–challenger comparison
              ↓
pregame forecast lock
              ↓
prospective grading + monitoring
              ↓
production analytics product
```

The emphasis is not “I trained a classifier.” It is **temporal correctness, probability quality, uncertainty, reproducibility, and production governance**.

---

## Data Science techniques used

| Area | Technique | How it is used |
|---|---|---|
| **Predictive Modeling** | Binary Classification | Predict MLB home-win probability rather than only a hard pick |
| | Logistic Regression | Interpretable probabilistic challenger model |
| | L2 Regularization | Reduces coefficient instability and overfitting |
| | Deterministic IRLS | Reference implementation fits reproducibly |
| **Feature Engineering** | Rolling features | Recent win rate and run differential |
| | Season-to-date strength | Built only from information available before the target game |
| | Rest / fatigue | Rest-day context as a point-in-time feature |
| | Home-field indicator | Explicit environmental/context feature |
| | Train-only standardization | Scaler statistics fit on train and frozen for later partitions |
| **Validation** | Walk-Forward Validation | Chronological train → validation → test; random split intentionally prohibited |
| | Out-of-Time Testing | Future season held out from fitting/calibration decisions |
| | Point-in-Time Reconstruction | Historical replay rebuilds what the model could have known at prediction time |
| | Look-Ahead Bias Prevention | Future outcomes and later facts are fenced from earlier predictions |
| | Holdout Governance | Test access is controlled to reduce repeated tuning against the holdout |
| **Probability Modeling** | Platt Scaling | Logistic recalibration on validation evidence |
| | Isotonic Regression | Monotonic non-parametric calibration |
| | OOF Calibration Selection | Candidate calibrators compared on out-of-fold validation predictions |
| **Evaluation** | Log Loss | Primary proper scoring rule for probability quality |
| | Brier Score | Squared probability error |
| | AUC | Ranking/discrimination quality |
| | ECE | Calibration reliability |
| | Accuracy | Secondary directional metric |
| **Statistical Inference** | Bootstrap Confidence Intervals | Quantifies metric uncertainty |
| | Paired Bootstrap | Engine 3.0 − Engine 2.2 differences on identical games |
| | Date-Cluster Bootstrap | Sensitivity to correlated/shared game-day conditions |
| | Sign Test | Non-parametric paired directional comparison |
| **Experimentation** | Champion–Challenger Testing | Incumbent Engine 2.2 vs Engine 3.0 under fixed rules |
| | Prospective Validation | Capture prediction first; grade only after outcome settlement |
| **MLOps** | Model/artifact versioning | Model + calibration identity is frozen and traceable |
| | Shadow evaluation | Challenger accumulates evidence without becoming production authority |
| | Monitoring / rollback | Production health and fallback paths are explicit |

---

## Validation design

Prediction Engine 3.0 uses a strict temporal split:

```text
2024                         2025                         2026
TRAIN                        VALIDATION                   TEST
  │                              │                         │
fit coefficients        model/calibration decisions      final comparison
fit scaler              OOF calibration evaluation       no fitting
```

Why this matters: random train/test splitting can create an unrealistically easy evaluation when the real task is forecasting the future. Here, **time is part of the experimental design**.

This demonstrates: **temporal validation, out-of-time testing, data-leakage prevention, reproducible feature construction, and holdout discipline**.

---

## Engine 2.2 vs Engine 3.0 — paired model experiment

This is intentionally **not described as an A/B test**. Games were not randomly assigned to treatments. Both engines score the same eligible games, so the correct framing is **paired model evaluation / champion–challenger testing**.

Validated strict historical benchmark:

| Metric | Engine 2.2 | Engine 3.0 V1 | Δ (3.0 − 2.2) |
|---|---:|---:|---:|
| **Log loss** | 0.700994 | **0.685063** | **−0.015931** |
| **Brier score** | 0.253664 | **0.246091** | **−0.007573** |
| **AUC** | 0.510558 | **0.556821** | +0.046263 |
| **ECE** | **0.036793** | 0.045429 | +0.008636 |
| **Accuracy** | **0.541667** | 0.533333 | −0.008334 |

Cohort: **120 paired 2026 games across 9 dates**.

Paired uncertainty:

- Log-loss Δ 95% CI: **[-0.044632, +0.013092]**
- Brier Δ 95% CI: **[-0.021630, +0.006606]**
- Sign test: **p = 0.201**

### Statistical conclusion

> Engine 3.0 is **directionally better** on log loss and Brier score in this cohort, but the improvement is **not statistically established** because the primary paired confidence intervals include zero.

That conclusion is less exciting than “Engine 3.0 wins,” but it is the defensible conclusion.

---

## A benchmark that invalidated itself

The initial comparison exposed nondeterministic predictions. Root cause: paginated database reads were not totally ordered, so repeated reads could produce different row sets around page boundaries.

The response was to:

1. invalidate the original result;
2. add deterministic ordering to paginated reads;
3. verify unique ordering keys;
4. rerun both engines from scratch;
5. reproduce the 120-game cohort across independent runs;
6. keep the statistical conclusion conservative.

This demonstrates **experimental-integrity debugging, data-quality validation, reproducibility, and willingness to invalidate a favorable result when the measurement system is wrong**.

---

## Probability calibration

For a user-facing forecast, “picked the winner” is not enough. A 70% prediction should behave like a 70% event over repeated observations.

The calibration layer compares:

- **identity / none** — raw model probability;
- **Platt scaling** — logistic transform;
- **isotonic regression** — monotonic non-parametric mapping.

Candidate calibrators are evaluated out-of-fold on validation data. Selection prioritizes log loss, then Brier score, then ECE, with simplicity as a tie-breaker. The test partition does not select the calibrator.

Skills demonstrated: **calibration analysis, proper scoring rules, OOF model selection, overfit prevention**.

---

## Feature engineering principles

A feature is not eligible simply because it correlates with outcomes. It must be reconstructable at prediction time.

Verified feature families include:

- recent team form;
- rolling run differential;
- season-to-date strength;
- rest days;
- home-field context.

Missing or unverifiable inputs are excluded rather than repaired with information that did not exist at the original prediction timestamp.

That makes **point-in-time feature engineering** a modeling constraint, not a post-hoc audit.

---

## Production ML / data engineering

The prediction engine sits on top of a larger data platform, so model correctness depends on engineering controls:

- append-only / write-once forecast and factual history;
- deterministic pagination;
- idempotent scheduled pipelines;
- freshness and reconciliation checks;
- production champion vs shadow challenger authority;
- CI regression tests derived from actual production failures;
- fail-closed publication when inputs or code identity are not trustworthy.

See [Data engineering](../../docs/DATA_ENGINEERING.md) and [Architecture](../../docs/ARCHITECTURE.md).

---

## Product evidence

### Overview — the product-first view

![NYX Aether Overview](../../assets/screenshots/overview.png)

*The portfolio leads with the shipped product: live/upcoming games, model context, and decision-oriented baseball intelligence rather than a benchmark scorecard.*

| Game Analysis | Game Center |
|---|---|
| ![Game Analysis](../../assets/screenshots/analysis.png) | ![Game Center](../../assets/screenshots/game-center.png) |
| Explainable matchup and model context | Game-level live / pregame information architecture |

### Evaluation evidence

The public **[Model History screenshot](../../assets/screenshots/model-history.png)** is kept as supporting evaluation evidence, not as the product hero. The benchmark numbers and their uncertainty are documented above and in [Model Evaluation](../../docs/MODEL_EVALUATION.md).

---

## Interview-ready talking points

1. Why a random split is the wrong default for a forecasting problem.
2. Why log loss / Brier are more informative than accuracy for probabilistic predictions.
3. How Platt vs isotonic calibration were selected without leaking the test set.
4. Why Engine 2.2 vs 3.0 is paired model evaluation rather than randomized A/B testing.
5. How bootstrap confidence intervals changed the model-promotion conclusion.
6. How a pagination bug invalidated the first benchmark and how reproducibility was restored.
7. Why point-in-time feature availability matters as much as predictive power.
8. How shadow evaluation, model authority, monitoring, and rollback reduce production risk.

---

## Skills summary

**Machine Learning:** Binary Classification · Logistic Regression · L2 Regularization · Feature Engineering · Feature Standardization · Probability Calibration · Platt Scaling · Isotonic Regression

**Statistics / Experimentation:** Log Loss · Brier Score · AUC · ECE · Bootstrap Confidence Intervals · Paired Bootstrap · Cluster Bootstrap · Sign Test · Champion–Challenger Evaluation

**Validation:** Walk-Forward Validation · Out-of-Time Testing · Point-in-Time Features · Look-Ahead Bias Prevention · Prospective Validation · Holdout Governance

**ML Engineering:** Reproducible ML · Model Versioning · Shadow Evaluation · Production Monitoring · Data Quality Controls · CI/CD · Scheduled Pipelines

**Data / Product:** Python · SQL · pandas · scikit-learn · PostgreSQL / Supabase · Next.js · GitHub Actions · Cloudflare Workers · Vercel

---

## Deeper technical material

- [Architecture](../../docs/ARCHITECTURE.md)
- [Technical case study](../../docs/TECHNICAL_CASE_STUDY.md)
- [Data engineering](../../docs/DATA_ENGINEERING.md)
- [Model evaluation](../../docs/MODEL_EVALUATION.md)
- [Project evolution](../../docs/PROJECT_EVOLUTION.md)

The full production source remains private; this showcase documents the methodology, verified results, system design, and failure analysis without publishing the complete production implementation.
