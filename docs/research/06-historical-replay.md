# 06 — Historical replay + backtesting

Status: **Complete**  
Research date: 2026-08-16

## Question

How should we test and tune the Attention Engine against real historical glucose/treatment data before trusting it live, without confusing glucose-prediction accuracy with the actual product goal or pretending retrospective simulation can prove a change in diabetes outcomes?

## Executive conclusion

**Build a deterministic, event-driven replay harness around the same pure Swift Attention Engine that will run live. Score complete Attention Episodes, not individual glucose predictions.**

Historical replay can answer questions such as:

- Would this policy have interrupted the user 2 times/day or 12?
- How early would it have surfaced a meaningful worsening episode?
- Would it have repeatedly nagged after insulin was logged?
- How often would an episode have resolved quietly because glucose recovered?
- How often would a fixed 10-minute defer have been cancelled early, retained, or escalated before expiry?
- What does approximate IOB add beyond recent-treatment timing?
- Which context signals materially improve the attention/no-attention trade-off?

It **cannot** prove that a counterfactual alert would have improved TIR, because the user did not actually receive and respond to that alert in the historical timeline. That requires prospective use.

The existing `mpereiragu/xdripswift-predict` backtester is a useful reference for Nightscout ingestion, chronological walk-forward evaluation and alert-frequency metrics, but it should not become our Attention Engine backtester. It evaluates glucose forecasts at fixed intervals. Our product needs a **discrete event replay** of every relevant glucose, treatment, user-action and scheduled-deadline event.

The recommended shape is:

```text
historical data source
        ↓
normalised ReplayEvent stream
        ↓
chronological priority queue
        ↓
production AttentionEngine + fake clock
        ↓
Attention decisions / episode state / scheduled deadlines
        ↓
decision trace
        ↓
metrics + episode review + policy comparison
```

The core rule is:

> **There must be one Attention Engine, not a production implementation and a separate backtest reimplementation.**

Python/notebooks can be useful for data preparation, plots and aggregate analysis, but the rules under test should be the same Swift domain code used by the app.

---

## 1. What the existing prediction backtester gets right

The strongest existing reference is `mpereiragu/xdripswift-predict`, branch `claude/xdrip-glucose-ai-prediction-r9v9c3`, particularly `server/tools/backtest.py`.

It already demonstrates several useful patterns:

- pulls historical SGV readings from Nightscout;
- sorts them chronologically;
- uses only information available at each evaluation point;
- leaves warm-up history before scoring;
- compares predictions to later observed glucose;
- calculates alert hit/miss and false-alarm metrics in addition to MAE/RMSE/MARD;
- reports alert frequency rather than considering prediction error alone.

Those are useful foundations.

### Why we should not reuse it unchanged

Its central loop evaluates a **prediction model every 30 minutes** and asks whether a future glucose value was predicted accurately.

Our product asks a different question:

> Given everything known *right now*, should diabetes interrupt the user, stay quiet, defer, escalate or resolve this episode?

The relevant unit is therefore the **episode**, not the forecast sample.

A technically excellent 30-minute predictor could still make a terrible Attention Engine if it:

- interrupts too often;
- repeats the same information;
- ignores logged treatment;
- fails to resolve quietly when the situation improves;
- treats stale data as continued physiology;
- cannot model user acknowledgement/defer state.

Prediction error should be a later supporting metric, not the primary score for the Attention Engine.

---

## 2. Replay the real event model

Pass 05 established that the live system should be event-driven. Historical replay should mirror that model exactly.

### Observed events

At minimum:

```text
GlucoseReading
TreatmentLogged
App/user context event (when historical data exists)
Data-gap / freshness change derived from timestamps
```

Later:

```text
Ate
NoInsulinNeeded
WaitingForRecovery
Attention acknowledgement
Exercise/context change
Meal-photo result
Health/Watch context
```

### Engine-generated events

Replay also needs events that do not exist in the source dataset but are created by the policy itself:

```text
DeferDeadlineReached
ScheduledFallbackNotificationDue
EpisodeExpiry / policy deadline
```

This means the replay runner should behave like a **discrete-event simulator**, not a loop that wakes every N minutes.

Conceptually:

```text
priority queue = observed historical events

while queue not empty:
    event = next chronological event
    fakeClock.now = event.time
    update replay state
    decision = AttentionEngine.evaluate(context)
    apply decision

    if decision schedules future boundary:
        insert synthetic event into queue

    if decision cancels boundary:
        invalidate scheduled synthetic event

    write full decision trace
```

This reproduces the architecture we intend to ship and naturally models the smarter `Remind 10 min` behaviour from pass 05.

---

## 3. Use the exact production evaluator

The production engine should be designed so replay can instantiate it directly.

Illustrative shape:

```swift
protocol AttentionClock {
    var now: Date { get }
}

protocol AttentionEvaluating {
    func evaluate(_ context: AttentionContext) -> AttentionDecision
}

struct ReplayClock: AttentionClock {
    var now: Date
}
```

The replay runner supplies:

- deterministic clock;
- in-memory or fixture-backed glucose provider;
- treatment provider;
- episode store;
- notification scheduler simulation.

The live app supplies:

- system clock;
- xDrip accessors;
- persistent episode store;
- UserNotifications adapter.

### Why this matters

If Python implements “approximately the same rules,” drift is inevitable:

```text
backtest says policy X is good
        ↓
Swift implementation differs subtly
        ↓
live behaviour is not what was tested
```

The replay result is only trustworthy if the code being evaluated is the code that will make live decisions.

### Role for Python

Python is still useful downstream for:

- loading/export conversion;
- exploratory analysis;
- plotting episode timelines;
- parameter-result tables;
- Pareto-frontier visualisation;
- statistical summaries.

It should not become the second owner of Attention decision logic.

---

## 4. Historical data sources

We should normalise every source into the same replay schema so the replay engine does not know whether data came from Nightscout, V7 backup or a test fixture.

### Route A — Nightscout

If the user's history is already in Nightscout, this is the cleanest initial extraction route.

Nightscout's documented APIs expose:

- CGM entries through `/api/v1/entries`;
- treatments through `/api/v1/treatments`;
- profiles through `/api/v1/profile`;
- query/count/date filters for retrieving more than the small default result set.

Nightscout v3 also provides collection-oriented APIs and history semantics.

Advantages:

- easy date-range extraction;
- already used by the existing prediction backtester;
- glucose and treatments share a common external source;
- avoids touching an iPhone app sandbox.

Caution:

- treatment completeness still needs to be measured;
- sync timestamps are not necessarily the same as the time xDrip received an event;
- Nightscout history can represent physiology well without perfectly reconstructing historical iOS execution latency.

### Route B — V7 backup

V7's Data Management UI now exposes backup options for:

- app settings and alerts;
- BG readings;
- treatments;
- optional account data.

The backup manifest also tracks counts and earliest dates, and the restore UI handles BG readings and treatments independently.

This is a strong future route because a user can produce a bounded file without sharing Nightscout credentials.

**Important implementation choice:** do not couple the replay engine directly to the `.xdripbackup` representation. Build a small importer/normaliser so backup format can evolve without touching the evaluator.

### Route C — V7 Nightscout historical import / local Core Data

V7's native Nightscout import UI independently confirms that historical BG readings and treatments can be imported into xDrip's existing local data model.

Once data is local, a future development-only exporter can serialise just the replay fields we need.

This is useful if the replay harness is run from Xcode/tests rather than directly against an external server.

### For existing pre-V7/Zukka history

We should not assume V7 backup can retroactively export another app installation's sandbox.

When we reach the first real-data run, choose the least-invasive source that actually contains the user's history:

1. Nightscout, if already populated;
2. an existing export/backup supported by the installed app;
3. a one-off local data extraction route if necessary.

We do not need to resolve this before defining the replay architecture.

---

## 5. Normalised replay schema

The replay layer should remove source-specific quirks before evaluation.

Illustrative model:

```swift
struct ReplayGlucose {
    let id: String
    let timestamp: Date
    let receivedAt: Date?
    let valueMgDL: Double
    let source: ReplaySource
}

struct ReplayTreatment {
    let id: String
    let timestamp: Date
    let kind: TreatmentKind
    let insulinUnits: Double?
    let carbsGrams: Double?
    let source: ReplaySource
}

enum ReplayEvent {
    case glucose(ReplayGlucose)
    case treatment(ReplayTreatment)
    case userEvent(AttentionUserEvent)
    case scheduledBoundary(ReplayScheduledBoundary)
}
```

Internally, choose one canonical glucose unit (probably mg/dL, because xDrip commonly stores calculations that way) and convert only at UI/report boundaries.

### Event time versus receipt time

Where data permits, distinguish:

- **event timestamp** — when the reading/treatment applies physiologically;
- **received timestamp** — when the app actually learned about it.

Historical sources may only give us the former reliably.

That creates an explicit fidelity level:

```text
physiology replay   = what the historical glucose/treatment sequence was
execution replay    = what the app knew and when it knew it
```

Early backtesting will mostly be physiology replay. From day one of our own app, log enough trigger metadata to support higher-fidelity execution replay later.

---

## 6. Data-quality gate before scoring anything

A replay report should start with a data-quality section. Otherwise a candidate can appear “smart” because the source data is incomplete.

Check at least:

- start/end date;
- total readings;
- median/expected reading cadence;
- number and duration of CGM gaps;
- duplicate timestamps/IDs;
- unit normalisation;
- treatment count;
- insulin-treatment count;
- carb-treatment count;
- days with no treatment records;
- suspicious treatment duplicates;
- stale-data periods;
- source/device changes where detectable.

### Critical treatment rule

**Absence of a historical insulin treatment means “no insulin record available,” not “the user definitely took no insulin.”**

This is the same principle the live product must follow.

If treatment logging was sparse historically, we can still evaluate:

- glucose-only episode segmentation;
- alert timing;
- persistence/rate logic;
- reminder burden;
- stale-data handling.

But we should mark action-aware metrics such as “unnecessary alert despite treatment” as low-confidence rather than treating missing records as ground truth.

---

## 7. Episode-level scoring

The core report should be organised around Attention Episodes.

Example episode trace:

```text
12:20  7.8 ↗   candidate state begins
12:25  8.4 ↗
12:30  9.1 ↑   Attention alert
12:33            insulin treatment recorded
12:35  9.5 ↑   policy stays quiet: recent action
12:40  9.7 ↗   policy stays quiet
12:45  9.6 →   improving
12:55  8.9 ↘   episode resolves silently
```

The useful questions are about the whole sequence, not whether 12:30 was classified correctly in isolation.

### Core Attention metrics

#### Cognitive-load metrics

- **interruptions per day**;
- **episodes alerted per day**;
- **repeat interruptions per episode**;
- percentage of episodes with more than one interruption;
- median time between repeated alerts;
- alerts shortly after a logged action;
- episodes that resolve without any interruption.

#### Timeliness metrics

- time from candidate episode onset to first alert;
- time from first alert to later severity boundary;
- proportion of worsening episodes surfaced before a chosen reference boundary;
- time saved versus static-threshold baseline.

#### Context-quality metrics

- episodes suppressed/downgraded after logged insulin;
- episodes later re-escalated despite logged action because new evidence worsened;
- reminders cancelled because glucose recovered;
- defer deadlines reached without fresh data;
- stale-data transitions handled without extrapolation.

#### Episode-outcome descriptors

These are descriptive, not causal:

- peak/trough after alert;
- episode duration;
- time to return below/above configured reference range;
- area/time beyond configured high/low reference where useful;
- treatment occurrence before/after alert.

### Do not optimize one magic “accuracy” number

False-positive/false-negative terminology is useful only where we have a defensible label for “needed attention.” Much of this problem does not have objective retrospective ground truth.

The primary view should therefore be a **trade-off surface**, not one score.

---

## 8. Use the user's configured thresholds as reference boundaries, not hidden medical truth

For retrospective comparison we will need severity/reference boundaries.

Prefer:

- the user's existing xDrip high/urgent-high/low settings;
- explicit configurable research thresholds;
- persistence/duration definitions stated in the report.

Do not silently invent new personalised clinical thresholds and then label the policy “correct” because it matched them.

The Attention Engine may ultimately alert *before* a static high threshold; the threshold is useful as a later outcome/reference boundary, not necessarily as the trigger itself.

---

## 9. Baselines we should compare

A candidate policy only becomes meaningful when compared against simpler alternatives on exactly the same event stream.

### Baseline A — static threshold + fixed repeat

Approximate the current user experience:

```text
cross configured high threshold
→ alert
→ fixed snooze/re-alert behaviour
```

We do not need a perfect reimplementation of every historical xDrip detail for the first experiment. The purpose is to represent the broad static-alert pattern.

### Baseline B — naive contextual rule

Example conceptual rule:

```text
high/rising persistently
AND no recent insulin record
→ alert
```

This tests whether the richer engine genuinely adds value over the obvious first implementation.

### Candidate C — Attention Episode policy

Include:

- persistence;
- rate/acceleration where robust;
- recovery state;
- recent treatment/action;
- defer state;
- stale-data handling;
- context-aware re-alert.

### Candidate D — later IOB-aware policy

After pass 07:

- replace crude “recent insulin” windows with approximate active-insulin context;
- compare the incremental value.

### Ablations

For each major signal, test the candidate with that signal removed:

```text
full engine
minus treatment context
minus acceleration
minus recovery state
minus defer logic
later: minus IOB
```

This tells us whether added complexity actually changes useful behaviour.

---

## 10. Optimise a Pareto frontier, not maximum sensitivity

The product has two objectives that naturally conflict:

```text
catch important situations earlier
                ↑
                │
                │
                └────────────→ fewer interruptions
```

A policy that alerts on every rise may “catch” everything but fail the product.

A policy that never alerts creates zero cognitive burden but also fails the product.

For a parameter sweep, plot candidate configurations using dimensions such as:

- x-axis: interruptions/day;
- y-axis: early-warning coverage or median lead time;
- secondary markers: repeats/episode, post-treatment nags, stale-data failures.

Prefer a sensible **knee of the Pareto frontier** rather than tuning for one maximum metric.

This keeps the product goal visible during technical optimisation.

---

## 11. Chronological tuning — no random train/test split

Glucose data is a time series with repeated days, routines and temporal correlation.

Do not randomly shuffle individual readings into train/test sets.

For personal policy tuning, use chronological evaluation such as:

```text
oldest period     → development/tuning
next period       → validation
most recent block → untouched holdout
```

or rolling/walk-forward evaluation.

The existing prediction fork already uses walk-forward semantics in the sense that each forecast only sees history available up to its timestamp.

### Avoid threshold overfitting

With one person's limited history, do not tune dozens of independent numerical thresholds until historical metrics look perfect.

Prefer:

- small number of interpretable policy parameters;
- broad parameter sweeps;
- robust regions rather than one exact optimum;
- untouched recent holdout;
- later prospective confirmation.

If changing a threshold from 14 to 15 minutes dramatically changes the conclusion, the rule is probably too brittle.

---

## 12. Historical user actions are missing — handle that explicitly

Past data generally will not contain the new product vocabulary:

- `Ate`;
- `No insulin needed`;
- `Waiting for recovery`;
- `Remind 10 min`;
- Attention acknowledgements.

We must not fabricate these and then present the replay as observed history.

### Mode 1 — observed-data replay

Use only events that really exist:

```text
CGM
insulin treatment records
carb treatment records
exercise treatment records if present
```

This is the highest-confidence historical evaluation.

### Mode 2 — counterfactual scenario replay

Inject clearly labelled hypothetical interactions to understand behaviour.

Examples:

- “What if `Ate` were recorded whenever a historical carb treatment exists?”
- “What if the first logged insulin treatment after an alert opportunity also acknowledged the episode?”
- “What would a 10-minute defer do at this point?”

Outputs from this mode must be labelled **simulation**, not historical fact.

### Stronger option: manual episode review

A small human-labelled set may be more valuable than clever synthetic assumptions.

Create a review view/report containing, for perhaps a representative sample of episodes:

- glucose chart;
- logged treatments;
- candidate alert points;
- reason codes;
- alternate-policy comparison.

The user can classify each episode with simple labels such as:

```text
wanted alert sooner
about right
unnecessary
insulin was actually taken but not logged
waiting for recovery
no action needed
```

This gives us personalised ground truth about **attention preference**, which cannot be recovered from glucose alone.

It also reveals how incomplete historical treatment logging is.

---

## 13. Decision trace is a first-class artifact

Every replay invocation should emit a machine-readable trace.

Example:

```text
DecisionTraceRecord
- timestamp
- trigger
- episodeID
- glucose IDs used
- treatment IDs used
- context snapshot / derived features
- decision
- reason codes
- previous episode state
- next episode state
- scheduled notification/deadline
- cancelled notification/deadline
- policy version
- parameter set ID
```

This provides:

- debugging;
- policy diffs;
- explainability;
- regression fixtures;
- manual episode review;
- later prospective/live comparison.

### Reason codes matter

A useful backtest should be able to explain:

```text
ALERT because persistentRise + noRecentRecordedAction
QUIET because recentInsulin + notWorseningEnough
RESOLVE because recoveringPersistently
ESCALATE because continuedRiseDespiteAction
```

This is much more actionable than a black-box scalar risk score.

---

## 14. Regression fixtures from real episodes

Once we identify representative historical situations, turn anonymised/minimised versions into Swift test fixtures.

Examples:

```text
meal rise + no recorded insulin → attention
meal rise + recent insulin → stay quiet initially
recent insulin + continued sharp worsening → re-escalate
low → meal → recovery → avoid premature insulin nag
high then steady recovery → resolve without repeat
CGM gap after rising reading → stale/unknown, not continued rise
```

Each fixture should contain only the small event sequence needed to reproduce the behaviour.

This converts research discoveries into permanent regression protection without requiring the entire private history in the repository.

**Do not commit the user's raw health history to GitHub.**

---

## 15. Privacy and data handling

Historical CGM/treatment history is sensitive health data.

For this personal project, the preferred workflow is:

```text
export/download privately
→ process locally
→ produce aggregate metrics + selected redacted fixtures
→ keep raw dataset outside Git
```

Repository fixtures should be:

- synthetic; or
- deliberately minimised/redacted slices with no unnecessary identifying metadata.

No Nightscout token, URL secret or raw personal export should be committed.

---

## 16. What historical replay can and cannot tell us

### It can tell us

- relative alert burden between policies;
- timing relative to real glucose trajectories;
- whether contextual suppression would have reduced repeated nags;
- whether a policy frequently waits too long;
- whether defer/re-alert rules behave sensibly;
- how often stale data occurs;
- whether a signal adds incremental value;
- whether rule changes cause regressions on known episodes.

### It cannot tell us

- what glucose would have done if the user had received and acted on a counterfactual alert;
- whether TIR would have improved;
- whether the user would have considered every simulated alert useful;
- whether an absent insulin record proves no insulin was taken;
- exact historical iOS wake/delivery timing if the source lacks receipt timestamps.

Therefore use replay to **select and de-risk policies**, then validate cognitive burden and glucose outcomes prospectively.

---

## 17. Recommended first replay report

For the first real-data experiment, aim for roughly 1–3 months if available, but let data quality determine the usable period.

### Section A — data quality

```text
period
reading count / coverage
gaps
insulin record coverage
carb record coverage
units / source
excluded periods
```

### Section B — baseline versus candidate

```text
interruptions/day
episodes alerted/day
repeats/episode
silent resolutions
first-alert timing
alerts after recorded insulin
stale-data events
```

### Section C — episode gallery

Show a small set of:

- clear wins;
- false/noisy alerts;
- missed/late episodes;
- edge cases;
- disagreements between candidate versions.

### Section D — parameter trade-off

Show the Pareto frontier for the small number of parameters under consideration.

### Section E — holdout result

Only after policy choices have been made from development/validation data, run once against the untouched holdout.

---

## 18. Suggested implementation sequence when we leave research

This is an implementation implication, not a request to build it yet.

1. Define pure `AttentionEngine`, `AttentionContext`, `AttentionDecision`, `AttentionReason`.
2. Define persisted/in-memory `AttentionEpisode` state machine.
3. Add deterministic `AttentionClock`.
4. Build Swift replay runner with in-memory adapters.
5. Define neutral JSON/CSV replay import schema.
6. Build Nightscout importer and/or V7-backup normaliser outside core engine.
7. Emit decision-trace JSON.
8. Add aggregate analysis script/notebook if useful.
9. Convert representative episodes into Swift regression fixtures.
10. Only then wire the same engine into live xDrip callbacks.

This order is attractive because we can exercise the core policy heavily **before it is allowed to send a single live notification**.

---

## 19. Reuse / adapt / build / avoid

| Area | Call |
|---|---|
| `mpereiragu` Nightscout-fetch/backtest ideas | **Reference / adapt concepts** |
| Existing predictor's MAE/RMSE as primary Attention metric | **Avoid** |
| Nightscout entries/treatments APIs | **Reuse as optional data source** |
| V7 selectable BG/treatment backup | **Reuse as optional data source** |
| V7 local treatment/BG model | **Reuse through adapters** |
| Pure production Attention evaluator | **Build once and reuse in replay** |
| Deterministic fake clock | **Build** |
| Discrete-event replay runner | **Build** |
| Decision trace | **Build** |
| Episode-level metrics | **Build** |
| Parameter sweeps / Pareto analysis | **Build lightly** |
| Separate Python clone of Attention rules | **Avoid** |
| Randomly shuffled time-series train/test split | **Avoid** |
| Treating missing insulin record as no insulin | **Avoid** |
| Committing raw personal health history to Git | **Avoid** |

---

## 20. Product-spec implications

1. `AttentionEngine` must be deterministic for a given context/clock/policy version.
2. The engine must not depend directly on UIKit, SwiftUI, UserNotifications, Core Data or wall-clock globals.
3. `AttentionEpisode` transitions and scheduled boundaries must be reproducible in replay.
4. Every decision needs stable reason codes suitable for traces and user-facing explanation.
5. The app should record trigger/evaluation metadata from day one so future replay can reconstruct what the app actually knew, not only physiological timestamps.
6. Raw source data should be normalised behind adapters; Nightscout and V7 backup are inputs, not architectural dependencies.
7. Missing treatment data remains unknown, never negative evidence.
8. Attention policies should be versioned so live behaviour can be compared with the exact historical policy that produced a trace.
9. Policy success is multi-objective: earlier useful attention **and** lower interruption burden.
10. Historical replay is a pre-live validation gate, not proof of improved clinical outcomes.
11. Representative real episodes should become privacy-safe regression fixtures.
12. A lightweight episode-review workflow should be considered because user-labelled “wanted attention / did not want attention” is stronger ground truth than inferred labels alone.

---

## 21. Open questions handed forward

### Pass 07 — IOB / COB

- What IOB representation should be available to `AttentionContext`?
- Can the IOB calculator itself be deterministic under a replay clock?
- What parameters must be captured/versioned so historical IOB is reproducible?
- Does IOB materially improve the Pareto frontier over simple “recent insulin” timing?

### Pass 10 — future personalised data model

- Which live timestamps and interaction metadata should we capture to move from physiology replay to high-fidelity execution replay?
- What explicit user feedback is worth storing for supervised/personalised attention preference later?

### Pass 11 — prediction

- If a future predictor is added, can its incremental value be tested as another context feature/ablation rather than making prediction the engine itself?

### Pass 12 — safety

- Which historical scenarios should become mandatory safety regression fixtures?
- What failure cases should automatically block a policy version from live use?

---

## 22. Key references

### Existing xDrip prediction/backtest reference

- `mpereiragu/xdripswift-predict`
- branch `claude/xdrip-glucose-ai-prediction-r9v9c3`
- `server/tools/backtest.py`

### xDrip V7 source reviewed read-only

- `xDrip/SwiftUIViews/Settings/DataManagementView.swift`
  - selectable BG-reading and treatment backup;
  - manifest counts/date summaries;
  - restore handling.
- `xDrip/SwiftUIViews/Settings/NightscoutImportView.swift`
  - historical BG/treatment/device-status import.

### Nightscout primary documentation

- `nightscout/cgm-remote-monitor` README — API v1 entries/treatments/profile endpoints and query examples.
- `lib/server/swagger.yaml` — Nightscout API v1 OpenAPI definition.
- `lib/api3/swagger.yaml` — Nightscout API v3 OpenAPI definition.

---

## Bottom line

Historical replay should become one of this project's strongest development tools, but only if it tests the **attention experience** rather than merely glucose prediction.

The target is not:

> “Which rule predicts glucose most accurately?”

It is:

> **“Which policy would have caught the situations that deserved attention, early enough to matter, while interrupting the user as little as possible?”**

And because historical response is counterfactual, the honest validation sequence is:

```text
historical replay
→ human episode review
→ privacy-safe regression fixtures
→ cautious live prospective testing
→ measure real cognitive burden + glucose outcomes
```
