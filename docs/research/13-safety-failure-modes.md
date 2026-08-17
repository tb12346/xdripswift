# Research 13 — Safety + Failure Modes

**Status:** Complete  
**Research date:** 2026-08-17

## Executive conclusion

**Separate core glucose safety from contextual attention management. Personalisation may help the app stay quiet, remind or escalate, but it must never be the only path protecting urgent glucose/data-health conditions.**

The recommended architecture is a small deterministic **Core Safety Monitor** — a “safety island” — alongside the richer Attention Engine:

```text
new glucose / treatment / user action / context
                    |
          +---------+----------+
          |                    |
          v                    v
 Core Safety Monitor      Attention Engine
 small + deterministic    contextual + stateful
 low dependency           quiet/watch/remind/escalate
          |                    |
          |                    +--> personal calibration
          |                    +--> similar episodes
          |                    +--> learned risk signal
          |                         (optional / bounded)
          v
 core glucose / data-health alerts
```

The Core Safety Monitor should remain functional if any of these fail or are wrong:

- Attention episode state;
- Nightscout;
- HealthKit / Apple Watch context;
- meal-photo estimation;
- personalised retrieval;
- learned models;
- optional network services.

The Attention Engine can still reduce cognitive burden aggressively, but its quieting must be **bounded, reversible and evidence-driven**:

1. every suppression/defer state expires or has a maximum safety ceiling;
2. every fresh glucose reading re-evaluates the situation;
3. worsening evidence can override `Handling it`, `No insulin needed`, `Waiting for recovery`, recent insulin and other mitigating context;
4. stale, missing, contradictory or low-confidence data reduces confidence rather than increasing it;
5. if an optional model cannot make a trustworthy inference, it abstains and deterministic policy takes over.

This pass deliberately does **not** choose exact urgent thresholds, maximum silence durations or treatment-specific numeric limits. Those are product-policy decisions that must be defined conservatively, tested in historical replay, verified on-device and revisited during the later safety/product-spec phase. Research should establish the invariants first, not invent precision.

---

## 1. Safety is not the same thing as attention optimisation

The product has two related but different jobs:

```text
Safety question:
“Is there a glucose/data condition that must remain visible regardless of context?”

Attention question:
“Given everything we know, does this non-emergency situation need the user's attention now?”
```

Trying to put both into one personalised score creates a dangerous coupling. A bug in meal context, IOB, exercise import or a learned model could accidentally suppress a condition that should never have depended on that context in the first place.

### Product consequence

Create two policy classes:

**Core safety paths**
- urgent low / severe low behaviour already provided by the underlying CGM alarm system;
- stale/lost glucose or sensor-health conditions where the user needs to know data is unavailable/unreliable;
- other existing core xDrip safety alarms that should not be weakened by our contextual layer.

**Contextual Attention paths**
- persistent rise/high situations;
- post-meal situations;
- “no recent insulin logged” situations;
- post-treatment re-alert timing;
- recovery/defer/acknowledgement handling;
- future personalised attention-risk signals.

The first group is not “personalised away.”

---

## 2. Suppression is a temporary hypothesis, never a permanent conclusion

A common failure pattern would be:

```text
rise detected
→ insulin logged
→ engine decides “handled”
→ suppress indefinitely
→ glucose continues worsening
```

The safer interpretation is:

```text
recent insulin = evidence that action was recorded
NOT proof that the situation is resolved
```

Likewise:

```text
No insulin needed
Waiting for recovery
Handling it
Remind me later
```

are **user-state acknowledgements**, not promises about future physiology.

### Required rule

Every context that reduces urgency must have:

- a time boundary / expiry;
- re-evaluation on every fresh glucose reading;
- an override path when trajectory worsens materially;
- a hard maximum silence ceiling appropriate to the alert class.

The exact ceiling should be chosen later through conservative product policy and replay rather than hard-coded from this research pass.

---

## 3. Fresh glucose is evidence; stale glucose is uncertainty

This is one of the strongest invariants from the earlier background/replay research.

If the last reading was rising and then readings stop, the app must **not** infer:

```text
“glucose kept rising for the next 20 minutes”
```

Nor should it infer:

```text
“the rise stopped because no new high reading arrived”
```

The only valid conclusion is:

```text
trajectory now unknown / stale
```

### Required behaviour

When glucose becomes stale:

- stop using slope/acceleration as if they were current;
- invalidate or strongly downgrade any prediction based on the old trajectory;
- preserve the last known value as historical context only;
- allow the existing stale/lost-data safety path to operate independently;
- do not resolve an Attention Episode solely because readings disappeared.

Once fresh readings resume, reconcile state before sending duplicate or stale alerts.

---

## 4. Missing insulin is unknown, not zero

The central MDI uncertainty remains:

```text
no insulin record found ≠ no insulin taken
```

The app may truthfully say:

> “I can’t see any recent insulin logged.”

It must not silently transform that into:

```text
insulin taken = 0 U
IOB = definitely 0
user forgot insulin
```

### Three separate concepts

Keep these distinct in code and decision traces:

```text
recentInsulinLogged: Bool / timestamp evidence
estimatedIOB: Double?        // derived only from recorded insulin
insulinStateConfidence: known-recorded / unknown / conflicted
```

This prevents a future learned model from treating missing treatment data as a confident negative feature.

---

## 5. Logged insulin can also be wrong

A more subtle failure is the opposite one: a treatment exists, but its amount or time is incorrect.

Example:

```text
user intended to log 1.5 U
accidentally logs 15 U
→ modeled IOB becomes very high
→ Attention Engine becomes too quiet
```

The app must assume treatment-entry mistakes are possible.

### Safety controls

- provide immediate **Undo/Edit** after quick insulin entry;
- make the entered amount clearly visible in the confirmation state;
- consider a soft confirmation for unusually large entries relative to configured/personal history, without telling the user what dose they “should” take;
- record treatment corrections for debugging/replay;
- cap how much modeled IOB can reduce Attention urgency;
- retain a maximum quiet interval even when modeled IOB is high;
- fresh worsening trajectory can re-escalate despite recorded insulin.

This protects against both true treatment insufficiency and logging errors without turning the app into a dose adviser.

---

## 6. Duplicate and edited treatments must not corrupt IOB

Potential duplicate sources include:

- local quick-entry followed by Nightscout import;
- retries after a network failure;
- app relaunch/reconciliation;
- multiple surfaces submitting the same action;
- imported treatment revisions.

Double-counted insulin is particularly dangerous because it could overstate IOB and suppress attention.

### Required behaviour

- stable UUID / external identifier for treatment-linked events;
- idempotent action processing;
- source-aware deduplication;
- a single canonical `TreatmentEntry` for standard insulin facts;
- any treatment edit/delete triggers an immediate Attention re-evaluation;
- derived IOB is recomputed from current canonical treatments rather than incrementally patched from stale state.

If deduplication is ambiguous, prefer an explicit uncertain/conflicted insulin state rather than confidently adding both records.

---

## 7. Backdated logging requires two clocks

The data model from Pass 11 deliberately preserves:

```text
occurredAt  = when the real-world action happened
recordedAt  = when the app learned about it
```

This is a safety property, not just an analytics feature.

Example:

```text
18:00 insulin actually taken
18:25 user remembers and logs it as 18:00
```

At 18:10 the live app did **not** know about the insulin. Historical replay must not give the earlier decision engine future knowledge just because the treatment was later backdated.

### Required rule

- physiology/IOB calculation uses the best known `occurredAt`;
- live decision availability is constrained by `recordedAt`;
- replay reconstructs what was knowable at each historical time.

This avoids look-ahead leakage when evaluating whether a policy would have alerted appropriately.

---

## 8. Meal AI is optional enrichment and cannot weaken core safety

Meal-photo estimation has several predictable failure modes:

- food classification wrong;
- hidden ingredients missed;
- portion size wrong;
- image incomplete;
- API unavailable;
- user accepts an inaccurate estimate;
- user never confirms the result.

### Required behaviour

The sequence remains:

```text
photo captured
→ Ate recorded immediately
→ AI proposes estimate
→ user may confirm/edit
```

An unconfirmed estimate:

- remains `modelEstimated`;
- never silently becomes confirmed carbs;
- never becomes a direct insulin-dose recommendation;
- never suppresses core glucose safety;
- should have less influence than user-confirmed meal information.

Unknown carbs remain unknown, not zero.

---

## 9. Exercise context is useful but inherently incomplete

The user will sometimes forget to log a Watch workout, which means:

```text
no HealthKit workout ≠ no exercise
```

Likewise, a manually logged exercise event may have little detail, and HealthKit synchronization may arrive late.

### Required behaviour

- manual Exercise remains first-class;
- HealthKit enriches rather than replaces manual information;
- missing HealthKit context lowers confidence rather than asserting inactivity;
- exercise may adjust urgency/confidence but must not encode a deterministic “exercise means glucose will fall” rule;
- fresh glucose evidence always outranks assumptions about expected exercise response.

If a HealthKit workout later overlaps a manual exercise event, reconcile them rather than double-counting exercise exposure.

---

## 10. `Waiting for recovery` is a state, not an exemption

This is particularly important after a low.

A rapid rise during recovery may be deliberate and appropriate contextually, so the system should avoid immediately treating it as an untreated high. But the opposite failure would be allowing `waitingForRecovery` to mute a worsening situation indefinitely.

### Required behaviour

`waitingForRecovery` should:

- reduce or reshape contextual high/rise urgency;
- expire;
- be re-evaluated with every fresh glucose reading;
- resolve when recovery stabilizes;
- be overridden if the situation materially departs from the expected recovery pattern.

The decision trace should preserve that recovery context was one reason for quieting, not claim that the later glucose outcome was caused by it.

---

## 11. Acknowledgement actions must remain reversible

Actions such as:

- `Handling it`;
- `No insulin needed`;
- `Remind 10 min`;
- `Waiting for recovery`;

are valuable because they distinguish deliberate non-action from forgetting. They are also easy to tap accidentally.

### Required behaviour

- acknowledgement changes the episode state, not the underlying glucose facts;
- every acknowledgement has bounded duration/expiry;
- fresh worsening evidence can override it;
- repeated taps are idempotent;
- accidental actions can be corrected/undone where practical;
- the user should never need to “unsnooze” a state manually for the system to regain awareness.

An acknowledgement is also a **preference/context signal**, not a physiological ground-truth label for future ML.

---

## 12. Notification delivery is itself a safety dependency

A scheduled iOS notification is not proof that the user saw or heard anything.

The FDA issued a safety communication on 5 February 2025 after receiving reports of missed smartphone alerts from diabetes devices. Reported causes included notification configuration, phone software changes, app behaviour, Focus/Do Not Disturb and connected accessories; some reports were associated with serious harm including severe hypoglycemia, severe hyperglycemia, DKA and death.

Apple exposes current notification authorization/settings through `UNUserNotificationCenter.notificationSettings()`, including alert, sound, scheduled-delivery/time-sensitive and critical-alert status. Critical Alerts require a special Apple entitlement and separate authorization; we must not assume a personal build has it.

### Product requirements

- treat notification capability as explicit app health state;
- inspect notification authorization/settings at onboarding and periodically at meaningful app lifecycle/configuration changes;
- surface a clear degraded-state warning if alerts/sounds are disabled or restricted;
- after major app/OS configuration changes, make alert-health easy to verify;
- do not mark an episode “acknowledged” because a notification was merely scheduled/delivered to iOS;
- only an explicit user action or later app interaction should count as acknowledgement;
- maintain useful in-app state even if notification delivery is unavailable.

### Important limitation

An app can inspect its configured notification settings, but it cannot prove a person actually heard or noticed a notification. The state should therefore be described as **notification capability/configuration**, not guaranteed delivery.

Sources:
- FDA, *FDA Alerts Patients to Regularly Check Diabetes-Related Smartphone Device Alert Settings, Especially Following Phone Hardware or Software Changes*, 2025-02-05.
- Apple Developer Documentation, `UNUserNotificationCenter.getNotificationSettings` / `notificationSettings()`.
- Apple Developer Documentation, Critical Alerts entitlement.

---

## 13. Network services are never in the local safety path

Nightscout is valuable for durable history, sync and replay. It must not be required to:

- log insulin;
- record `Ate`;
- record manual Exercise;
- acknowledge/defer an episode;
- calculate local context from available records;
- run core glucose safety alerts;
- resolve or re-evaluate an Attention Episode.

### Required pattern

```text
local write succeeds
→ local engine evaluates
→ remote sync queued independently
```

Nightscout outage therefore becomes a **sync-health problem**, not a reason the local Attention Engine stops working.

The same principle applies to meal-recognition APIs and any future remote model service.

---

## 14. HealthKit is optional context, not required truth

HealthKit/Watch failure modes include:

- read permission denied;
- workout not logged;
- workout sync delayed;
- workout edited/deleted;
- phone not receiving the update promptly.

The engine should represent:

```text
exercise context available
exercise context unavailable/unknown
```

rather than:

```text
workout found = true/false physiological reality
```

No core alert or local treatment workflow should depend on HealthKit availability.

---

## 15. Glucose noise and sensor-health problems must reduce confidence

Trend logic is especially vulnerable to bad samples because derivative/acceleration features amplify noise.

Failure examples:

- one spurious high value creates apparent rapid rise;
- compression low creates apparent recovery/rebound;
- sensor warm-up or unreliable period creates erratic slope;
- duplicated/out-of-order reading corrupts trend calculations.

### Required behaviour

- validate chronological ordering and deduplicate readings;
- use persistence over multiple fresh readings for important trend conclusions;
- expose a data-quality/noise state to the Attention Engine;
- low-quality data should reduce reliance on slope/acceleration and learned predictions;
- a questionable trend should not invisibly suppress existing core CGM safety visibility;
- resolution/recovery should use hysteresis/persistence rather than one favourable arrow/sample.

A future policy can distinguish “uncertain — watch/recheck” from “confidently quiet.”

---

## 16. Unit conversion must be treated as safety-critical code

xDrip supports mg/dL and mmol/L presentation. Internal Attention logic must not allow presentation units to leak into threshold/model arithmetic ambiguously.

### Required rule

Use one canonical internal glucose unit and convert only at explicit boundaries.

Tests must cover:

- mg/dL → mmol/L → mg/dL round trip;
- thresholds at both display units;
- persisted settings migration;
- model feature generation;
- Nightscout import/export;
- notification copy.

A unit bug can create an order-of-magnitude decision error, so conversion helpers belong in the invariant test suite rather than scattered UI code.

---

## 17. Time, timezone and clock changes need explicit handling

The engine mixes:

- absolute event chronology;
- elapsed durations;
- defer deadlines;
- time-of-day personal patterns.

Those are not the same concept.

### Required behaviour

- use absolute `Date`/UTC chronology for event ordering and deadlines;
- preserve timezone/UTC offset at occurrence for later circadian learning;
- do not calculate elapsed insulin/episode time from formatted local clock strings;
- reconcile pending reminders after timezone/clock changes;
- ensure daylight-saving transitions cannot duplicate or skip defer logic;
- app relaunch should recompute deadlines from persisted absolute timestamps.

---

## 18. Crash/relaunch recovery must be deterministic

A background-capable health app cannot assume an Attention Episode lives only in memory.

On relaunch:

```text
load persisted episode state
+ fetch latest canonical glucose/treatments
+ inspect pending local notifications
+ reconcile current time/defer deadline
→ run one idempotent evaluation
```

### Required behaviour

- no duplicate alerts simply because the app restarted;
- no stale notification left scheduled after the episode has resolved;
- no forgotten defer state because in-memory timers were lost;
- no double-submission of treatment actions after retry/relaunch.

This is another reason evaluation needs stable IDs and idempotent commands.

---

## 19. Learned models are advisory evidence, never the safety authority

Pass 12 recommended personal calibration, retrieval and later task-specific learned models. Their safety contract should be defined before implementation.

### Required contract

A learned signal must provide:

- model/version identifier;
- input freshness/coverage state;
- confidence/uncertainty where meaningful;
- ability to abstain;
- deterministic fallback when unavailable;
- bounded influence on Attention policy;
- no ability to suppress the Core Safety Monitor.

### Out-of-distribution / low-data situations

Examples:

- recent glucose shape unlike training history;
- missing insulin/meal context that was usually present in training;
- unusual illness/travel pattern;
- new insulin formulation/settings;
- sensor behaviour changes;
- too few similar historical episodes.

The correct behaviour is not to manufacture a confident probability. It is to **fall back toward transparent deterministic policy**.

### Promotion process

1. train/evaluate offline chronologically;
2. test important context slices rather than aggregate average only;
3. deploy in shadow mode;
4. compare candidate decisions with real outcomes/feedback;
5. enable bounded influence only after evidence of improvement;
6. preserve previous model/policy version for rollback;
7. monitor performance/drift after deployment.

Relevant FDA/IMDRF Good Machine Learning Practice principles emphasize representative data, clinically relevant testing, human-AI interaction, independent evaluation and monitoring deployed model performance. Those principles are useful engineering discipline here even while the app remains an experimental personal project.

---

## 20. Evaluate safety slices, not only averages

A model/policy can appear good overall while being poor in precisely the situations that matter.

Every replay/shadow evaluation should report separate slices for at least:

- recent recorded insulin;
- no recorded insulin;
- low/recovery context;
- recent exercise;
- meal/Ate context;
- noisy readings;
- stale-data boundaries;
- overnight;
- high-rate rise/fall;
- notification-degraded periods where known;
- treatment corrections/duplicates;
- missing optional context.

The metric question is not just:

```text
“How accurate was the model?”
```

but:

```text
“Did it delay visibility or create unnecessary interruptions in any important scenario?”
```

---

## 21. User feedback must not become physiological truth

A future model may learn from:

- `Useful`;
- `Already handled`;
- `No insulin needed`;
- `Waiting for recovery`;
- `Remind later`;
- explicit episode review.

These are valuable labels about **attention preference and user intent**.

They do not prove:

- insulin was or was not physiologically needed;
- a particular dose was correct;
- glucose would have resolved without another action;
- the same action is appropriate next time.

Keep subjective preference labels and objective glucose outcomes separate in the training dataset.

---

## 22. Avoid alert fatigue without hiding unresolved episodes

The app is specifically trying to reduce cognitive burden, so “safe” cannot mean “alert every five minutes forever.” That would quickly train the user to ignore it.

### Episode-level controls

- one unresolved situation maps to one episode, not a new independent alarm per reading;
- re-alert cadence changes with worsening/improving evidence;
- duplicate triggers collapse into the same episode;
- acknowledgement changes cadence but does not create indefinite silence;
- meaningful escalation can break through an earlier defer;
- resolution requires persistent evidence, then cancels pending contextual reminders.

This balances two failure modes:

```text
under-alerting → issue drifts unnoticed
alert flooding → user habituates/ignores app
```

Replay must score both.

---

## 23. Extreme high glucose: stay visible, do not diagnose from CGM context alone

The Attention Engine may use context to decide *when* a rising/high situation needs another prompt, but it should not infer diagnoses such as DKA from CGM trajectory alone.

For very high or otherwise concerning conditions, product copy should remain conservative and defer to established CGM/diabetes safety guidance rather than generating bespoke medical conclusions.

Likewise, if sensor data is inconsistent with how the user feels or is otherwise suspect, the app should preserve existing manufacturer/clinical safety pathways rather than treating its own prediction as superior evidence.

This research pass does not define individualized treatment instructions.

---

## 24. Failure-mode matrix

| Failure / uncertainty | Risk | Required safe behaviour |
|---|---|---|
| CGM becomes stale | Old slope treated as current | Invalidate trajectory; data state becomes unknown; independent stale-data alert path |
| Noisy/outlier CGM | False rise/fall/recovery | Persistence + data-quality gating; downgrade prediction; no one-sample resolution |
| Insulin taken but not logged | System assumes zero insulin | Missing remains unknown; neutral wording; never confident IOB=0 |
| Insulin amount/time typo | False high IOB suppresses alerts | Undo/edit, visible confirmation, bounded IOB quieting, worsening override |
| Duplicate insulin record | Double-counted IOB | Stable IDs, idempotency, source-aware dedupe, conflict state if ambiguous |
| Backdated insulin | Replay gains future knowledge | Preserve occurredAt + recordedAt; availability follows recordedAt |
| Treatment edited/deleted | Cached context becomes wrong | Recompute IOB/context and re-evaluate episode immediately |
| Meal AI wrong | False carb/context confidence | Estimate remains uncertain until confirmed; no direct dose; cannot suppress core safety |
| Meal absent/unlogged | System concludes no food | Missing meal context stays unknown |
| Workout forgotten | System concludes no exercise | Missing exercise remains unknown; manual Exercise first-class |
| Workout misclassified/late | Wrong expected trajectory | Exercise is weak context; fresh glucose outranks it; reconcile later |
| Nightscout unavailable | Logging/alerts blocked | Local-first operation; queue sync independently |
| HealthKit unavailable/denied | Engine assumes inactivity | Optional/unknown context; no core dependency |
| Notification permission/sound disabled | User never gets prompt | Inspect settings; visible degraded state; scheduled ≠ acknowledged |
| Focus/accessory/OS change affects alerts | User thinks alerts are working | Alert-health checks/guidance; never guarantee audibility |
| App crash/relaunch | Episode/defer lost or duplicated | Persist + reconcile + idempotent re-evaluation |
| Old local notification remains queued | Alert arrives after resolution | Stable notification IDs; cancel/reconcile pending requests |
| Model low confidence/OOD | False precise decision | Abstain; deterministic fallback; bounded model influence |
| Model drift/regression | Historically good policy becomes poor | Versioning, shadow evaluation, monitoring, rollback |
| Accidental acknowledgement | Long unwanted silence | Expiry, worsening override, undo/correction where practical |
| Single falling reading | Premature episode resolution | Require persistence/hysteresis across fresh readings |
| Unit conversion error | Large threshold/feature error | Canonical internal unit + strict conversion tests |
| DST/timezone/device clock change | Wrong defer/IOB timing | Absolute timestamps for chronology; timezone only as contextual feature |
| Sync/import replay | Duplicate action/notification | Stable identifiers + idempotent processing |
| Optional service exception | Engine crash | Fail closed to deterministic context; core safety remains independent |
| Conflicting context | False certainty | Bias toward visibility/recheck, not confident silence |

---

## 25. Safety invariants that should become automated tests

These are strong candidates for product-level invariant tests before feature tuning.

### Core independence

```text
Given any personalised/model output,
urgent core-low path cannot be suppressed by Attention policy.
```

```text
Given Nightscout/HealthKit/meal API unavailable,
local core safety and local logging still operate.
```

### Data uncertainty

```text
Given stale CGM,
engine never reports a fresh continued trajectory.
```

```text
Given no insulin record,
estimated IOB is unknown/not-observed rather than confidently zero.
```

```text
Given no HealthKit workout,
engine never concludes exercise definitely did not occur.
```

### Context validity

```text
Given an unconfirmed meal estimate,
it never becomes a confirmed carb treatment automatically.
```

```text
Given Handling/NoInsulinNeeded/WaitingForRecovery,
fresh worsening evidence can re-escalate the episode.
```

```text
Given recent recorded insulin,
contextual quieting still has a finite safety ceiling.
```

### Idempotency

```text
Given the same action/event twice,
only one treatment/event effect is applied.
```

```text
Given app relaunch with the same episode state,
reconciliation does not duplicate notifications.
```

### Model fallback

```text
Given model unavailable/low-confidence/out-of-distribution,
engine produces a deterministic policy decision without crashing.
```

### Resolution

```text
Given one transient improving sample,
an unresolved episode does not resolve unless persistence/hysteresis requirements are met.
```

---

## 26. Decision reason codes are a safety/debugging feature

Every meaningful Attention decision should keep structured reason codes, for example:

```text
Decision: REMIND
Reasons:
- persistentRise
- mealRecent
- noRecentRecordedInsulin
- noRecoveryContext
Confidence modifiers:
- exerciseUnknown
- glucoseFresh
Policy: attention-v1.3
```

Or:

```text
Decision: QUIET_FOR_NOW
Reasons:
- recordedInsulinRecent
- modeledInsulinActivityMeaningful
- trajectoryDecelerating
Safety bounds:
- reevaluateEveryFreshReading
- maxQuietDeadline=<timestamp>
```

This is valuable for:

- debugging unexpected alerts;
- replay comparison;
- user trust/explainability;
- finding model regressions;
- understanding why a learned signal changed behaviour.

Avoid persisting opaque “risk score = 0.73” without the inputs/model version/state needed to interpret it.

---

## 27. What to reuse / adapt / build / ignore

### Reuse

- xDrip's existing core glucose alarm plumbing rather than replacing it;
- canonical glucose/treatment stores;
- notification scheduling/cancellation infrastructure;
- existing stale/data-health/noise handling where appropriate after V7 implementation review;
- Nightscout sync as optional remote durability rather than local safety state.

### Adapt

- current snooze/alert infrastructure so Attention suppression is episode-scoped and bounded;
- notification settings checks into an explicit alert-health status;
- treatment quick entry with undo/correction and stable IDs;
- replay harness to include safety slices and invariant assertions.

### Build

- Core Safety Monitor / safety-island boundary;
- explicit `DataQualityState` / freshness inputs;
- `NotificationCapabilityState`;
- bounded acknowledgement/defer semantics;
- idempotent episode reconciliation after relaunch;
- reason-coded decisions;
- model abstention/fallback contract;
- safety invariant tests and synthetic failure fixtures.

### Ignore / defer

- direct insulin-dose recommendation;
- DKA diagnosis from CGM/context alone;
- exact FMEA risk-number scoring without evidence;
- unrestricted adaptive model self-modification;
- dependency on Critical Alerts entitlement for baseline correctness;
- remote/cloud services in the core safety path.

---

## 28. Product-spec implications

The eventual product specification should include explicit requirements that:

1. **Core Safety Monitor is architecturally independent** of contextual personalisation.
2. Attention quieting is always bounded, expires and is re-evaluated on fresh glucose.
3. Fresh worsening evidence can override mitigating context and acknowledgements.
4. Stale glucose invalidates current trajectory inference.
5. Missing insulin/meal/exercise data remains unknown rather than becoming zero/false.
6. Modeled IOB is derived from recorded insulin only and has bounded suppressive influence.
7. Treatment correction/duplication is handled safely and causes re-evaluation.
8. Manual Exercise and HealthKit context coexist without assuming absence means none.
9. Meal AI remains optional estimated context until confirmed.
10. Core operation is local-first and independent of Nightscout/HealthKit/remote APIs.
11. Notification authorization/settings become a visible system-health state.
12. Scheduled notification is not treated as user acknowledgement.
13. App restart/reconciliation is idempotent.
14. One canonical glucose unit is used internally with strict conversion tests.
15. Contextual episode resolution uses persistence/hysteresis, not one reading.
16. Learned models can abstain, have deterministic fallback and cannot suppress core safety.
17. Models/policies are versioned, shadow-tested, evaluated chronologically and rollbackable.
18. Replay reports high-risk/context slices, not average metrics alone.
19. Decision-changing outputs retain structured reason codes.
20. Exact safety ceilings/thresholds are configuration/policy values selected conservatively and validated, not hidden constants scattered through UI code.

---

## 29. Sources consulted

Primary/current sources used for this pass include:

- U.S. FDA, **FDA Alerts Patients to Regularly Check Diabetes-Related Smartphone Device Alert Settings, Especially Following Phone Hardware or Software Changes: FDA Safety Communication**, issued 2025-02-05.  
  https://www.fda.gov/medical-devices/safety-communications/fda-alerts-patients-regularly-check-diabetes-related-smartphone-device-alert-settings-especially

- U.S. FDA, **Good Machine Learning Practice for Medical Device Development: Guiding Principles**, reflecting the final IMDRF principles released in January 2025.  
  https://www.fda.gov/medical-devices/software-medical-device-samd/good-machine-learning-practice-medical-device-development-guiding-principles

- U.S. FDA / Health Canada / MHRA, **Transparency for Machine Learning-Enabled Medical Devices: Guiding Principles**.  
  https://www.fda.gov/medical-devices/software-medical-device-samd/transparency-machine-learning-enabled-medical-devices-guiding-principles

- Apple Developer Documentation, **UNUserNotificationCenter.getNotificationSettings / notificationSettings()**.  
  https://developer.apple.com/documentation/usernotifications/unusernotificationcenter/getnotificationsettings(completionhandler:)

- Apple Developer Documentation, **UNNotificationSettings** and **Critical Alerts entitlement**.  
  https://developer.apple.com/documentation/usernotifications/unnotificationsettings  
  https://developer.apple.com/documentation/bundleresources/entitlements/com.apple.developer.usernotifications.critical-alerts

- Prior project research passes 01–12, especially iOS background constraints, historical replay, Nightscout storage, IOB/COB, Apple Health context, event-ledger design and prediction/personalisation.

---

## Bottom line

The safest architecture is not “make the Attention Engine smart enough to understand everything.” It is:

> **Keep a small core that remains reliably conservative, then let smarter contextual layers earn the right to reduce unnecessary attention within explicit bounds.**

That preserves the product's real goal — less diabetes brain-space — without making silence depend on fragile assumptions.