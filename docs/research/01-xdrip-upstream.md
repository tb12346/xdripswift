# 01 — xDrip upstream PR + issue archaeology

Status: **Complete**  
Research date: 2026-08-16

## Question

What has already been proposed, built, rejected or discussed in upstream xDrip4iOS around alarms, snoozing, treatments, quick actions, notifications, prediction and related interaction patterns — and what should this project reuse, adapt, avoid or build differently?

## Executive conclusion

Upstream xDrip already contains many of the **primitives** needed by this project: glucose alerts, multiple urgency thresholds, pre-snooze/unsnooze, short auto-snooze/cooldown behaviour, configurable fast-rise/drop gating, treatment storage, Nightscout treatment sync, Home Screen quick actions, Siri/App Intent precedent, rich notifications, widgets, Watch support, and access to AID IOB/COB data in some configurations.

What it does **not** appear to have is a persistent model of an unresolved diabetes situation or episode. The existing alert system is mainly driven by thresholds, alarm configuration and snooze state. It generally does not know that the user ate, took an action, deliberately decided not to act yet, is waiting for recovery, or has already handled the situation.

That distinction is the strongest result of this pass. Our proposed **Attention Engine** should not replace xDrip’s alert plumbing. It should sit above/beside it as a context/episode layer that decides whether the existing notification machinery should remain quiet, remind, escalate, or resolve.

Several upstream requests independently validate the user problem we are targeting:

- users ask not to be alerted when glucose is already moving in the desired direction;
- users ask for alarms to apply only in contextually relevant glucose ranges;
- users ask for conditional escalation while otherwise snoozed;
- users ask for a glucose-condition reminder instead of a fixed timer;
- users ask for Shortcuts/Siri treatment entry to avoid opening the app.

## 1. Alerts and snoozing: strong plumbing, weak episode context

### Issue #69 — conditional snoozing was requested very early

[Issue #69: Snoozing and Alert sounds](https://github.com/JohanDegraeve/xdripswift/issues/69) requested:

- pre-emptive snoozing;
- seeing and cancelling active snoozes;
- keeping a snooze active across temporary glucose fluctuations;
- **conditional snoozing**, e.g. snooze a high for an hour but alert if glucose becomes much higher.

Pre-snooze and unsnooze were subsequently implemented. The proposed solution for conditional escalation was to use a separate Very High / Very Low alarm rather than create a contextual snooze rule.

**Implication:** xDrip has long supported multiple static urgency thresholds, but the conceptual model remains “alarm A is snoozed; alarm B may still fire,” rather than “this episode is being handled, unless its risk materially changes.”

### Issue #191 — automatic short alert suppression already has precedent

[Issue #191](https://github.com/JohanDegraeve/xdripswift/issues/191) described high/low alarms repeatedly firing within a few minutes, especially with one-minute Libre readings, and proposed an automatic five-minute snooze/cooldown. The issue was completed.

**Implication:** automatic re-alert suppression is not foreign to xDrip. Our dynamic re-alerting can reuse the same general notification/snooze machinery while changing the policy that chooses the interval.

### Issue #186 + PR #676 — upstream accepted contextual gating, but made it manually configurable

[Issue #186: Target limitations for alarms](https://github.com/JohanDegraeve/xdripswift/issues/186) starts from a very familiar problem: when glucose is very high and dropping quickly, a Fast Drop alert can be unwanted because the movement is desirable. The requester wanted Fast Drop to become relevant again only nearer the target range.

The discussion exposed an important difficulty: different users wanted different boundaries and opposite behaviours. Paul Plant ultimately chose simple explicit settings — **Only When Under** / **Only When Over** — rather than deriving behaviour from other alarm thresholds or hidden logic.

That work shipped in 6.2.0 and corresponds to merged [PR #676](https://github.com/JohanDegraeve/xdripswift/pull/676), which also added per-alarm enable/disable and improved the Snooze view.

**Implication:** upstream is willing to add context to alarm evaluation, but prioritises predictability and explainability. Our Attention Engine should preserve that principle. Contextual decisions need to be inspectable and deterministic enough that the user can understand why an alert was or was not generated.

### Issue #694 — the exact “don’t nag me while this is resolving” problem remains open

[Issue #694: No alerting, when trend goes into right direction](https://github.com/JohanDegraeve/xdripswift/issues/694), opened in December 2025, asks why a high alarm reappears when its snooze expires even though glucose is now falling. The requested feature is essentially “disable the alarm when the trend is going in the right direction.”

Paul Plant responded that the idea makes sense and would be considered. Another participant raised a legitimate concern: trend arrows can be noisy or disagree with actual recent movement, so making alarm behaviour depend on a single trend indicator could become unpredictable.

**Implication:** this is extremely close to our core problem, and it also gives us a safety/design requirement. We should not suppress or escalate based on one trend arrow. The engine should assess a short sequence of readings, persistence, data quality/noise and — eventually — logged action/IOB.

### Issue #480 — condition-based attention is more useful than a timer

[Issue #480: alarm/BG level reminder](https://github.com/JohanDegraeve/xdripswift/issues/480) asks for a temporary reminder triggered when glucose reaches a chosen value after corrective action, rather than a timer. The requester explicitly explains that the problem with timers is not friction: physiological response time varies, so multiple timers can still miss the useful moment.

**Implication:** “remind me when the situation changes” should be a first-class Attention Engine capability. This is a better abstraction than simply offering more snooze durations.

### Issue #695 — demand exists for genuinely multi-signal alerts

[Issue #695](https://github.com/JohanDegraeve/xdripswift/issues/695) proposes an alarm that combines AID “carbs required” information, amount, time horizon and current glucose threshold.

**Implication:** upstream users are already asking for alarms based on combined context rather than a single glucose threshold. We should build a general context-decision layer rather than adding a new special-case alarm for every combination.

## 2. Notification infrastructure is reusable

Merged [PR #551](https://github.com/JohanDegraeve/xdripswift/pull/551) added:

- richer iOS and Watch notifications with glucose charts;
- a notification content extension;
- Snooze All;
- visible snooze status in the Snooze view and Home Screen.

The current architecture therefore already has mature notification delivery and snooze state that the Attention Engine can feed into.

**Recommendation:** **reuse/adapt** the existing notification infrastructure. Do not create a parallel alert delivery system unless research pass 05 uncovers a hard iOS limitation.

## 3. Treatments: the data model is already there; low-friction entry is the gap

Historical [PR #296](https://github.com/JohanDegraeve/xdripswift/pull/296) introduced the treatment model and UI for carbs, insulin and exercise using Core Data and Nightscout integration. Later [PR #336](https://github.com/JohanDegraeve/xdripswift/pull/336) integrated treatment markers into the glucose chart. Treatment storage/sync has since continued to evolve, and V7 rebuilds the treatment UI in SwiftUI.

For this project, that means we should **not invent a second insulin/carb treatment database** unless there is a very specific reason. Our new lightweight events should either use existing treatment entities where semantically correct or link clearly to them.

### Issue #569 — upstream users explicitly want treatment entry through Shortcuts/Siri

[Issue #569: Shortcuts support for treatments](https://github.com/JohanDegraeve/xdripswift/issues/569) asks for shortcut actions to register treatments because it would “greatly improve the UX of adding treatments.” The requester reframed it as adding treatments via Siri. Paul Plant replied that it is technically possible and added it to the list, but it has not been prioritised.

**Implication:** our low-friction insulin logging is not speculative. It is an existing upstream UX gap with an acknowledged technical path.

**Recommendation:** **build/adapt** App Intent based treatment entry in research/build pass 04 rather than creating a custom shortcut mechanism.

## 4. Quick Actions and App Intents already have precedent

Merged [PR #346](https://github.com/JohanDegraeve/xdripswift/pull/346) added a Home Screen icon Quick Action via a `QuickActionsManager` for toggling Speak Readings.

[PR #492](https://github.com/JohanDegraeve/xdripswift/pull/492) proposed a `GlucoseIntent` using Apple App Intents and a SwiftUI response. That PR itself was closed rather than merged, but Siri glucose functionality later appeared in xDrip release work. Its discussion also covers `AppShortcutsProvider`, Spotlight exposure and intent donation.

**Implication:** the project already has both old-style Home Screen Quick Action and modern App Intent precedent. “Ate,” “Log insulin,” and acknowledgement actions should be feasible to expose through system surfaces; the exact best combination belongs in research pass 04.

## 5. Prediction / IOB / COB: useful prior art, but upstream provides a warning about scope and validation

### PR #366 — local IOB exists as old upstream prior art

[PR #366: Implemented Insulin on Board](https://github.com/JohanDegraeve/xdripswift/pull/366) is an old, still-open implementation of local IOB using logged insulin with configurable duration and peak, based on OpenAPS documentation.

**Recommendation:** **reference, do not merge wholesale**. It proves that local MDI-oriented IOB fits the existing treatment architecture, but the implementation predates major changes in the codebase.

### PR #633 — broad prediction + IOB/COB experiment was rejected for good reasons

[PR #633](https://github.com/JohanDegraeve/xdripswift/pull/633) proposed a large prediction system with multiple regression models plus IOB/COB calculations. The author explicitly said the feature had been generated with Claude Code and that they had not personally reviewed the Swift implementation in detail.

Johan Degraeve’s response is particularly relevant to how we should build this project. The concerns were:

- the change set was very large;
- AI-generated code still required detailed review and testing;
- impact on existing functionality was unclear;
- some claimed problems were not established upstream problems;
- the validity/currentness of the prediction algorithms was uncertain;
- unnecessary background modes had been added;
- the product value was not clearly tied to a known missing capability.

The author then noted that iAPS was not a good fit for their MDI use case and continued experimenting on their fork.

**Implication:** this is a strong argument for our proposed development method, not against the project. We should do the opposite of PR #633:

- start from a precise user problem;
- keep domain logic modular and changes narrow;
- separate prediction from alert/action policy;
- backtest against real history;
- avoid dosage recommendations in the early system;
- review and test every generated change rather than treating a working demo as validation.

## 6. What upstream appears to be missing

This archaeology did **not** find a mature upstream implementation of the following concepts:

- a persistent “meal happened but decision/action is unresolved” state;
- an episode that can be acknowledged as “I’m handling this”;
- explicit “no insulin needed” / “waiting for recovery” states;
- alerts that automatically de-escalate because a relevant treatment/action was logged;
- escalating reminders based on the same unresolved episode;
- dynamically chosen re-alert intervals based on worsening/resolving context;
- an “Ate” event independent of precise carb counting;
- a general condition-based reminder engine (“remind me when X changes”) rather than fixed-time snooze;
- a unified attention score/state combining glucose trajectory, meals, treatments and acknowledgements.

These should therefore be treated as **our domain layer**, not as another set of xDrip alarm settings.

## Reuse / adapt / build / ignore

| Area | Recommendation | Why |
|---|---|---|
| CGM ingestion / glucose storage | **Reuse** | Mature core; keep untouched wherever possible. |
| Existing alert delivery / sounds / notification categories | **Reuse** | Already battle-tested and supports rich notifications. |
| Snooze infrastructure | **Adapt** | Useful mechanism, but policy should become context-aware. |
| Fast-rise/drop threshold gating | **Reuse as input** | Helpful signal, but insufficient as the Attention Engine itself. |
| Treatment Core Data + Nightscout sync | **Reuse** | Avoid parallel insulin/carb records. |
| Home Screen Quick Actions | **Adapt** | Existing precedent for low-friction entry. |
| App Intents / Siri | **Adapt/build** | Best candidate for Ate/insulin/acknowledgement system actions. |
| Old local IOB PR #366 | **Reference/rebuild** | Concept useful; code too old to adopt directly. |
| PR #633 prediction system | **Do not adopt wholesale** | Scope/validation concerns; mine ideas only. |
| Separate special-case alarm for every context combination | **Avoid** | Leads to configuration complexity; build general context/episode logic instead. |
| Single trend-arrow suppression | **Avoid** | Too noisy/unpredictable for a safety-relevant decision. |

## Product-spec implications

1. **Attention Episode should be a first-class domain object/state**, distinct from xDrip’s existing `AlertKind` and snooze state.
2. The Attention Engine should **consume existing glucose/treatment data and output an attention decision**; existing notification infrastructure should deliver it.
3. “Snooze” should evolve conceptually into both:
   - time-based deferment when appropriate; and
   - **condition-based deferment**, where the engine reassesses on each new reading and can resolve or escalate earlier.
4. Alert suppression should require evidence of resolution (multiple readings/persistence/context), not a single trend arrow.
5. Low-friction logging should be designed as a system capability, probably backed by App Intents, rather than an app-screen-only feature.
6. Existing treatment records should remain the source of truth for logged insulin/carbs. New project-specific events such as `Ate`, acknowledgement and deliberate waiting may need a lightweight adjacent event model.
7. Local IOB is likely valuable for MDI because it lets the engine distinguish “rising with no known active insulin” from “rising while logged insulin is still active.” It should be researched separately in pass 07.
8. Any prediction/personalisation work should be **downstream** of a validated attention-policy layer and historical replay/backtesting.

## Open questions handed to later research passes

- **Pass 02 — V7:** where should the Attention Engine attach to the new SwiftUI architecture, and how stable are the relevant alert/treatment APIs?
- **Pass 03 — other diabetes apps:** do Loop/Trio/iAPS/AAPS already model “user has acted / treatment in progress / expected response” in a way we can borrow?
- **Pass 04 — low-friction logging:** which App Intent, widget, Lock Screen, notification and Watch surfaces can write treatments/events safely with the fewest taps?
- **Pass 05 — iOS constraints:** what can actually run/re-evaluate reliably in the background after a notification or new glucose reading?
- **Pass 06 — backtesting:** how should an Attention Episode be replayed across historical CGM + treatment data?
- **Pass 07 — IOB/COB:** what modern local calculation is appropriate for manually logged injections?

## Key upstream references

- [Issue #69 — Snoozing and Alert sounds](https://github.com/JohanDegraeve/xdripswift/issues/69)
- [Issue #191 — automatic five-minute alert snooze](https://github.com/JohanDegraeve/xdripswift/issues/191)
- [Issue #186 — target limitations for fast-rise/drop alarms](https://github.com/JohanDegraeve/xdripswift/issues/186)
- [PR #676 — alarm changes, including Only When Over/Under](https://github.com/JohanDegraeve/xdripswift/pull/676)
- [Issue #694 — suppress alerts when trend is going the right way](https://github.com/JohanDegraeve/xdripswift/issues/694)
- [Issue #480 — BG-condition reminder instead of timer](https://github.com/JohanDegraeve/xdripswift/issues/480)
- [Issue #695 — multi-signal carbs-required alarm](https://github.com/JohanDegraeve/xdripswift/issues/695)
- [PR #551 — richer notifications and Snooze All](https://github.com/JohanDegraeve/xdripswift/pull/551)
- [Issue #569 — Shortcuts/Siri treatment entry](https://github.com/JohanDegraeve/xdripswift/issues/569)
- [PR #346 — Home Screen Quick Actions](https://github.com/JohanDegraeve/xdripswift/pull/346)
- [PR #492 — App Intent / Siri glucose experiment](https://github.com/JohanDegraeve/xdripswift/pull/492)
- [PR #366 — local Insulin on Board](https://github.com/JohanDegraeve/xdripswift/pull/366)
- [PR #633 — prediction + IOB/COB experiment](https://github.com/JohanDegraeve/xdripswift/pull/633)
- [PR #296 — treatment persistence/Nightscout foundation](https://github.com/JohanDegraeve/xdripswift/pull/296)
- [PR #336 — treatments on glucose chart](https://github.com/JohanDegraeve/xdripswift/pull/336)
