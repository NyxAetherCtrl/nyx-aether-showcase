# Project Evolution

Eight stages from first commit to operated production system, reconstructed from the git record: 1,000 commits and 438 merged pull requests between June 10 and September 2, 2026. The through-line is that nothing "launched" — things were **promoted**, behind flags, with rollback paths, after evidence.

## Stage 1 — Skeleton with a spine (June 10)

The first commit already contained the three-tier shape: Next.js app, Supabase schema, Python pipeline. Two decisions from week one outlived everything built on top of them:

- **Migration 0001 included a cross-source player identity map** (MLBAM / Retrosheet / Baseball-Reference / FanGraphs IDs). Entity resolution before the second data source existed.
- **Pull requests became the rule within the first week** — 438 merged since, and the habit is the audit trail.

## Stage 2 — An engine in production in five days (June 15)

Engine v2.2 — the transparent pillar engine with calibrated probabilities — was promoted to official five days after the first commit, via a migration that flipped a config flag with a tested rollback. The bet: a simple, explainable engine operating in production teaches more than a sophisticated one in a notebook. It has been the official engine ever since; the machine-learned challenger built to replace it has not yet cleared the evidence bar.

## Stage 3 — The data platform hardens (June–July)

The middle weeks built out ingestion across all six sources — and the schema tightened as failures taught lessons:

- Upserts were caught rewriting "pregame" probabilities after first pitch — 33 of 60 archived values had drifted → the **write-once forecast archive**: insert-only writes behind a unique lock, verified by standing checksum audits.
- "What did we know then?" kept mattering → **three-clock observation records** and append-only history for starters, market odds, and weather, with immutability enforced by database refusal triggers — nine tables by the end of July, twelve by mid-August.

## Stage 4 — The clock question (June → August)

A Cloudflare Worker drove the intraday dispatches from week one; GitHub's native cron was never trusted with time-sensitive lanes. In late August, before handing the official-writer lane to scheduled cloud infrastructure, I measured bare GitHub cron on a production workflow: more than half the scheduled fires never happened ([case study #2](TECHNICAL_CASE_STUDY.md#2-scheduling-on-infrastructure-that-misses-half-its-alarms) has the numbers). The measurement justified finishing the migration — the Worker became the platform's single timing authority, with slot-identity dispatch and exactly-once dedupe, and jobs became delay-invariant by deriving work from the schedule slot rather than the wall clock.

## Stage 5 — The V3 product (July 20)

A full UI generation — Overview, Game Center, Live, Analysis, Players, Playoffs — promoted to the clean production URLs via rewrites, with the previous generation kept in-tree as the rollback path and temporary (not permanent) redirects so a rollback would never fight browser caches. Internationalization followed: the product ships in English, Japanese, Spanish, and Korean.

## Stage 6 — The experimental engine program (July–August)

A machine-learned challenger to the production engine, built under governance designed before results existed: contracts and holdout seasons were frozen first, development proceeded walk-forward, and the one-shot out-of-time validation was preregistered, run once, and recorded as consumed (the full stack is in [MODEL_EVALUATION.md](MODEL_EVALUATION.md)). August added the retrospective benchmark — which invalidated its own first result over a harness bug before publishing — and the prospective comparison ledger with its minimum-evidence rule.

## Stage 7 — The hardening era (August)

The month of fences, each one installed after a specific failure:

- The REST layer's **silent 1,000-row truncation** found by reconciliation → paged reader + CI simulation suites.
- The **encoding corruption** incident (seven files, all tests green) → byte-level guard whose CI job proves the guard can fail before trusting its PASS.
- The user-facing **candidate-labeling seal** → a repository test sweeping the production UI tree.
- The **product-language denylist** enforcing betting-advice-free copy across all four locales.

## Stage 8 — Full cloud operation (September)

The official forecast writer — the one component still running on a local machine — cut over to cloud execution: a SHA-pinned workflow with a code-identity preflight that refuses to publish on checkout divergence, and fail-loud liveness and coverage verification after every publish chain. The local path was demoted to a documented emergency fallback. Alongside it, the public **Model History** page began grading the engine in the open (521–438 on 959 picks at time of writing, under the "compare the records, not just the percentages" doctrine), and new content systems entered development under the same contract-first discipline.

## What the arc shows

- **Promotion over launch.** Every major transition — engine, UI, writer — shipped behind a flag or rewrite with a tested rollback.
- **Measurement before trust.** The official writer moved onto scheduled infrastructure only after the scheduler's failure rate was a number, not a suspicion.
- **Failures become gates.** Every incident in this history ends as a CI check. The system's maturity is the accumulation of things it can no longer do silently.
- **Governance scales down.** Preregistration and frozen protocols are usually institutional machinery; they turned out to be exactly what a solo project needs, because the reviewer they replace is the one person who can't be objective.
