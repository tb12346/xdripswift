# Diabetes App Research

This folder is the durable research record for the personal diabetes attention-management project built on xDrip4iOS/xdripswift.

## Purpose

The research phase should answer the major product and technical questions before we write the product specification or begin substantial feature development.

For every research pass, capture:

1. **What we found**
2. **What matters for this project**
3. **Reuse / adapt / build / ignore**
4. **Open questions**
5. **Product-spec implications**

## Research principles

- Treat this as research, not implementation.
- Prefer proven patterns and existing code over rebuilding from scratch.
- Keep the core problem in view: reduce the amount of conscious attention diabetes requires while improving glucose control.
- Avoid premature work on polished UI, branding, direct insulin-dose recommendations, or sophisticated ML until the attention-management concept is validated.
- Keep experimental decision support separate from clinically validated dosing advice.

## Research backlog

| # | Research area | Status | Main decision |
|---|---|---|---|
| 01 | [xDrip upstream PR + issue archaeology](./01-xdrip-upstream.md) | **Complete** | What existing alert, snooze, logging, widget and Watch work can we reuse? |
| 02 | [V7 readiness + migration path](./02-v7-readiness.md) | **Complete** | Stable 6.x, V7, or a deliberately portable hybrid? |
| 03 | [Other diabetes apps: attention + alert patterns](./03-other-diabetes-apps.md) | **Complete** | What should we borrow from Loop, Trio/iAPS, AAPS, xDrip+ and related tools? |
| 04 | [Low-friction meal/treatment logging on iOS](./04-low-friction-logging.md) | **Complete** | What is the fastest reliable way to log Ate, insulin and acknowledgements? |
| 05 | [iOS notification + background constraints](./05-ios-background-constraints.md) | **Complete** | What attention-engine behaviours are actually possible on iPhone and Watch? |
| 06 | [Historical replay + backtesting](./06-historical-replay.md) | **Complete** | How can we tune alert logic against real historical CGM/treatment data? |
| 07 | [Nightscout API + data storage](./07-nightscout-api-data.md) | **Complete** | What can Nightscout store, and which data should remain local? |
| 08 | [Insulin-on-board + carbs-on-board](./08-iob-cob.md) | **Complete** | How early should approximate IOB/COB become part of context? |
| 09 | [Meal photo recognition + carb estimation](./09-meal-photo-carb-estimation.md) | **Complete** | Open source, API, or multimodal model for a personal prototype? |
| 10 | [Apple Health + Watch context](./10-apple-health-watch-context.md) | **Complete** | Which exercise, sleep, HR and activity signals are useful and accessible? |
| 11 | [Future personalised data model](./11-personalised-data-model.md) | **Complete** | What should we start recording from day one to support later learning? |
| 12 | [Prediction + personalisation approaches](./12-prediction-personalisation.md) | **Complete** | What existing forecasting and learning approaches are worth adapting, and what should personalisation actually predict? |
| 13 | [Safety + failure modes](./13-safety-failure-modes.md) | **Complete** | Where must the system become conservative or avoid false certainty? |
| 14 | Personal-use deployment | **Next** | How do we make builds easy to install, update and run alongside Zukka? |
| 15 | Licensing + future distribution boundary | Not started | What changes if this moves beyond private personal use? |

## Findings worth carrying forward

### 01 — xDrip upstream archaeology

xDrip already provides most of the delivery and data primitives: mature alarms/snoozing, treatment storage and sync, notification infrastructure, quick actions, App Intent/Siri precedent, widgets and Watch support. The missing layer is **persistent episode context** — knowing the user ate, acted, deliberately deferred action, or is already handling a situation. The Attention Engine should sit above/beside the existing alert machinery rather than replacing it.

### 02 — V7 readiness

V7 is the preferred base after a short real-device qualification gate. `RootApplicationCoordinator` gives a much cleaner service-integration seam than 6.x UIKit. Keep the Attention Engine pure/testable behind protocols so backend work survives V7 churn; avoid meaningful new 6.x UI investment.

### 03 — other diabetes apps

The product idea is differentiated, but the physiology and mechanics are not novel: xDrip+ validates smart/direction-aware snoozing, Loop validates meal state independent of insulin, Trio/iAPS validate unannounced-meal and recovery contexts, and AAPS validates multi-signal contextual logic. Borrow those internals while keeping the user-facing question simple: **does this situation need attention now?**

### 04 — low-friction logging

Use one shared domain-action layer behind notification actions, App Intents, Controls, widgets, Watch and in-app UI. Notification actions are strongest for reactive actions; App Intents/Controls are strongest for proactive `Ate`. Arbitrary insulin entry is better via text-input notification/Siri than fixed-dose buttons. Keep actions structured and record source/provenance.

### 05 — iOS background constraints

The Attention Engine is viable only as an **event-driven** system. Fresh glucose is the primary physiological clock; treatments, user actions and app reconciliation are additional triggers. Time-based defer should schedule an OS-owned local notification fallback while fresh readings can cancel/escalate earlier. Never depend on periodic background timers or treat stale glucose as continued trend.

### 06 — historical replay + backtesting

Replay complete Attention Episodes, not just glucose predictions. Use the same pure Swift engine live and historically with a fake clock. Score interruptions/day, repeats/episode, first-alert timing, post-treatment nags, silent recoveries and stale-data behavior. Keep raw health history out of Git; store only aggregate results and synthetic/redacted fixtures. Replay selects policies but cannot prove counterfactual TIR improvement.

### 07 — Nightscout API + data storage

Nightscout should be the durable interoperability/history layer for standard diabetes facts, while Attention-specific state remains local-first. Glucose/insulin/carbs use existing xDrip + Nightscout paths; `Ate` without carbs, acknowledgements, defer/recovery states and episode lifecycle belong in local Attention data. Use least-privilege Nightscout tokens and keep raw exports/credentials out of Git.

### 08 — IOB + COB

Approximate local **IOB should enter early; classical COB should wait**. Model recorded rapid insulin using a proven action curve and expose both remaining insulin and activity/phase internally. Use IOB to modify attention urgency, never to recommend dose. Missing logs remain unknown. Keep `Ate` useful without forcing carb precision; dynamic COB comes later only if carb estimation and replay prove value.

### 09 — meal photo + carb estimation

Meal photos should be a capture-and-confirm aid. Record `Ate` immediately, then let a pluggable vision provider propose food components, portion assumptions and an **editable carb range with uncertainty**. Confirmed carbs can become normal TreatmentEntry/Nightscout data; unconfirmed AI output remains estimated context. Start with multimodal/specialist APIs and nutrition grounding rather than training a custom model.

### 10 — Apple Health + Watch context

Use Apple Health narrowly at first: **recent completed workouts are the valuable signal**. Read workout type, timing, duration and optionally active energy plus workout-associated average/max HR, normalize to a small `RecentExerciseContext`, and feed that to replay/Attention logic as a confidence/urgency modifier. Do not treat all exercise as glucose-lowering; aerobic and high-intensity/resistance activity can differ materially. Existing V7 HealthKit code only exports glucose, so add a separate read-side `HealthContextProvider`. HealthKit background delivery can help after workouts are saved but is not a guaranteed live Watch feed. Keep manual Exercise first-class because Watch workouts will not always be logged. HealthKit glucose read-back is unnecessary in the intended architecture because xDrip/Zukka is already the source writing those readings to HealthKit.

### 11 — future personalised data model

Build personalization on an **honest event history rather than permanent derived features**. Keep xDrip `BgReading` and `TreatmentEntry` canonical; add a separate local `AttentionStore` for Ate, manual exercise, acknowledgements, defer/recovery actions, meal estimates/corrections, episode state and decision-changing events. Every new event should preserve `occurredAt` and `recordedAt`, structured provenance/source, certainty, linkage and revision information. Manual exercise remains first-class and can later be enriched/reconciled with overlapping HealthKit workouts rather than replaced. Keep original AI estimates when users correct them, version policies/models/configuration, retain timezone-at-occurrence for circadian learning, and recompute features such as IOB/exercise recency under future models. Avoid duplicated HealthKit glucose, raw Watch sensor archives and “collect everything for ML.” The principle is: **store facts, uncertainty and provenance; derive intelligence later.**

### 12 — prediction + personalisation

**Personalise the attention decision before trying to perfectly predict glucose.** Keep the deterministic Attention Engine as the decision authority, then layer in transparent personal calibration and similar historical episodes before training a learned model. Similar-episode retrieval is especially attractive for a one-person system because it is interpretable and improves naturally as history grows. When labels are sufficient, start with a small task-specific tabular model for outcomes such as unresolved rise or likely recovery; only then test a short-horizon glucose forecaster as an additional signal. Current 2026 benchmarking reinforces that foundation models can be strong with little personal data, but a lightweight supervised LSTM can still win when enough task-specific data exists, so do not assume a larger Transformer/foundation model is better. Evaluate chronologically and by clinically/product-relevant slices, use uncertainty explicitly, and promote learned models only through replay and shadow evaluation. GluPredKit is a useful MIT-licensed offline research harness with Nightscout parsing and multiple forecasting baselines, but it should stay outside the iOS runtime. Never use observational prediction to infer a safe insulin dose or causal response to a hypothetical dose.

### 13 — safety + failure modes

**Keep core glucose safety independent from contextual personalisation.** A small deterministic Core Safety Monitor should preserve urgent/core glucose and data-health alert paths even if the Attention Engine, Nightscout, HealthKit, meal AI or learned models fail. Contextual quieting is always bounded and reversible: every acknowledgement/defer expires, every fresh glucose re-evaluates the episode, and worsening evidence can override recent insulin, `Handling it`, `No insulin needed` or `Waiting for recovery`. Stale/missing/conflicting data reduces confidence rather than manufacturing certainty. Treat notification capability as explicit system health because a scheduled iOS notification is not proof the user heard or saw it. Learned models may abstain and fall back to deterministic policy; they never suppress the core safety island. Exact safety ceilings and thresholds belong in the product-spec/replay stage, backed by invariant tests rather than scattered constants.

## Working product hypothesis

The central problem is not simply calculating insulin. It is deciding **when diabetes genuinely needs the user's attention** and handling the rest with as little cognitive overhead as possible.

A future Attention Engine should be able to consider signals such as:

- current glucose
- rate and persistence of rise/fall
- acceleration/deceleration
- recent meal / Ate event
- recent insulin and approximate IOB
- low/recovery state
- recent exercise context
- prior alerts and acknowledgements
- whether the user has explicitly said they are handling the situation
- later: personal historical calibration, similar-episode evidence and learned risk signals

and translate them into a simple attention decision rather than exposing every raw signal.
