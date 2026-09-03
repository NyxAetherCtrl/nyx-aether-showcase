# Data Engineering

The pipeline layer is the largest part of the system, ingesting six external sources into a PostgreSQL data platform on a schedule that runs all day, every day, unattended. This document covers the patterns that keep it correct — most of them purchased with a specific failure.

## Sources and ingestion

| Source | Data | Client discipline |
|---|---|---|
| MLB Stats API | Schedule, box scores, line scores, rosters, standings, transactions | Shared session, timeouts, 4-attempt exponential backoff, payloads normalized to flat rows at the boundary |
| Statcast (pybaseball) | Pitch-level and batted-ball events | Bulk historical pulls, validated before landing |
| The Odds API | Sportsbook moneylines | Explicit request-quota budgeting (free tier makes cost a design input) |
| Kalshi | Prediction-market prices | Snapshot cadence aligned to game schedule |
| Open-Meteo | Stadium weather forecasts | Archived as forecast-as-known-then, never overwritten by observations |
| NOAA GHCNh | Historical hourly weather | Backfill source for historical context |

Normalization happens at the client boundary — everything downstream of a client speaks flat row dictionaries with consistent naming, so no pipeline contains source-specific parsing.

## The idempotency doctrine

Every write path is safe to repeat. This is the single most load-bearing property in the system, because it makes every other reliability decision cheap:

- **Keyed upserts** where the current state is the product (standings, rosters).
- **Append-only inserts with content-addressed or natural keys** where history is the product (forecasts, market snapshots, starter observations). Re-running a slot re-derives the same keys and inserts nothing new.
- **Monotonic completion** for slowly-resolving facts — e.g., injury stints upsert an activation date only from NULL to a value, never backward, and rows are never deleted.

Because duplicate work is free, the scheduler is allowed to be paranoid: overlapping runs, retries, and defensive re-dispatch are all harmless. Correctness doesn't depend on exactly-once delivery — it depends on at-least-once plus idempotent writes, which is a property you can actually have.

## Healing, not hoping

Scheduled runs fail: sources go down, runners die, rate limits bite. The system's posture is that gaps are normal and healing is routine:

- **Lookback windows.** Daily pipelines re-scan a trailing window rather than only "today," so a missed run's work is picked up by the next run automatically.
- **Set-based reconciliation.** Instead of assuming per-item success, reconciliation compares the *set* of rows that should exist against the set that does, and fills the difference. This is also how the silent-truncation bug was caught ([case study #3](TECHNICAL_CASE_STUDY.md#3-the-database-was-silently-lying)).
- **Backfill as a first-class citizen.** Every analytics dataset declares a backfill entry point in the pipeline registry; historical reconstruction runs the same validated code path as the daily build.

## Freshness that can't lie

Every pipeline run executes inside an observability wrapper — a context manager that ties together run metadata, timing, and a freshness ledger. The invariant: **freshness is only advanced by a successful run.** A pipeline that crashes cannot mark its data fresh; a pipeline that succeeds cannot forget to. The product reads the freshness ledger to badge stale data honestly rather than presenting old numbers as current.

## The registry and the semantic layer

Two deliberately boring registries that keep paying off:

- **Pipeline registry.** A frozen manifest declares every dataset: builder, validator, backfill, source, owner, status (production / candidate / planned), and schedule. It is the single answer to "what exists, who owns it, is it production" — the data-catalog question, answered in code.
- **Formula registry.** Shared statistical formulas (rate stats, derived indexes) live in one module with denominator guards, null handling, and fixed rounding. Before adoption, its outputs were verified byte-identical to the inline implementations it replaced — a semantic layer introduced without a single silent value change. Each entry maps metric → formula → methodology, and separates public MLB metrics from the platform's own indexes.

## Validation as recomputation

Validators don't check shapes; they **recompute**. Each analytics dataset's validator independently re-derives the builder's outputs from source data and compares — internal consistency proven, not asserted. The build / validate / backfill triad (~57 modules across the analytics layer) means every number on the product has a second, independent derivation standing behind it.

Two examples of fail-loud validation in practice:

- **Injury stints.** Rebuilt leak-free from the official transactions feed by pairing IL placements with activations on effective dates, so "on the IL as of date D" is point-in-time safe. Overlapping stints, duplicates, or an activation predating its placement refuse the run. Writes are dry-run by default; mutation requires an explicit confirm flag.
- **Engine benchmark harness.** Caught its own non-determinism and invalidated its first result before publishing anything ([case study #5](TECHNICAL_CASE_STUDY.md#5-evaluation-governance-that-cannot-overclaim)).

## Identity resolution

A cross-source player identity map (MLBAM, Retrosheet, Baseball-Reference, FanGraphs IDs) has existed since the very first migration, with partial unique indexing on the primary source ID. Every multi-source join in the platform — Statcast to box scores, injuries to rosters — resolves through it. Getting entity resolution in place *before* the second data source arrives is one of the few things this project got right the first time.

## Byte integrity: the encoding guard

A shell round-trip once corrupted seven merged files — mojibake byte sequences replacing multibyte characters — while every test stayed green, because the corrupted bytes were in strings and comments. The response:

- A **stdlib-only byte-level guard** that scans tracked files for corruption signatures.
- A CI job that **first proves the guard can fail** — it feeds the guard a known-corrupted fixture and requires a FAIL — before trusting the guard's PASS on the repo. A green check from a broken checker is the same silent-failure class as everything else in this document.

The repo also standardizes line-ending policy explicitly, because a Windows + POSIX toolchain otherwise turns every diff into noise and every byte-level guarantee into a coin flip.

## Scheduling and time

Covered fully in [case study #2](TECHNICAL_CASE_STUDY.md#2-scheduling-on-infrastructure-that-misses-half-its-alarms), but the data-engineering summary:

- Primary timing authority is a Cloudflare Worker; GitHub Actions is compute. The decision followed measurement — bare GitHub cron dropped more than half its scheduled fires ([case study #2](TECHNICAL_CASE_STUDY.md#2-scheduling-on-infrastructure-that-misses-half-its-alarms)). Each workflow keeps its GitHub cron as an offset backup, so execution is **at-least-once**; idempotent writes make a doubled slot harmless.
- Jobs derive their slate from the **schedule slot**, not the wall clock — late runs do the right work.
- Eastern-time schedules are DST-doubled cron pairs whose inactive half is a verified no-op.
- Information cutoffs are fixed per slate (e.g., forecasts use only data available at a declared cutoff time), so multiple daily slots resolve to one canonical package and repeat slots verify idempotence instead of double-writing.

## Testing

- **3,137 test functions across 147 files** in the pipeline layer (pytest), including simulation suites for infrastructure behavior: a PostgREST-shaped fake that enforces the 1,000-row cap, DST-transition schedule tests, and append-only refusal tests.
- **96 test files in the web tier** using `node:assert` with zero test-framework dependencies — including snapshot-based regression tests for content ranking and the sweep test that keeps experimental output off user-facing surfaces.
- CI is 16 gates wired to path filters. The gates that encode past incidents (encoding guard, pagination suites, product-language denylist, candidate-labeling sweep) run as blocking checks, not advisories.

## The short version

Nothing in this layer is exotic. It is retries, idempotent writes, reconciliation, freshness ledgers, recomputing validators, and regression gates — each one installed in response to a specific, measured failure, and each one pinned so the failure class can't return. Data engineering here isn't a framework choice; it's the accumulation of fences around everything that once bit.
