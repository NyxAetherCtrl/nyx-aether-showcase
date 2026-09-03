# Technical Case Study

Five problems from operating NYX Aether in production. Each one follows the same arc: something failed silently or threatened to, the failure was measured before it was fixed, and the fix was pinned with a regression gate so the failure class — not just the instance — was retired.

None of the code in this document is copied from the production repository. Where a snippet helps, it is labeled as simplified pseudocode.

---

## 1. The forecast of record

### Problem

A forecasting product is only as credible as its claim that "this is the probability we published *before* the game." The pipeline that wrote pregame win probabilities used upserts — convenient for reruns, and quietly catastrophic: a pipeline pass after first pitch could overwrite a "pregame" probability with one computed from in-game information.

### Why it mattered

Every evaluation metric — log loss, Brier score, the public win–loss record — is computed against archived pregame forecasts. If those rows can be rewritten, every metric is unfalsifiable. Nobody would ever notice, which is exactly the problem: the system would drift into looking better than it is, with no bug report ever filed.

### Failure mode measured

An audit compared archived probabilities against the values first captured for the same games: **33 of 60 differed**. The upsert was doing its job; the job was wrong.

### Design decision

Make the forecast archive **write-once, and make any violation detectable**:

- A dedicated archive table with a unique lock per game and model. The writer only inserts; no update path targets the archive.
- A standing checksum audit re-verifies the archive against recorded digests, so drift is caught mechanically rather than by luck.
- The product reads the forecast of record from the archive, never from mutable working tables — and evaluation reads the same archive, so the number a user saw and the number the model is graded on are provably the same row.

The incident also set a doctrine. For the archival tables built afterward — market snapshots, starter observations, weather archives, engine artifacts — immutability moved into the database itself: `BEFORE UPDATE OR DELETE` refusal triggers that bind every role, including the pipelines' own privileged service role. Twelve tables now carry that enforcement.

Simplified conceptual shape of the refusal-trigger pattern (not production code):

```sql
-- Simplified conceptual example
CREATE TRIGGER history_is_immutable
BEFORE UPDATE OR DELETE ON archival_table
FOR EACH ROW EXECUTE FUNCTION refuse_mutation();
```

### Validation

The checksum audit runs as a standing check, and the trigger-enforced tables refuse mutation in tests — attempted updates fail loudly, even with service-role credentials.

### Result & tradeoffs

Corrections can no longer be silent: a wrong forecast stays wrong in the archive, and any amendment is a *new* row with its own timestamp. That is the tradeoff, and it is the point. Storage grows monotonically; that cost is trivial next to the credibility it buys.

**Learning:** integrity guarantees belong in the lowest layer that can enforce them — and where the database can't hold the guarantee, an independent audit has to stand behind the convention.

---

## 2. Scheduling on infrastructure that misses half its alarms

### Problem

The platform is schedule-driven: pull odds before games, lock closing lines at T-5 minutes, refresh live scores every few minutes. All compute runs on GitHub Actions, whose native cron is documented as best-effort. A Cloudflare Worker had dispatched the intraday jobs since week one — the native scheduler was never trusted with time-sensitive lanes, on instinct and anecdote. But before the official forecast writer — the most correctness-critical lane — could move onto scheduled cloud infrastructure, "best effort" needed a number.

### Why it mattered

A late odds pull isn't degraded service, it's wrong data: a "closing line" captured 90 minutes after the game started isn't a closing line. Timing failures here corrupt the dataset, not just the user experience.

### Failure mode measured

Instrumented bare GitHub cron on a production workflow over a three-day window: **8 of 15 scheduled fires never happened at all**; among runs that did fire, the median delay was **121 minutes** and the worst was **479 minutes**. That is not jitter to tolerate; that is a scheduler to replace.

### Design decision

The measurement settled the architecture: *timing authority* and *compute* stay split, for every scheduled lane. The Worker — infrastructure whose one job is running on the minute — became the platform's single clock; GitHub Actions remained the runtime.

- The Worker fires on schedule and dispatches the target workflows via API, so jobs start within seconds of their slot instead of whenever the queue drains.
- Every dispatch carries a **slot identity** (which scheduled slot this run represents), and dedupe ensures each slot executes exactly once even if dispatch retries.
- Downstream jobs derive the slate they should process **from the slot, not from the wall clock**. A run that starts late still computes the right answer for the right slot.
- Overlapping or duplicate runs are harmless, because every write path is a keyed, idempotent upsert or an append with a content-addressed key.

One wrinkle worth naming: cron on a UTC clock doubles against US daylight saving time. Schedules that must track Eastern Time run as paired entries, where the half that doesn't match the current DST offset resolves to a deliberate no-op — a correct no-op, verified as such, rather than a missing run.

### Validation

Slot-identity labels make coverage auditable: for any day, the set of slots that should have run can be diffed against the set that did. Zero-write runs of the redundant DST half are verified as intentional no-ops, distinguishing "correctly idle" from "silently broken" — the distinction the original scheduler couldn't make.

### Result & tradeoffs

Deterministic timing at the cost of one more moving part and a cross-system dependency. Worth it: the Worker's failure mode (dispatch didn't happen) is observable, while the old failure mode (dispatch happens hours late with a fresh-looking timestamp) was invisible.

**Learning:** measure the platform you're standing on. The scheduler's unreliability was a rumor until it was a number; the number is what justified the architecture.

---

## 3. The database was silently lying

### Problem

Supabase exposes Postgres through a REST layer that caps responses at ~1,000 rows *without raising an error*. A query for 14,825 rows returns 1,000 and a success status.

### Why it mattered

Four production readers were consuming truncated results. The nastiest case: rows came back in an order where truncation dropped the *newest* data — so the pipeline computing "recent form" was reliably blind to the most recent games. Everything downstream looked plausible. Nothing failed.

### Failure mode observed

Found during a reconciliation pass when a downstream count didn't match the source table. The gap traced back to the reader, not the data.

### Design decision

Two layers, because the fix without the fence is just the setup for the sequel:

1. **A paged reader as the only read path** — walks pages under a deterministic total order (a unique key tiebreaker, so pagination is stable even when the sort column has duplicates) until exhaustion, and is the standard client for every consumer.
2. **A CI regression suite that simulates the truncation** — tests run readers against a fake that enforces the 1,000-row cap, proving each consumer survives it. New code that naively reads a capped endpoint fails CI before it ships.

### Validation

The reconciliation that found the bug now runs as a standing check; the regression suite is wired into CI as a gate, not an optional test.

### Result & tradeoffs

Paged reads cost round trips. The alternative — fast reads that are sometimes silently 7% of the table — is not an alternative.

**Learning:** the most dangerous API behavior is the one that succeeds. Anything that can partially succeed must be wrapped in a layer that makes partial success impossible or loud.

---

## 4. Point-in-time correctness as a schema property

### Problem

Forecasting data has a shape most app schemas can't represent: the truth changes over time, and *the history of what you believed* matters as much as the current value. A probable starting pitcher announced Tuesday, scratched Thursday, replaced Friday — a model trained on "what we knew at game time" must be able to replay exactly that, years later.

### Why it mattered

This is the leakage problem, and leakage is the death of forecasting credibility. A model that trains on quietly-updated rows is training on the future. The failure is undetectable at training time and shows up only as performance that evaporates in production.

### Design decision

Encode time in the schema, and make the database enforce the invariants:

- **Three clocks per observation.** Every observed fact carries the time the *source* claimed, the time the platform *observed* it, and the time the row was *written*. Disagreement between clocks is information (ingest lag, source corrections), not noise to collapse.
- **Observations append; they never update.** "The probable starter changed" is a new observation row. As-of-time queries reconstruct belief at any instant — the current state is just the latest observation, not the only one.
- **Content-addressed identity.** Key archival rows use a hash of their canonical content as primary key, and a `CHECK` constraint recomputes the hash *in the database* — a row whose content doesn't match its identity is unrepresentable.
- **The closing line is a pure function.** The sportsbook line of record locks five minutes before first pitch, derived entirely from the append-only snapshot history — and the deadline arithmetic is itself a `CHECK` constraint, so a "locked" row that violates its own deadline cannot exist.
- **Forecast-as-known-then for weather.** Stadium weather archives what the forecast *said in advance*, with integrity digests derived by Postgres itself — because grading a pregame model against observed weather (rather than forecast weather) is a subtle form of leakage.

### Validation

Replays are exact: as-of queries against the observation history are covered by tests, and the integrity constraints mean a corrupted archive fails at write time, not at analysis time — the invariants hold even against a buggy pipeline with full write credentials.

### Result & tradeoffs

Queries get more complex (as-of reconstruction beats `SELECT current_value`), and storage is strictly larger. In exchange, an entire class of leakage becomes structurally impossible rather than procedurally avoided.

**Learning:** "we're careful about leakage" is a process claim; process claims decay. Schema claims don't.

---

## 5. Evaluation governance that cannot overclaim

### Problem

A solo project has no reviewer to catch motivated reasoning. When the person who builds the model also grades it and writes the launch narrative, the incentives point one way — and MLB forecasting, where real edges are tiny, is exactly where wishful evaluation flourishes.

### Why it mattered

The experimental engine exists to eventually challenge the production engine. If its evaluation is contaminated — by peeking, by retrying validations until one passes, by cherry-picking windows — the comparison is worthless, and so is every decision built on it.

### Design decision

Replace self-discipline with structure. The rules are code, and the code refuses:

- **Frozen protocols.** Split policy, metric definitions, and season roles were fixed as hashed contracts *before* training. Holdout seasons are refused by the data loaders themselves — asking for a sealed season raises, regardless of who asks.
- **Preregistered, execute-once validation.** The out-of-time test on the frozen model ran exactly once, and its consumption is recorded in a machine-readable module: no retry, no rerun, no recalibration, no second look, permanently.
- **A live ledger with a minimum-evidence rule** and a banned-vocabulary check on its own reports ([MODEL_EVALUATION.md](MODEL_EVALUATION.md) has the details).
- **No authority for the challenger.** The experimental engine's writer cannot reach production tables (an allowlist raises before any network call). Promotion cannot happen as a side effect of anything; it requires a human decision.
- **The user-facing seal.** A repository test sweeps the production UI tree and fails if experimental output renders anywhere without candidate labeling.

### The moment it paid for itself

The retrospective benchmark comparing the two engines found a non-determinism bug **in its own harness** — unordered paginated reads made results irreproducible. It declared its first result invalid, fixed the read order, and re-ran to 120/120 bit-exact reproduction across independent executions. The final published verdict, on N=120 strictly-filtered games: the experimental engine is **directionally better and not statistically established** (sign test: 67 of 120, p ≈ 0.2).

### Result & tradeoffs

The cost is real: an execute-once rule means a promising model can never be re-validated on the same data; the only way forward is waiting for new evidence to accrue. But the numbers that survive are load-bearing. "Here's my model" is a claim; "here's the machinery that would have caught me if I'd cheated" is evidence.

**Learning:** governance is an engineering artifact. If a rule matters, it should be enforced by something that doesn't sleep, doesn't rationalize, and doesn't want the model to win.

---

## Common thread

Every case reduces to the same discipline: **instrument → quantify → fix → pin**. The scheduler was measured before it was fully trusted with the critical lane; the truncation was counted before it was fenced; the leakage was made unrepresentable rather than discouraged; the evaluation was governed before it was believed. Systems fail silently by default — the engineering work is making silence impossible.
