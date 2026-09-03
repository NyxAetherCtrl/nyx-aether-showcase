# Model Evaluation

How the models are measured, what the numbers actually are, and the machinery that polices the numbers. Every decision-grade result here is scoped with its cohort, sample size, and uncertainty; development diagnostics are labeled as such, with their cohort and N.

## The domain, stated plainly

MLB game outcomes are close to coin flips: strong teams lose to weak teams constantly, and that's the sport, not a modeling failure. Across this project's internal walk-forward experiments with the current feature family, useful single-game performance clusters near **AUC 0.58–0.60** — treat that as an *internal empirical* ceiling observed on this data, not a cited external bound. Any presentation of this domain that leads with a big accuracy number is either measuring something else or hiding a denominator. The evaluation doctrine follows from that reality:

- **Calibration, log loss, and Brier score are the primary metrics.** For a probability product, "when we say 60%, does it happen 60% of the time" matters more than ranking.
- **AUC and selective accuracy are secondary** — they can identify a skill signal, but never automatically justify replacing a full-game model.
- **Every decision-grade claim carries its cohort, N, and uncertainty.** A number without a denominator is marketing.

## The two engines

**Engine v2.2 — production.** The transparent pillar engine described in [ARCHITECTURE.md](ARCHITECTURE.md): hand-weighted category scores at its core (the explainer layer, never replaced), with a calibrated probability layer that blends in a regularized linear model learned from the same sub-scores. What matters for evaluation: it was promoted five days into the project behind a config flag with a tested rollback, and every forecast decomposes into pillar contributions a reader can interrogate.

**Engine 3.0 — experimental.** A machine-learned challenger (regularized logistic regression over a frozen feature contract) developed under walk-forward validation with sealed holdout seasons. Experimental builds run in shadow mode on the live slate — same games, same information cutoff — publishing only to candidate-labeled surfaces, with no production authority. Its own development metrics stay in the private research record; the only public numbers for it are the retrospective-benchmark toplines further down, not the arc in the next section (which is the production model line's).

## The production model line: development arc (diagnostic)

These are **development diagnostics**, not decision-grade headline metrics — a walk-forward record of how the production model line improved as features were added. All four rows share one cohort: cross-season, expanding-window walk-forward with no leakage (train precedes test; features reconstructed point-in-time, strictly before first pitch), evaluated **out-of-sample across two recent held-out seasons, N = 3,145 games**.

| Iteration | OOS AUC |
|---|---|
| Team-form baseline (rolling win/run differentials only) | 0.520 |
| + pillar sub-scores and Statcast signals | 0.552 |
| + learned weights (regularized linear model over the sub-scores) | 0.5626 |
| + confirmed-lineup completeness (same-day availability) | 0.5673 |

The ladder is shown in AUC on purpose: it is the one comparable single number across iterations, which makes it the right *diagnostic* for "did this feature add signal." It is not the promotion criterion — that role belongs to calibration, log loss, and Brier (the primary metrics above), which is why no model was promoted on an AUC gain alone. On the same walk-forward out-of-sample cohort (the two held-out seasons, N = 3,145), calibration held to a mean decile gap of ≈ 0.014 between predicted and observed win rates. The gains are deliberately incremental — each step had to earn its place out-of-sample against a domain where the internal empirical ceiling sits near 0.58–0.60.

(These figures belong to the production-side model line. The experimental Engine 3.0's development numbers are deliberately not published; its only public numbers are the retrospective-benchmark toplines below.)

## The governance stack

Five mechanisms, ordered by when they bind:

1. **Frozen protocols before training.** Split policy, metric definitions, and season roles were fixed as hashed contracts before any model fitting. Holdout seasons are refused by the data loaders themselves — asking for a sealed season raises, regardless of who asks. Changing the rules after seeing results is not a temptation to resist; it's an operation the code refuses.
2. **Preregistered one-shot validation.** The frozen experimental candidate was validated on a held-out out-of-time season exactly once, under a preregistered protocol, and passed its preregistered acceptance criteria — calibration- and log-loss-first metrics fixed before execution. The run is recorded as **consumed** in a machine-readable module: no retry, rerun, recalibration, or replacement headline, permanently. The detailed metrics stay in the private research record; the public claim is exactly this: run once, passed.
3. **A challenger with no keys.** The experimental engine's writer routes every table through an allowlist that raises an exception before any network call if the target is a production table. It cannot promote itself, overwrite official forecasts, or touch product config. Promotion requires a human decision recorded in config; it cannot happen as a side effect of anything.
4. **A prospective ledger with a minimum-evidence rule.** The live engine-vs-engine comparison accrues paired games append-only and refuses to emit any directional statement below **200 paired games**. Its report renderer asserts output against a banned-vocabulary check — below threshold, a report containing "beats" or "superior" fails the run.
5. **A user-facing seal.** A repository test sweeps the production UI tree and fails CI if experimental output renders anywhere without candidate labeling. "No unlabeled candidate numbers in front of users" is a test, not a norm.

## The retrospective benchmark — including the part where it broke

A one-time retrospective comparison graded both engines on an identical strictly-filtered cohort: **N = 120 games** across 9 dates of the 2026 season, same information cutoff, paired by game.

What makes it worth writing about:

- The harness **caught its own bug**: results were not reproducible across runs, traced to unordered paginated database reads. The first result was declared invalid.
- After fixing read determinism, the re-run reproduced **120/120 bit-exact** across independent executions.
- The published verdict, with the incentive running the other way: the experimental engine is **directionally better and not statistically established** — sign test 67 of 120, p ≈ 0.2, consistent with chance. At N=120, a real edge and luck are indistinguishable, and the document says so.

The conclusion drawn was procedural, not promotional: keep accruing prospective evidence through the ledger; no promotion case exists yet.

## The public record

The product grades itself in the open. The Model History page shows the production engine's record on every graded forecast — snapshot as of **September 2, 2026: 524–450 on 974 graded picks — 53.8%, ± 3.1 percentage points at 95% confidence**. That is above a coin flip (two-sided p ≈ 0.017), which is the weakest relevant baseline; the page shows wins and losses equally. (This figure moves daily as games grade; the snapshot date is stated so the number is dated rather than pretending to be live.)

It also shows sportsbook favorites' records over a recent overlapping window: roughly **56.8–57.3%**, each book graded *on its own priced cohort* (213–227 games each). The page explains, in product copy, why these percentages must not be compared directly: different denominators, different game sets, different abstention behavior. The product's stated doctrine — "compare the records, not just the percentages" — is a deliberate refusal to let a favorable-but-wrong comparison stand, in either direction, and a refusal to treat book hit rates as an "accuracy" benchmark the model should beat.

This is the same rule that governs internal evaluation, applied where users can see it.

## What is deliberately absent

- **No headline accuracy number without a cohort.** Every percentage in this document has a denominator attached.
- **No claim that the experimental engine is better.** The evidence to date: directionally promising, not statistically established, minimum-evidence threshold not yet met. That is the strongest claim the data licenses, so it is the claim.
- **No reader-checkable calibration plots — yet.** Calibration evidence currently lives in internal tooling (the decile-gap figure above is its summary); publishing reliability curves with per-bin counts is the natural next addition to this document.
- **No betting framing.** The product presents explainable sports intelligence with confidence and counter-evidence, and a CI denylist keeps betting-advice language out of all four locales.
- **No model internals.** Feature definitions, weights, coefficients, and calibration parameters stay private; this document describes the evaluation machinery, which is the transferable part anyway.

## Why this matters beyond baseball

The evaluation problem here — tiny real edges, abundant noise, one person with every incentive to overclaim — is the same problem as A/B testing a product change or validating a growth model. The solutions are the same too: preregistration, frozen metrics, minimum sample thresholds, paired comparisons with uncertainty, and separating the system that generates claims from the system that grades them. Baseball is the dataset; the discipline is the skill.
