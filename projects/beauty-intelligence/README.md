# Beauty Intelligence

**NLP / Information Extraction · Taxonomy Engineering · Consumer Analytics · Human-Ground-Truth Evaluation · Time-Series Intelligence · Data Products**

[← Portfolio home](../../README.md) · [Live product](https://beauty-intelligence-seven.vercel.app)

Beauty Intelligence is a production analytics product for turning noisy consumer reviews into structured, decision-ready intelligence for brands and product teams.

The central data-science problem is not “run sentiment analysis.” It is:

> **How do you define a stable semantic system that can identify what consumers are talking about, distinguish praise from complaints and speculation from asserted experience, preserve evidence, aggregate it correctly, and validate it against human judgment?**

---

## Recruiter summary

This project demonstrates a different DS skill set from the Baseball forecasting system:

```text
retailer product + review data
            ↓
normalization / identity resolution
            ↓
domain taxonomy
            ↓
offline semantic tagging
            ↓
concept + evidence + sentiment + assertion
            ↓
persisted mention layer
            ↓
time-series / denominator rollups
            ↓
brand + product intelligence
            ↓
human-ground-truth evaluation + error analysis
            ↓
production B2B analytics product
```

The engine is designed so that analytics pages consume **persisted semantic evidence** rather than rerunning text interpretation on every request.

---

## Data Science techniques used

| Area | Technique | How it is used |
|---|---|---|
| **NLP / Information Extraction** | Multi-label concept extraction | A review may express multiple benefits, complaints, usage experiences, or product attributes |
| | Evidence-span extraction | Semantic outputs retain the supporting text span instead of returning only a label |
| | Phrase / context matching | Multi-word and contextual patterns reduce naive keyword matching |
| | Negation handling | Distinguishes statements such as “not irritating” from “irritating” |
| | Assertion state | Separates asserted experience from expectation, speculation, or other non-equivalent language |
| | Sentiment classification | Concept mentions carry positive / negative semantic orientation where supported |
| **Semantic Modeling** | Taxonomy / ontology design | Beauty language is mapped into stable concept and dimension definitions |
| | Versioned concept schema | Engine/taxonomy identity is frozen so changes can be evaluated rather than silently redefining metrics |
| | Domain-specific semantics | Cosmetic performance, feel, wear, application, irritation, pores, hydration and other domain meanings are handled explicitly |
| **Evaluation Design** | Human Ground Truth | Blind human labels are the authority for semantic-accuracy measurement |
| | Blind Annotation | Annotators do not see engine predictions while labeling evidence |
| | Stratified Sampling | Representative cohort is sampled proportionally across platform × rating |
| | Challenge Sets | Hard linguistic cases are evaluated separately rather than mixed into the headline cohort |
| | Span Alignment | Predicted and human mentions are matched by concept and evidence-span overlap |
| | Intersection over Union | Mean IoU quantifies evidence-span quality |
| | Exact-Match Evaluation | Exact span agreement is reported separately from overlap-based agreement |
| | Macro-F1 | Sentiment / assertion quality is evaluated across classes, not only by majority-class accuracy |
| | Confusion Matrices | Per-class failure modes are visible for semantic error analysis |
| | Strict End-to-End Metric | Requires concept + span + sentiment + assertion to all be correct |
| **Cohort / Sampling** | Deterministic cohorts | Evaluation cohorts can be rebuilt identically from the same frozen sample |
| | Representative vs challenge separation | Hard cases do not inflate or contaminate the representative accuracy estimate |
| **Analytics Engineering** | Review-level denominators | Shares are defined against reviews, not raw mention counts, where the product claim is review-based |
| | Support thresholds | Low-sample metrics surface as limited / unavailable rather than false precision |
| | Distinct-review counting | Repeated mentions do not silently become repeated consumers/reviews |
| | Typed availability states | Query failure, genuine no-data, and insufficient support remain distinct states |
| **Time-Series Analytics** | Monthly concept rollups | Semantic mentions are aggregated into longitudinal brand/category/global series |
| | Emerging-signal analysis | Recent concept movement can be compared against historical context |
| | Dirty-window recomputation | Incremental updates rebuild only affected time windows instead of full history |
| **Ranking / Decision Systems** | Complaint ranking | Complaint concepts are ranked from persisted evidence with explicit support requirements |
| | Comparable-peer selection | Brand benchmark peers use deterministic eligibility and comparability rules |
| | Product-family aggregation | Related variants are reconciled so product intelligence does not depend on one arbitrary SKU |
| **Data Engineering** | Multi-source review ingestion | Retailer review sources are normalized into a common analytical layer |
| | Incremental tagging | New/changed reviews can be processed without retagging the entire corpus |
| | Idempotent writes / recovery | Retries and recovery paths are designed not to duplicate analytical state |
| | Data-quality tests | Denominator, drift, read-failure and semantic-contract bugs become regression tests |
| **Production Governance** | V2 / V3 side-by-side tables | New semantic outputs are isolated from the incumbent data path |
| | Feature gating | V3 reads can be enabled only when explicitly authorized |
| | Rollback boundaries | A new engine version can be disabled without mutating the prior engine’s tables |

---

## Taxonomy engineering

Review analytics becomes unreliable when the semantic categories themselves drift.

Beauty Intelligence therefore treats taxonomy as a versioned data-science artifact rather than a loose list of keywords.

A concept definition must answer questions such as:

- What consumer language should count as evidence?
- What should **not** count despite containing the same word?
- Is the statement positive, negative, neutral, speculative, or negated?
- Does the evidence refer to product performance, expectation, comparison, or some unrelated sense?
- Can the same rule be applied consistently across retailers and time periods?

The engine uses a structured concept layer so downstream analytics can compare brands/products using stable semantic definitions.

### Why this is a Data Science skill

This is **feature representation for unstructured data**. The quality of downstream complaint rates, product strengths, trend charts, and brand comparisons depends on whether the text-to-concept mapping has a defensible definition.

---

## Human-ground-truth evaluation

A major design rule is:

> **An AI/model judging another semantic engine is not human ground truth.**

The tracked evaluation harness freezes a real-review sample and builds a blind annotation workflow. Real review text and labels remain outside the public/source-controlled evaluation code; the annotation tooling, cohort logic, evaluator, taxonomy snapshot, and labeling guide are versioned.

### Evaluation pipeline

```text
frozen 440-review sample
          ↓
deterministic cohort construction
          ↓
representative cohort + challenge cohort
          ↓
blind annotation batches
(predictions hidden from annotator)
          ↓
human evidence / concept / sentiment / assertion labels
          ↓
span-aligned evaluator
          ↓
concept + span + sentiment + assertion metrics
          ↓
error slices + adjudication
```

### Two evaluation cohorts

**Representative cohort**  
A proportional stratified sample across platform × rating. This is the cohort eligible to support a general real-accuracy estimate once human annotation is complete.

**Challenge cohort**  
Targeted difficult language — for example negation, failed mitigation, expectation/speculation, matcher traps, acne/fragrance ambiguity, and star/text disagreement. Challenge metrics are reported by slice and are **not folded into the headline representative result**.

This separation avoids an easy but common evaluation mistake: deliberately enriching the test set with difficult cases and then presenting that score as population accuracy.

---

## Evaluation metrics

The evaluator does not reduce semantic quality to one accuracy number.

### Concept detection

- mention-level overlap match;
- mention-level exact match;
- review-level set agreement.

### Evidence quality

- exact span rate;
- mean **Intersection over Union (IoU)** between human and engine evidence spans.

### Sentiment & assertion

- accuracy;
- macro-F1;
- per-class metrics;
- confusion matrices.

### Strict end-to-end correctness

A prediction counts as fully correct only when **concept + evidence span + sentiment + assertion** align under the strict policy.

### Error analysis

Metrics are sliced by challenge tag / linguistic failure mode rather than relying only on an aggregate score.

---

## Important evaluation disclosure

The evaluation infrastructure is production-grade, but the portfolio does **not** claim human-validated semantic accuracy before the blind annotation program establishes it.

That distinction is intentional. A system can have excellent engineering, complete corpus coverage, and useful product outputs while its semantic precision still requires independent human measurement.

Skills demonstrated here include **evaluation design, annotation protocol design, metric design, error taxonomy, and scientific restraint** — not a fabricated accuracy number.

---

## Persisted semantic layer

The product does not parse hundreds of thousands of reviews from scratch on every request.

Instead:

```text
review
  ↓
offline tagger
  ↓
concept mention
  ├─ concept identity
  ├─ evidence span
  ├─ sentiment
  └─ assertion/context
  ↓
persisted semantic tables
  ↓
brand / product / category / time-series queries
```

This turns NLP output into an **analytics-ready semantic data layer**.

Benefits:

- deterministic downstream queries;
- faster serving;
- auditable evidence;
- reproducible historical metrics;
- version-to-version comparison;
- ability to repair semantic logic offline without changing every UI query.

---

## V2 → V3 engine evaluation

Review Intelligence V3 was designed as a side-by-side system rather than an in-place rewrite.

The V3 migration creates separate semantic tables while leaving V2 untouched. Readers are feature-gated, and the engine can be evaluated before production reads are redirected.

This is best described as:

**Versioned Engine Evaluation / Shadow Rollout / Side-by-Side Validation**

—not as an A/B test, because users are not randomly assigned to competing semantic treatments.

This architecture demonstrates:

- challenger isolation;
- backward compatibility;
- rollback design;
- schema versioning;
- evaluation before promotion.

---

## Denominator design — an underrated DS problem

Consumer analytics can look mathematically precise while answering the wrong question.

Example:

```text
complaint mentions / all mentions       ≠       reviews with complaint / eligible reviews
```

A verbose reviewer can produce many mentions. If a product metric is supposed to represent review-level prevalence, raw mention counts are the wrong denominator.

The system therefore has explicit tests ensuring review-based denominators do not silently derive from mention volume. Similar rules distinguish:

- all collected reviews;
- rated reviews;
- reviews with semantic support;
- distinct platform + review identities;
- low-sample vs unavailable states.

This is **metric semantics / analytics engineering**, and it is central to making the B2B product trustworthy.

---

## Complaint Intelligence

The Complaint Evidence Engine ranks complaint concepts from **pre-tagged persisted data**. It does not rerun NLP during every dashboard request.

That design separates:

1. semantic interpretation;
2. analytical aggregation;
3. product presentation.

A complaint can therefore be traced back to supporting review evidence while the ranking layer remains deterministic and fast.

Skills demonstrated: **ranking logic, evidence-based analytics, semantic aggregation, explainable data products**.

---

## Peer benchmarking

Brand comparisons use deterministic peer-eligibility rules rather than simply displaying any famous competitor.

Comparable peers consider data availability and minimum support before ranking. Missing metrics carry typed reasons so a failed read is not displayed as a real zero.

This is a useful example of **business-facing model/metric governance**: selecting a benchmark cohort is itself an analytical decision.

---

## Data collection & quality engineering

The project also includes production review-collection pipelines. Real failures shaped the architecture:

- failed retailer fetches are not marked as successful checkpoints;
- retries use bounded backoff and explicit failure classes;
- partial data can be retained while the failed identity remains retryable;
- zero reviews must be explicitly verified rather than inferred from a network failure;
- recovery runs are isolated and idempotent;
- product-family identity is reconciled so variants can share review intelligence correctly.

This is relevant to Data Science because the semantic engine cannot be more trustworthy than the corpus feeding it.

---

## Product surfaces

Beauty Intelligence translates the semantic/data layer into business-facing workflows such as:

- Brand Intelligence;
- Product Intelligence;
- Complaint Radar;
- Consumer Love / benefit signals;
- review evidence exploration;
- top concepts by brand/product;
- peer benchmarking;
- emerging trend / concept context;
- product and category comparisons.

The product distinguishes **real empty**, **insufficient support**, and **read failure** rather than collapsing all three into `0` or `—`.

---

## Interview-ready skill summary

**NLP / Semantic DS:** Information Extraction · Multi-Label Concept Extraction · Taxonomy / Ontology Design · Negation Handling · Context Disambiguation · Sentiment Analysis · Assertion Classification · Evidence Extraction

**Evaluation:** Human-in-the-Loop Evaluation · Blind Annotation · Stratified Sampling · Challenge Sets · Span Matching · Intersection over Union · Macro-F1 · Confusion Matrices · Error Analysis · Adjudication Workflows

**Analytics:** Time-Series Aggregation · Trend Detection · Complaint Ranking · Cohort Design · Peer Benchmarking · Metric Semantics · Denominator Design · Support Thresholds

**Data Engineering:** Python · SQL · PostgreSQL / Supabase · Incremental ETL · Idempotency · Data Quality Testing · Multi-Source Ingestion · Recovery Pipelines · Data Lineage

**Production:** Engine Versioning · Feature Gating · Side-by-Side Validation · Rollback Design · Next.js · TypeScript · Vercel · CI/CD

---

## What I would discuss in an interview

1. **Why “sentiment analysis” alone was not sufficient** for product/manufacturing intelligence.
2. **How taxonomy design becomes a modeling problem** when real beauty language is ambiguous.
3. **Why human ground truth must be blind** and why an LLM-as-judge is not treated as final semantic authority.
4. **How representative and challenge cohorts answer different questions.**
5. **Why span IoU matters** when a system claims evidence, not only labels.
6. **How incorrect denominators can create more damage than an imperfect classifier.**
7. **Why V3 was built side-by-side with V2** instead of replacing production tables in place.
8. **How production read failures are kept distinct from true analytical zeroes.**

The full production source remains private. This public case study documents the DS methodology and product reasoning without publishing the complete semantic engine, proprietary review corpus, internal operational files, or production credentials.
