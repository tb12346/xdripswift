# Research 08 — Insulin on Board + Carbs on Board

**Status:** Complete  
**Research date:** 2026-08-16

## Executive conclusion

**Approximate local IOB should enter the product early. Classical COB should not.**

IOB is directly useful for the core attention problem: the same glucose rise means something different when there is no recently logged insulin versus when a recent logged injection still has substantial modeled activity remaining. We can calculate this locally from xDrip treatment history using a proven insulin-action model without making insulin-dose recommendations.

COB is a weaker early signal for this product. A meaningful COB estimate requires a carb quantity plus assumptions about absorption, and the intended workflow explicitly allows a one-tap `Ate` event with no carb count. Loop and AAPS improve COB by observing glucose deviations and therapy settings, but that adds substantially more modeling and can misattribute non-food effects to carbs. We should not manufacture a precise-looking COB number from incomplete meal data.

Recommended product sequence:

1. **Recent insulin evidence** — immediately useful from treatment logs.
2. **Modeled IOB + insulin activity/phase** — early Attention Engine input.
3. **Meal recency / Ate state** — use even when carbs are unknown.
4. **Optional carb amount** — retain as ordinary treatment data when available.
5. **COB / dynamic meal absorption** — later, once carb estimates and validation justify it.

The Attention Engine should use IOB to change **attention urgency**, not to calculate or recommend a dose.

---

## Questions for this pass

- Can we calculate useful IOB locally from existing xDrip treatment records?
- Is there existing xDrip/Loop/OpenAPS code worth reusing?
- What does IOB mean for an MDI user rather than a pump user?
- Is raw IOB enough, or do we need insulin activity as well?
- How reliable is COB when carb entries or absorption information are incomplete?
- Should IOB/COB live in xDrip, Nightscout, or a new data store?
- How should these signals enter historical replay and the Attention Engine?

---

## 1. IOB is a high-value Attention Engine signal

For this product, IOB is useful even without glucose prediction or dosing calculations.

Consider two otherwise identical situations:

```text
A. glucose rising persistently
   meal 35 minutes ago
   no recent insulin recorded
   modeled active insulin ≈ 0

B. glucose rising persistently
   meal 35 minutes ago
   insulin logged 18 minutes ago
   substantial modeled insulin still active
```

These should not create the same attention policy.

Situation A is stronger evidence that the user may have forgotten an action. Situation B is stronger evidence that the situation may already be being handled and deserves more patience before another interruption, subject to glucose severity and trajectory.

This does **not** require answering "how much insulin should be taken?" It only requires answering "how much evidence do we have that insulin has already been taken and is still active?"

### Product implication

IOB should enter the Attention Engine as a **mitigating/context signal**, not as a bolus recommendation input exposed to the user.

---

## 2. xDrip4iOS already has a strong local IOB prototype

Upstream PR #366, `Implemented Insulin on Board`, is still open and unmerged.

Source:
- https://github.com/JohanDegraeve/xdripswift/pull/366

The PR implemented:

- a local `InsulinOnBoardCalculator`;
- configurable insulin activity duration;
- configurable peak time;
- calculation from existing `.Insulin` `TreatmentEntry` records;
- historical IOB calculation for charting;
- optional chart display.

The implementation deliberately makes the dose-level insulin-decay calculation a **pure function** and cites both OpenAPS and Loop work as the mathematical basis.

The core shape is good for us even though the surrounding 2022 UIKit/UI work is obsolete for V7:

```text
TreatmentEntry insulin records
        ↓
insulin action model
        ↓
sum contribution of each active dose
        ↓
modeled IOB at time t
```

The PR defaults were 300 minutes duration and 75 minutes peak, but those old defaults should not simply be copied into our product as medical assumptions. The valuable part is the implementation pattern and exponential model, not the historic settings UI.

### Reuse decision

**Reference/rebuild rather than merge wholesale.**

Reasons:

- the PR is four years old and unmerged;
- much of its UI integration targets the old `RootViewController` architecture;
- V7 gives us a cleaner coordinator/domain-service boundary;
- we want an engine-facing model, not primarily a chart feature;
- the model should be independently testable and replayable.

---

## 3. Current Loop validates the exponential-model approach

Loop continues to model insulin using exponential activity curves and describes separate rapid-acting and ultra-rapid presets built from the same underlying model.

Sources:
- https://loopkit.github.io/loopdocs/operation/algorithm/prediction/
- https://loopkit.github.io/loopdocs/faqs/algorithm-faqs/
- https://loopkit.github.io/loopdocs/version/code-custom-edits/

Loop's model distinguishes several related ideas:

- **insulin delivered**;
- **insulin remaining / active insulin (IOB)**;
- **insulin activity over time**;
- **modeled glucose effect**, which additionally requires insulin sensitivity.

For every dose, an action curve determines how much insulin remains active over time. Contributions from active doses are combined.

This distinction matters to us because two moments with a similar amount of IOB can have different near-term activity depending on where those doses are on their action curves.

### Recommendation

Internally calculate at least:

```text
modeled active insulin remaining
current modeled insulin activity / phase
last logged insulin timestamp
contributing dose count
```

We do **not** need to convert that into a predicted glucose drop for the first Attention Engine version.

---

## 4. IOB for MDI is simpler than pump IOB — but semantics matter

Loop/AAPS pump IOB includes insulin delivered above or below scheduled basal and can therefore become negative when basal insulin has been reduced or suspended.

That is not the natural model for this project.

For an MDI user, our first useful quantity is:

> active rapid/mealtime/correction insulin remaining from logged injections.

If we are summing ordinary positive injection records, this value should not become negative.

### Long-acting insulin

Long-acting basal insulin should **not** simply be mixed into the same short-acting IOB number. Its physiological purpose and action profile are different, and doing so would create a misleading attention signal.

If long-acting injections are logged later, they should be represented separately as basal context, adherence/history, or a distinct model — not treated as ordinary meal/correction IOB.

---

## 5. Trio provides a strong MDI precedent: external insulin can enter IOB

Trio explicitly supports logging **External Insulin**: insulin that was given outside Trio can be added to IOB without Trio delivering it.

Sources:
- https://triodocs.org/usage/features/bolus-calculator/
- https://github.com/nightscout/Trio/releases

That is directly relevant to our MDI use case. It demonstrates that an iOS diabetes system can use manually entered insulin as algorithmic context without requiring pump delivery to be the source of truth.

### Product implication

The user's one-tap/text-input insulin log can immediately become an input to local IOB. We do not need pump integration.

---

## 6. Nightscout already validates the alert-suppression use case

Nightscout's existing plugins are unusually relevant to our product concept.

Its `iob` plugin calculates IOB from insulin treatments and profile information. Its `cob` plugin calculates COB from carb treatments and profile information.

More importantly, **Bolus Wizard Preview (`bwp`)** already uses IOB to alter alert behaviour. Nightscout documents that BWP can snooze high-glucose alarms when there is enough IOB to account for the high, with a configurable short snooze interval; the documented default `BWP_SNOOZE_MINS` is 10 minutes.

Source:
- https://github.com/nightscout/cgm-remote-monitor#plugins

This is conceptually very close to our intended behaviour:

```text
high/rising glucose
        +
active insulin evidence
        ↓
less immediate repetition / reassess soon
```

We should borrow this **attention concept**, not BWP's bolus-calculation behaviour.

### Product implication

IOB-aware alert suppression is not an untested UX idea. There is long-standing Nightscout precedent for "high glucose + active insulin = snooze/reassess rather than immediately repeat the same alarm."

---

## 7. Never present modeled IOB as proof of insulin actually taken

The model is only as complete as the treatment history feeding it.

This project already has a crucial rule:

> no insulin logged ≠ no insulin taken.

The same rule applies to IOB.

If an injection was taken but not logged, our calculated IOB will be too low. Therefore the engine should conceptually treat the quantity as:

> **modeled IOB from recorded insulin**

rather than "insulin currently in your body."

### Recommended internal shape

```swift
struct ActiveInsulinContext {
    let modeledUnitsRemaining: Double
    let modeledActivity: Double
    let lastRecordedDoseAt: Date?
    let contributingDoseCount: Int
    let provenance: InsulinEvidenceProvenance
}
```

The exact types belong in the later product/technical spec, but provenance should survive into the design.

### Alert-language implication

Prefer language such as:

> "I can see insulin logged recently."

or:

> "I can't see any recent insulin logged."

Do not imply the app knows with certainty whether an injection occurred.

---

## 8. V7 treatment storage is sufficient for early IOB, but not insulin subtypes

Current V7 `TreatmentEditorViewModel` supports:

- `.Insulin`
- `.Carbs`
- `.Exercise`
- `.BgCheck`
- `.Note`

and saves a treatment timestamp and numeric value into existing `TreatmentEntry` storage.

Source:
- Paul Plant `v7-beta`, `xDrip/SwiftUIViews/Treatments/TreatmentEditorViewModel.swift`

There is no insulin subtype/action model field in this UI/data path. An insulin treatment is essentially "X units of insulin at time T" plus general notes/source fields.

### Early recommendation

Do **not** add insulin-type friction to every quick log.

Instead, if the personal prototype uses one rapid/ultra-rapid insulin for meal/correction injections, configure the insulin action model once in settings and apply it to ordinary insulin treatment entries.

If the product later needs to support multiple short-acting insulin types or long-acting injections simultaneously, extend treatment metadata then.

---

## 9. Architectural boundary for IOB

Do not put Core Data reads or UI logic inside the mathematical model.

Recommended conceptual boundary:

```text
TreatmentEntry / Nightscout import
            ↓
TreatmentHistoryProviding
            ↓
      InsulinDoseEvent[]
            ↓
     InsulinActivityModel
            ↓
     ActiveInsulinContext
            ↓
       AttentionEngine
```

The core model should be a deterministic pure Swift component so the exact same implementation can be used for:

- live evaluation;
- historical replay;
- unit tests;
- future model comparison.

The V7 coordinator can wire providers and trigger recalculation when a treatment or fresh glucose reading arrives.

---

## 10. What IOB should do to the Attention Engine

IOB should modify urgency, not act as a binary suppression switch.

Candidate behaviour to test in replay:

```text
persistent rise + no recent recorded insulin + ~0 modeled IOB
    → stronger evidence attention is needed

persistent rise + recent insulin + high current modeled activity
    → allow more time / reduce repeat-alert urgency

persistent rise + recent insulin but activity now in tail
    → less mitigating than a fresh/active dose

severe or accelerating rise despite substantial active insulin
    → do not blindly suppress; worsening trajectory can outweigh mitigation

falling glucose + active insulin
    → greater reason to avoid nagging for additional action
```

The specific thresholds belong in the product spec and replay tuning, not this research pass.

### Important anti-pattern

Do not implement:

```text
IOB > X  => silence high alerts
```

as an unconditional rule.

The engine must still consider glucose level, persistence, trajectory, worsening, meal context, stale data and prior episode state.

---

## 11. COB is substantially harder than IOB

A classical COB number is not just "carbs entered minus time."

Modern Loop and AAPS algorithms update carb absorption using observed glucose behaviour and therapy settings.

Loop compares observed glucose change with the glucose change expected from insulin alone; the difference is an **Insulin Counteraction Effect (ICE)** and is used to estimate dynamic carb absorption.

Sources:
- https://loopkit.github.io/loopdocs/operation/features/carbs/
- https://loopkit.github.io/loopdocs/operation/algorithm/prediction/
- https://loopkit.github.io/loopdocs/operation/features/ice/

AAPS similarly estimates absorbed carbs from glucose deviations together with insulin sensitivity and carb ratio.

Source:
- https://androidaps.readthedocs.io/en/latest/DailyLifeWithAaps/CobCalculation.html

This means a credible dynamic COB implementation starts pulling in:

- entered carb amount;
- assumed absorption duration/type;
- glucose response;
- insulin action;
- insulin sensitivity;
- carb ratio;
- minimum absorption rules;
- handling of activity/stress/other deviations.

That is much more model-heavy than local IOB.

---

## 12. COB has a particular mismatch with this product's intended workflow

A core interaction is deliberately:

> **Ate** — no carb number required.

That creates a valuable meal event but not enough information for a conventional grams-of-carbs-on-board estimate.

We should not convert:

```text
Ate at 13:05
```

into a fabricated point estimate such as:

```text
COB = 35 g
```

just to satisfy an algorithm.

The product can get substantial value from simpler, more honest signals:

```text
meal happened
minutes since meal
carb amount known? yes/no
optional carb estimate/range
observed glucose response since meal
```

### Recommendation

In the early Attention Engine, use **meal state + meal recency** rather than classical COB when carb quantity is unknown.

---

## 13. Even when carbs are entered, COB is uncertain

Loop's own documentation notes that carb absorption can vary with meal composition and physiological context, and that dynamically inferred absorption can misattribute other positive glucose effects to entered carbs until those carbs are considered absorbed.

That matters for this project because a precise COB number can look more authoritative than it is.

### Product implication

If we introduce COB later, treat it as an estimate with provenance/uncertainty rather than objective ground truth.

A future meal-photo system may naturally produce **carb ranges**, not exact grams. That suggests the long-term model may be better represented as a probabilistic/range-based meal context rather than immediately collapsing every uncertain meal into one exact COB number.

---

## 14. Nightscout IOB/COB should be a comparison source, not the live source of truth

Nightscout can calculate IOB and COB through server-side plugins, but the live app should calculate the context locally.

Reasons:

- Attention logic must work offline;
- the same model should run in live and replay modes;
- treatment arrival from Nightscout may be delayed;
- we need explicit control over model semantics and provenance;
- Nightscout/plugin values can still be useful as a comparison baseline during validation.

### Historical replay

Nightscout gives us a straightforward route to reconstruct modeled IOB over historical periods:

```text
Nightscout insulin treatments
        ↓
normalize to InsulinDoseEvent
        ↓
our local insulin action model
        ↓
IOB/activity at every replay timestamp
```

We should recompute it ourselves instead of importing a server-calculated IOB series so historical and live logic remain identical.

---

## 15. Backtesting questions for IOB

Pass 06 gives us the right evaluation framework. Add IOB as an ablation rather than assuming it helps.

Compare at least:

1. glucose-only attention logic;
2. glucose + `recent insulin logged` boolean/time-since-dose;
3. glucose + modeled IOB;
4. glucose + modeled IOB + current insulin activity/phase.

Measure effects on:

- alerts shortly after insulin was logged;
- repeated alerts during an already-treated rise;
- high episodes where no insulin was recorded;
- delayed escalation when glucose continues worsening despite insulin;
- false reassurance/suppression;
- total interruptions/day;
- repeat alerts/episode.

Historical missing treatment data remains **unknown** and should not be labeled as a forgotten dose.

---

## Reuse / adapt / build / avoid

| Component | Decision | Why |
|---|---|---|
| Existing xDrip `TreatmentEntry` insulin history | **Reuse** | Already local and Nightscout-synced source of recorded injections |
| PR #366 mathematical approach | **Reference / adapt** | Good pure-function exponential model; old UI architecture should not be merged |
| Loop exponential insulin model concepts | **Adapt** | Mature distinction between remaining insulin and activity |
| Trio external-insulin pattern | **Reuse concept** | Strong precedent for manually administered MDI insulin entering IOB |
| Nightscout IOB plugin | **Comparison/reference** | Useful baseline, but remote calculation should not be runtime dependency |
| Nightscout BWP IOB-aware snoozing | **Reuse concept** | Very close to our attention use case |
| Current V7 Treatments UI | **Reuse** | Insulin units/timestamp already supported |
| Per-dose insulin-type picker | **Avoid early** | Adds friction to the highest-frequency action |
| Long-acting insulin in rapid IOB | **Avoid** | Different semantics/action profile |
| Classical COB in early Attention Engine | **Defer** | Requires carb quantity and absorption assumptions not guaranteed by workflow |
| Fake COB from `Ate` | **Avoid** | False precision |
| Direct bolus recommendation from IOB | **Avoid** | Outside current product goal and materially raises safety/regulatory stakes |

---

## Product-spec implications

1. **IOB should be an early capability, not a late prediction feature.** It directly improves action-aware alerting.
2. Keep the insulin model **pure, local and testable**, behind a provider interface rather than coupled to Core Data/UI.
3. Use existing `TreatmentEntry` insulin records as the recorded-dose source of truth.
4. Model both **remaining active insulin and current insulin activity/phase**; the latter may be more informative for short-term attention decisions.
5. Call the signal **modeled/recorded IOB** internally and in explanatory UI where needed; never imply missing logs prove no injection occurred.
6. Apply one configured rapid-insulin action model to quick insulin logs initially. Do not require insulin subtype on every entry.
7. Keep long-acting basal insulin separate if/when it is added.
8. IOB should **modulate** attention urgency, never automatically suppress severe/worsening situations.
9. Do not make direct insulin-dose recommendations from the IOB implementation.
10. Treat `Ate` as meaningful meal context even with unknown carbs; **COB is not required for the first useful meal-aware engine.**
11. Delay dynamic COB until after meal-photo/carb-estimation and personalization research clarifies whether the extra model complexity earns its keep.
12. Historical replay should explicitly test whether full IOB/activity adds value beyond the simpler `time since insulin log` signal.

---

## Open questions for later spec / implementation

- Which rapid/ultra-rapid insulin action preset should the personal prototype use, and should it be a single user setting?
- Does the user want long-acting insulin logged in this app at all, or only rapid meal/correction insulin?
- Should current IOB be visible in the UI, or remain primarily an internal Attention Engine input?
- Is modeled insulin **activity** materially better than simple time-since-dose for our alert policy after replay testing?
- When meal-photo estimates arrive, should uncertain carbs feed a range/confidence model rather than a single COB number?
- Does a full dynamic COB model add enough attention benefit to justify requiring ISF/carb-ratio/absorption settings, or can learned meal-response context do the job more naturally later?

---

## Bottom line

**Build IOB early; defer COB.**

The project does not need to become an AID system to benefit from insulin-action modeling. A small local model of **recorded active insulin + current activity phase** can make the Attention Engine much better at recognising when a rising glucose situation has already been acted on.

For meals, preserve uncertainty. `Ate` is already useful context. An honest "meal happened, carbs unknown" signal is better for this product than a sophisticated-looking but poorly grounded COB estimate.
