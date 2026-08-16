# 03 — Other diabetes apps: alert and attention patterns

Status: **Complete**  
Research date: 2026-08-16

## Question

What interaction and decision patterns already exist in xDrip+, Loop, Trio/iAPS and AndroidAPS that are relevant to our Attention Engine — especially around IOB/COB, missed or unannounced meals, action-aware alerts, snoozing, recovery states, low-friction logging and multi-signal decision logic?

## Executive conclusion

The core product idea is still differentiated, but **many of its building blocks already exist separately in mature diabetes apps**.

The strongest patterns to borrow are:

1. **xDrip+: smart snoozing and direction-aware suppression.** It already contains options conceptually equivalent to “keep snoozing if glucose is heading in the right direction” and “don’t alert if glucose is heading in the right direction,” plus pre-emptive snoozing when the user has already treated a high/low.
2. **Loop: meal state is independent from bolus state.** Carbs can be saved without bolusing, predictions use IOB/COB, and Loop can detect a likely missed meal from a glucose excursion and prompt the user.
3. **Trio/iAPS: unannounced-meal detection.** UAM uses unexpected glucose deviation as evidence that the observed glucose is not matching the expected model. This is highly relevant as an Attention Engine signal, even though we should not reuse it to automate insulin dosing.
4. **Trio: external insulin can be logged into IOB without being delivered by the app.** This is unusually relevant to our MDI use case.
5. **AAPS: multi-condition automation is a mature pattern.** AAPS can combine contextual triggers and produce actions/notifications, and its algorithms explicitly gate behaviour on data reliability, IOB, COB, temp-target/recovery context and recent trends.
6. **AAPS/Trio/iAPS: recovery context changes behaviour.** High temporary targets used for hypo recovery can suppress aggressive insulin automation. The transferable lesson is that the same rising glucose trajectory can mean something very different immediately after a low.

None of these apps, however, appears to expose exactly the simple user-facing episode model we want:

> Ate → awaiting decision → handled / no insulin needed / waiting for recovery → monitor → resolve or re-alert.

Our opportunity is therefore **not to invent new diabetes physiology**, but to combine proven contextual signals into a much simpler, MDI-focused attention workflow.

---

## 1. xDrip+ — closest precedent for smarter alert policy

Source: NightscoutFoundation xDrip+ documentation and repository.

### Relevant existing capabilities

xDrip+ supports:

- glucose-level alerts;
- persistent-high alerts based on glucose staying above a threshold for a configured period;
- forecast-low alerts;
- pre-emptive snoozing;
- default snooze periods;
- re-raise intervals for unacknowledged alerts;
- Watch-based snooze;
- voice/keypad/Watch treatment entry;
- insulin and carbohydrate action curves;
- predictive simulation;
- smart snoozing / smart alerting.

Most importantly, the current strings/settings include explicit concepts equivalent to:

- **keep snoozing if glucose is heading in the right direction**;
- **don’t alert if glucose is heading in the right direction**;
- alerts can start snoozed and must persist before triggering;
- re-raise an alert sooner if it has not been acknowledged.

The xDrip+ snooze documentation also frames pre-emptive snooze around user action: if the user has already taken insulin for a high or eaten to treat a low, they can suppress the corresponding alert before it fires.

### What to borrow

**Strongly borrow/adapt the alert-policy ideas.**

In particular:

- direction-aware suppression;
- persistence before escalating;
- shorter re-raise when unacknowledged;
- pre-emptive “I’ve dealt with this” semantics;
- persistent-high duration as a signal, not merely the current threshold.

### What not to copy directly

xDrip+ mostly represents “already handled” as a **snooze**, not as structured context about what the user actually did.

For us:

```text
snooze 30 min
```

is weaker than:

```text
user action = insulin logged
or
user action = ate to treat low
or
user state = deliberately waiting for recovery
```

The latter lets the system re-evaluate intelligently rather than merely wait for a timer.

### Product implication

Our Attention Engine can be thought of as a semantic extension of xDrip+ smart snoozing:

> Smart snoozing asks “is this alert still relevant?”  
> Attention Engine asks “what situation are we in, has the user acted, and does this situation currently require attention?”

---

## 2. Loop — separate meal state from insulin state

Sources: LoopDocs.

Loop predictions use:

- CGM glucose;
- IOB;
- COB;
- therapy settings.

### Carbs can exist without a bolus

Loop explicitly supports **Save without Bolusing**. A meal/carb entry is therefore not treated as synonymous with insulin delivery.

This is a very important model for us.

Our `Ate` event should similarly mean only:

> food intake happened / meal context exists.

It should not mean:

> insulin was required, recommended or taken.

That distinction is fundamental for the user’s real behaviour, where some meals require no insulin and insulin may be deliberately delayed during a low or falling glucose state.

### Missed Meal Notifications

Loop has a particularly relevant feature: **Missed Meal Notifications**.

Loop detects glucose excursions that suggest carbs may have been consumed without being entered. The notification can estimate when the meal may have happened and how many carbs may already have been absorbed, after which the user can correct the entry.

### Why this matters to us

This validates an important future signal:

```text
unexpected rise + no meal context
→ possible missed/unlogged meal
```

However, our product should initially phrase this more cautiously than a dosing system:

> “Glucose is rising in a way that could be consistent with food, and I can’t see a recent meal logged.”

### Dynamic absorption

Loop updates carb absorption estimates using observed glucose response while the meal is active.

We should not attempt to reproduce Loop’s full dosing model, but the underlying concept is useful:

> expected response should be revised as actual glucose arrives.

That principle belongs directly in future Attention Episodes.

### Low-friction interaction precedent

Loop supports meal entry from Apple Watch as well as the phone.

The Watch flow is still relatively detailed because Loop requires carb amount/absorption information. Our MDI attention product can potentially make the Watch interaction substantially lighter with a one-tap `Ate` acknowledgement followed by optional detail.

---

## 3. Trio — especially relevant to our MDI use case

Sources: current TrioDocs.

Trio is pump/AID-oriented, but several patterns transfer unusually well.

### Unannounced Meals (UAM)

Trio can use UAM to react when glucose rises unexpectedly even when no carbohydrates were logged, or when logged carbohydrate amount was wrong.

Conceptually, UAM compares **expected glucose behaviour with observed glucose behaviour** and treats persistent positive deviation as evidence of an unmodelled influence such as food.

For us, the useful abstraction is not “dose SMB.” It is:

```text
observed trajectory is materially worse than the trajectory implied by known context
→ attention evidence increases
```

This could eventually help distinguish:

- ordinary post-meal rise with active insulin;
- unexpectedly strong rise despite logged insulin;
- rise with no meal and no insulin context;
- rebound after low treatment.

### External insulin logging

Trio’s bolus calculator supports an **External Insulin** option: insulin can be added to IOB without being delivered by Trio.

This is one of the most directly reusable product ideas found in this pass.

For an MDI user, “insulin taken outside the app” is the normal case. The app needs to model it as active insulin without implying that the app delivered it.

**Recommendation:** our eventual treatment UX should explicitly support the semantic distinction:

```text
logged insulin / external insulin
≠
app-delivered insulin
```

xDrip treatment storage can remain the underlying source of truth; the Attention Engine can derive approximate local IOB from those logged injections.

### Shortcuts

Trio exposes iOS Shortcut actions including:

- Add Carbs;
- Bolus (optional/explicitly enabled);
- List State;
- activate/cancel temp targets and overrides.

`List State` can expose current glucose, trend, age, delta, IOB and COB.

This reinforces the feasibility and value of making our own actions available through App Intents/Shortcuts rather than requiring the main app to be opened.

### Watch

Trio’s Watch app can show BG/IOB/COB and enter carbs/boluses/temp targets.

Again, the infrastructure pattern is relevant, but our interaction can be intentionally simpler.

### Recovery context

Trio guidance highlights an important safety/context distinction: high temporary targets are often used during low recovery, and aggressive SMB behaviour is normally disabled in that context.

This directly maps to our proposed `waitingForRecovery` state.

A rapid rise after treating a low should **not** be interpreted the same way as an unexplained rapid rise from a normal starting glucose.

---

## 4. iAPS — similar UAM concepts, useful cautionary evidence

Current iAPS remains an experimental iOS AID system built on OpenAPS algorithms and Loop frameworks.

Its documentation explicitly positions UAM as a way to cope with:

- missed carb entries;
- incorrect carb estimates;
- fewer manual corrections.

Its prediction UI separates multiple possible trajectories including an unannounced-meal trajectory.

### Particularly useful low-recovery lesson

iAPS troubleshooting documentation describes rebound-low scenarios where fast carbs after a low can cause glucose to rise quickly; aggressive insulin response to that rise can then contribute to another low. The suggested AID mitigation is a temporary higher target with SMB suppressed for a period.

We should not copy the dosing instruction, but the context lesson is important:

```text
low → corrective carbs → fast rise
```

must carry a strong mitigating signal in the Attention Engine.

This reinforces our earlier proposed episode state:

`waitingForRecovery`

rather than simply suppressing alerts for a fixed number of minutes.

### Current iAPS development signal

The 2026 iAPS 7.x release work includes instant IOB/COB updates after loops, deliveries and carb changes, demonstrating how central live treatment-state updates are to modern diabetes decision systems.

---

## 5. AndroidAPS — strongest precedent for a general context engine

Sources: current AndroidAPS 3.4 documentation.

AAPS continuously reasons over combinations of:

- BG;
- IOB;
- COB;
- sensitivity;
- temp targets;
- meal state;
- observed deviations;
- data reliability/noise;
- time/location and other user-defined automation context.

### Automation rules

AAPS has a general Automation feature in which one or more **triggers/conditions** are combined and then one or more actions are executed.

Examples include glucose, IOB, time, location/Wi-Fi and other context.

This is conceptually close to an Attention Engine, but it is exposed as a power-user rule-building system.

### Product lesson

We should borrow the **internal architecture**, not necessarily the UI.

Internally:

```text
multiple signals
    ↓
conditions / evidence
    ↓
state decision
    ↓
action
```

Externally, the user should not have to construct rules such as:

```text
IF glucose > X
AND delta > Y
AND IOB < Z
AND meal age < N
THEN notify...
```

The product’s value is that sensible rules are already encoded and can later personalize themselves.

### Data-quality gating

AAPS explicitly restricts some aggressive behaviour unless the glucose source is considered reliable and filtered enough, because noisy glucose can falsely appear to be a rapid rise.

This is an important requirement for us too.

Attention decisions should include data confidence / freshness / noise so that:

- poor data does not trigger confident contextual conclusions;
- stale/missing CGM becomes its own attention state;
- a single noisy delta does not cause escalation.

### Use short history, not one arrow

AAPS has options that use a short-average delta over recent readings to reduce sensitivity to noisy single readings.

This independently supports the conclusion from upstream xDrip issue #694: **do not base suppression/escalation on one trend arrow.**

### Carb-required notifications

AAPS can generate carbs-required notifications based on its prediction/context model, with snooze options such as 5, 15 or 30 minutes.

The useful pattern is not the carb recommendation itself; it is that **snooze duration is short and the need is continuously re-evaluated from physiological context**.

This matches the user’s dissatisfaction with one-hour high-alert snoozes.

### COB is inferred from observed glucose response

AAPS adjusts carb absorption based on glucose deviations and can detect when COB may be wrong.

Again, we should not reproduce the full dosing model initially. But it establishes an important future personalization principle:

> user-entered meal context is a hypothesis that actual glucose data can confirm, revise or contradict.

---

## 6. Cross-app pattern matrix

| Capability / pattern | xDrip+ | Loop | Trio / iAPS | AAPS | Our project |
|---|---:|---:|---:|---:|---|
| Threshold glucose alerts | Yes | Limited/AID context | Yes | Yes | Reuse xDrip |
| Direction-aware alert suppression | **Yes** | Prediction affects dosing | Indirect | Indirect | **Core** |
| Persistent-high duration | **Yes** | Prediction-based | Prediction-based | Prediction-based | **Core signal** |
| Pre-emptive “already treated” snooze | **Yes** | Treatment state implicit | Treatment state implicit | Treatment state implicit | **Explicit semantic action** |
| IOB | Yes/modelled | **Core** | **Core** | **Core** | **Approx local IOB early** |
| COB | Yes/modelled | **Core** | **Core** | **Core** | Later/optional |
| Meal without insulin | Treatment possible | **Yes** | **Yes** | **Yes** | **Ate must be independent** |
| Missed/unannounced meal detection | Prediction model adjacent | **Missed Meal** | **UAM** | **UAM** | **Attention signal, not auto-dose** |
| Low-recovery context | Basic snooze | Prediction/IOB | **Temp-target context** | **Temp-target context** | **Explicit waitingForRecovery** |
| Multi-signal automation | Limited | Internal algorithm | Internal algorithm | **General rule system** | **Internal engine, simple UX** |
| Short re-alert/snooze | Configurable | Alert-specific | Notification-specific | **5/15/30m example** | **Dynamic 5–10m when unresolved** |
| Watch treatment entry | **Yes** | **Yes** | **Yes** | **Yes** | Later, ultra-light actions |
| iOS Shortcuts/App Intents | N/A Android | Some ecosystem integration | **Yes** | N/A Android | **High priority** |
| External/manual insulin → IOB | Treatment modelling | Pump delivery mainly | **Explicit External Insulin** | Virtual/manual contexts possible | **Very important** |

---

## 7. The most useful concepts to adapt

### A. `Ate` must not imply insulin

Loop/Trio/AAPS all separate carbohydrate/meal context from insulin delivery.

Our even lighter `Ate` event is therefore conceptually sound.

It can later be enriched with:

- approximate carbs;
- photo-derived food/carbs;
- insulin treatment;
- resolution state.

### B. Add an explicit external-insulin concept

Borrow Trio’s semantic clarity:

> insulin was taken and should contribute to active-insulin context, but the app did not deliver it.

For our MDI use case this should be the normal path, not a special case.

### C. Use **unexpected deviation** as evidence

Borrow the UAM concept without borrowing automated dosing:

> glucose behaving materially differently from what known context suggests is itself an attention signal.

Initially this can be simple and rule-based. Later it can become personalized.

### D. Make recovery a real state

Borrow the safety logic behind high temp targets / suppressed SMB after lows, but represent the human intent directly:

`waitingForRecovery`

This state can suppress inappropriate high/rise nags while still allowing escalation if the situation becomes genuinely concerning.

### E. Contextual snooze should continuously reassess

xDrip+ smart snoozing and AAPS short re-alert patterns both support this direction.

Instead of:

```text
snooze until 20:30
```

prefer:

```text
defer while resolving;
re-evaluate each new reading;
alert sooner if risk increases;
resolve silently if evidence improves.
```

### F. Internally use a rule/evidence engine; externally keep it simple

AAPS proves that rich condition composition is useful, but its complexity is inappropriate for our primary UX.

The user-facing choices should remain things like:

- Ate
- Insulin taken
- I’m handling this
- No insulin needed
- Waiting for recovery
- Remind me soon

The internal engine can combine many signals without requiring the user to understand its rule graph.

---

## 8. Important things *not* to copy

### Do not turn this into a pump/AID controller

Loop, Trio/iAPS and AAPS are primarily insulin-delivery systems. Their predictions are designed partly to decide how much insulin to automate.

Our early product is different:

> decide whether the user’s attention is needed.

Keeping that distinction should make the system simpler, safer and much easier to validate.

### Do not require precise carbs as the entry ticket

Mature AID apps get significant value from detailed carb data because they are dosing insulin.

Our product goal is lower cognitive load. Requiring accurate carb entry for every meal would undermine the central value proposition.

### Do not expose AAPS-like configuration complexity initially

The internal model may eventually be sophisticated; the surface should not require dozens of thresholds and algorithm settings.

### Do not infer “insulin taken” from glucose response

UAM-like reasoning can infer that *something is unmodelled*. It cannot safely prove that insulin was or was not taken.

The neutral wording rule from earlier research remains essential.

---

## 9. Proposed Attention Episode model after this research

Still research-stage, not final spec:

```text
Episode
├─ trigger
│  ├─ Ate
│  ├─ unexpected rise
│  ├─ persistent high
│  ├─ low recovery
│  └─ other attention signal
│
├─ known context
│  ├─ recent glucose history
│  ├─ recent treatment entries
│  ├─ approximate IOB
│  ├─ meal context
│  ├─ exercise context (later)
│  └─ data quality/freshness
│
├─ user state
│  ├─ unresolved
│  ├─ handling
│  ├─ insulinLogged
│  ├─ noInsulinNeeded
│  ├─ waitingForRecovery
│  └─ remindSoon
│
└─ engine state
   ├─ quiet
   ├─ monitor
   ├─ remind
   ├─ escalate
   └─ resolved
```

This combines the useful pieces we found while remaining materially simpler than a closed-loop controller.

---

## 10. Product-spec implications

Carry forward:

1. **Meal state and insulin state must be separate.**
2. **Manual/external insulin must contribute to active-insulin context.**
3. Approximate local IOB is now even more strongly justified as an early feature.
4. `waitingForRecovery` should be a first-class episode/user state, not merely a timer.
5. Alert relevance should use recent trajectory/persistence rather than a single trend arrow.
6. Data quality/noise/freshness must be explicit inputs to attention decisions.
7. A future “possible unlogged meal” signal can use unexpected glucose deviation, inspired by Loop Missed Meal and UAM, but should not claim certainty.
8. Dynamic snoozing should re-evaluate on each reading and resolve early when the situation improves.
9. App Intents/Shortcuts and Watch actions have strong precedent and should be investigated in pass 04.
10. The engine can internally resemble a rule/evidence system, but the user-facing model should remain a handful of meaningful actions/states.
11. We should validate **attention decisions**, not insulin-dose quality, in our historical replay tooling.

---

## Reuse / adapt / build / avoid

| Pattern | Call |
|---|---|
| xDrip+ smart snoozing / direction-aware suppression | **Adapt strongly** |
| xDrip+ pre-emptive snooze | **Adapt into explicit acknowledgement state** |
| xDrip+ persistent-high duration | **Reuse concept** |
| Loop independent carb + bolus state | **Reuse concept** |
| Loop Missed Meal detection | **Adapt as cautious attention signal** |
| Loop dynamic absorption model | **Study later; do not need for MVP** |
| Trio External Insulin → IOB | **Adapt strongly for MDI** |
| Trio/iAPS UAM | **Adapt signal concept; never copy auto-dose behaviour** |
| AAPS multi-condition Automation | **Adapt architecture, not power-user UX** |
| AAPS data-quality gating / short-average trend | **Adapt strongly** |
| Closed-loop dosing algorithms | **Do not make part of early product** |

---

## Open questions handed to later passes

- **Pass 04 — low-friction logging:** Can `Ate`, external insulin and episode acknowledgements be written from App Intents, notification buttons, widgets and Watch with minimal friction?
- **Pass 05 — iOS notification constraints:** How close can we get on iOS to xDrip+ style smart snoozing and AAPS-style continuous contextual re-evaluation?
- **Pass 06 — historical replay:** How do we test false nags, missed attention moments and re-alert timing against the user’s real history?
- **Pass 07 — IOB/COB:** What insulin action model is appropriate for logged injections and what assumptions are safe when logging is imperfect?
- **Pass 09 — Health/Watch context:** Which activity/exercise signals are reliable enough to function like AAPS temp-target context without requiring explicit user entry?

## Key references

- xDrip+ repository: https://github.com/NightscoutFoundation/xDrip
- xDrip+ snooze docs: https://navid200.github.io/xDrip/docs/Snooze.html
- xDrip+ persistent-high docs: https://navid200.github.io/xDrip/docs/Alerts/PersistentHigh.html
- Loop meal entries: https://loopkit.github.io/loopdocs/operation/features/carbs/
- Loop settings / Missed Meal Notifications: https://loopkit.github.io/loopdocs/loop-3/settings/
- Loop closed-loop concepts: https://loopkit.github.io/loopdocs/operation/loop/close-loop/
- Loop Watch meal entry: https://loopkit.github.io/loopdocs/operation/features/watch/
- Trio notifications: https://triodocs.org/configuration/settings/notifications/trio-notifications/
- Trio bolus calculator: https://triodocs.org/usage/features/bolus-calculator/
- Trio Shortcuts: https://triodocs.org/configuration/settings/features/shortcuts/
- Trio SMB/UAM: https://triodocs.org/configuration/settings/algorithm/smb-settings/
- Trio Watch: https://triodocs.org/configuration/settings/devices/smart-watch/
- iAPS repository: https://github.com/Artificial-Pancreas/iAPS
- iAPS notifications: https://iaps.readthedocs.io/en/main/settings/services/notifications.html
- AndroidAPS current docs: https://androidaps.readthedocs.io/en/latest/
- AndroidAPS key features: https://androidaps.readthedocs.io/en/latest/DailyLifeWithAaps/KeyAapsFeatures.html
- AndroidAPS Automation: https://androidaps.readthedocs.io/en/latest/DailyLifeWithAaps/Automations.html
- AndroidAPS COB calculation: https://androidaps.readthedocs.io/en/latest/DailyLifeWithAaps/CobCalculation.html
