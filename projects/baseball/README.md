# NYX Aether — Baseball Intelligence

**Predictive ML · Probabilistic Forecasting · Temporal Validation · Statistical Inference · Production ML**

[← Portfolio home](../../README.md) · [Live product](https://baseball.nyx-aether.com) · [Architecture](../../docs/ARCHITECTURE.md) · [Model evaluation](../../docs/MODEL_EVALUATION.md)

NYX Aether is a production MLB analytics and forecasting system. The modeling problem sounds simple — estimate which team is more likely to win — but the real DS problem is harder: **produce a probability from only information available before first pitch, preserve that forecast, and evaluate it later without rewriting history.**

---

## Recruiter summary

This project demonstrates an end-to-end predictive data-science lifecycle:

```text
MLB / Statcast / contextual data
              ↓
point-in-time feature engineering
              ↓
model training
              ↓
probability calibration
              ↓
time-separated validation
              ↓
champion–challenger evaluation
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
| | L2 Regularization | Penalizes unstable coefficients and reduces overfitting |
| | Deterministic optimization | Engine 3.0 reference implementation uses deterministic IRLS fitting |
| **Feature Engineering** | Rolling features | Recent win rate and run differential over historical windows |
| | Season-to-date features | Team strength built only from information available before the target game |
| | Rest / fatigue context | Rest-day signal included as a point-in-time feature |
| | Home-field control | Explicit home indicator rather than leaving home advantage implicit |
| | Standardization | Scaling statistics fit on train only, then frozen for later partitions |
| **Validation** | Walk-Forward Validation | Chronological train → validation → test; random split is intentionally prohibited |
| | Out-of-Time Testing | Future season held out from fitting and calibration decisions |
| | Point-in-Time Reconstruction | Historical scoring rebuilds what the model could have known at that moment |
| | Look-Ahead Bias Prevention | Future outcomes, postgame facts, and later-ingested records are fenced from earlier predictions |
| | Test-set governance | Final test access is guarded to reduce repeated tuning against the holdout |
| **Probability Modeling** | Probability Calibration | Raw model probability is evaluated separately from calibration |
| | Platt Scaling | Logistic recalibration on validation evidence |
| | Isotonic Regression | Monotonic non-parametric calibration with edge-stability guards |
| | Out-of-Fold Selection | Calibration candidates are compared on OOF validation predictions rather than their own fit rows |
| **Evaluation** | Log Loss | Primary proper scoring rule for probabilistic forecasts |
| | Brier Score | Measures squared probability error |
| | AUC | Measures ranking/discrimination quality |
| | Expected Calibration Error | Measures probability reliability |
| | Accuracy | Secondary directional win/loss metric |
| **Statistical Inference** | Bootstrap Confidence Intervals | Quantifies uncertainty around model metrics |
| | Paired Bootstrap | Resamples game-level Engine 3.0 − Engine 2.2 differences on the same games |
| | Cluster-Aware Bootstrap | Resamples date clusters to address shared game-day conditions / effective sample size |
| | Sign Test | Non-parametric paired directional comparison |
| **Experimentation** | Champion–Challenger Testing | Incumbent Engine 2.2 vs Engine 3.0 challenger under fixed comparison rules |
| | Paired Benchmarking | Both engines score the identical eligible game cohort |
| | Prospective Validation | Predictions are captured before outcomes, then graded after settlement |
| **MLOps** | Model / artifact versioning | Model and calibration identity are versioned and content-addressed |
| | Shadow evaluation | Challenger can accumulate evidence without silently becoming production authority |
| | Reproducibility | Same inputs / config / code identity are expected to reproduce the same output |
| | Monitoring & rollback | Production authority, health checks, and fallback paths are explicit |

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

Why this matters: a random split would allow later-season information patterns to leak into the model’s apparent historical performance. In a forecasting problem, **time is part of the experimental design**.

Related public documentation: [Model evaluation](../../docs/MODEL_EVALUATION.md) · [Technical case study](../../docs/TECHNICAL_CASE_STUDY.md)

---

## Engine 2.2 vs Engine 3.0 — paired model experiment

This is intentionally **not described as an A/B test**. There is no randomized assignment of games to treatments. Both engines score the same eligible games; the unit of comparison is a paired game.

Validated strict historical benchmark:

| Metric | Engine 2.2 | Engine 3.0 V1 | Δ (3.0 − 2.2) |
|---|---:|---:|---:|
| **Log loss** | 0.700994 | **0.685063** | **−0.015931** |
| **Brier score** | 0.253664 | **0.246091** | **−0.007573** |
| **AUC** | 0.510558 | **0.556821** | +0.046263 |
| **ECE** | **0.036793** | 0.045429 | +0.008636 |
| **Accuracy** | **0.541667** | 0.533333 | −0.008334 |

Cohort: **120 paired 2026 games across 9 dates**.

Paired game-level uncertainty:

- Log-loss Δ 95% CI: **[-0.044632, +0.013092]**
- Brier Δ 95% CI: **[-0.021630, +0.006606]**
- Sign test: **p = 0.201**

### Statistical conclusion

> Engine 3.0 is **directionally better** on log loss and Brier score in this cohort, but the improvement is **not statistically established** because the primary paired confidence intervals include zero.

That conclusion is deliberately less exciting than “Engine 3.0 wins.” It is also the scientifically defensible conclusion.

---

## A benchmark that invalidated itself

One of the strongest parts of the project is a failure, not a model score.

The initial engine comparison exposed nondeterministic predictions. Root cause: paginated database reads were not ordered, so page boundaries could return inconsistent row sets. A benchmark that cannot reproduce its own inputs cannot credibly measure a small model difference.

The response was to:

1. invalidate the original result;
2. add deterministic total ordering to paginated reads;
3. verify unique ordering keys;
4. rerun both engines from scratch;
5. reproduce the 120-game cohort bit-exact across independent full runs;
6. keep the statistical conclusion conservative.

This is the kind of **data-quality / experimental-integrity debugging** that is easy to miss in notebook projects and central to real DS work.

---

## Feature engineering principles

A feature is not eligible simply because it correlates with outcomes. It must also be reconstructable at prediction time.

Examples of active / verified feature families include:

- recent team form;
- rolling run differential;
- season-to-date strength;
- team rest days;
- home-field indicator.

Candidate features are admitted only when their historical availability can be reconstructed without future leakage. Missing or unverifiable inputs are excluded rather than backfilled with information that would not have existed at prediction time.

This makes **point-in-time feature engineering** a core modeling constraint rather than a post-hoc audit.

---

## Probability calibration

For a user-facing forecast, “picked the winner” is not enough. A 70% prediction should behave like a 70% event over repeated observations.

The calibration layer compares:

- **identity / none** — preserve the raw model probability;
- **Platt scaling** — fit a logistic transform of raw logits;
- **isotonic regression** — fit a monotonic non-parametric mapping.

Candidate calibrators are evaluated out-of-fold on validation data. Selection prioritizes log loss, then Brier score, then ECE, with simplicity as a tie-breaker. The test partition does not select the calibrator.

Skills demonstrated: **calibration analysis, proper scoring rules, OOF model selection, overfit prevention**.

---

## Data engineering & production ML

The prediction engine sits on top of a broader data platform, so DS correctness also depends on engineering controls:

- append-only / write-once historical records for forecasts and factual snapshots;
- deterministic pagination and total ordering;
- idempotent scheduled pipelines;
- source freshness and reconciliation;
- cloud scheduling and retry-safe execution;
- explicit production champion vs shadow challenger authority;
- CI regression tests created from real production failure modes;
- fail-closed publication when inputs or code identity are not trustworthy.

See [Data engineering](../../docs/DATA_ENGINEERING.md) and [Architecture](../../docs/ARCHITECTURE.md).

---

## Product evidence

| Model History | Game Analysis |
|---|---|
| ![Model History](../../assets/screenshots/model-history.png) | ![Analysis](../../assets/screenshots/analysis.png) |
| Public grading and market context | Explainable matchup and model context |

| Overview | Game Center |
|---|---|
| ![Overview](../../assets/screenshots/overview.png) | ![Game Center](../../assets/screenshots/game-center.png) |

---

## Interview-ready skill summary

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
