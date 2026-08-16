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
| 06 | Historical replay + backtesting | **Next** | How can we tune alert logic against real historical CGM/treatment data? |
| 07 | Insulin-on-board + carbs-on-board | Not started | How early should approximate IOB/COB become part of context? |
| 08 | Meal photo recognition + carb estimation | Not started | Open source, API, or multimodal model for a personal prototype? |
| 09 | Apple Health + Watch context | Not started | Which exercise, sleep, HR and activity signals are useful and accessible? |
| 10 | Future personalised data model | Not started | What should we start recording from day one to support later learning? |
| 11 | Prediction + personalisation approaches | Not started | What existing forecasting approaches are worth adapting later? |
| 12 | Safety + failure modes | Not started | Where must the system become conservative or avoid false certainty? |
| 13 | Personal-use deployment | Not started | How do we make builds easy to install, update and run alongside Zukka? |
| 14 | Licensing + future distribution boundary | Not started | What changes if this moves beyond private personal use? |

## Findings worth carrying forward

### 01 — xDrip upstream archaeology

xDrip already has most of the delivery and data primitives we need: mature alarms and snoozing, contextual Fast Rise/Drop gates, rich notifications, treatment storage/sync, Home Screen Quick Actions and App Intent/Siri precedent. The important missing layer is **persistent episode context** — knowing that the user ate, acted, deliberately deferred action, or is already handling a situation.

The Attention Engine should sit above/beside xDrip's existing alert machinery rather than replacing it. It should evaluate an Attention Episode and decide whether to remain quiet, remind, escalate or resolve, while reusing existing notification infrastructure.

### 02 — V7 readiness

V7 should be the **preferred technical base after a short build/device qualification gate**. The key reason is architectural: `RootApplicationCoordinator` separates long-lived services from SwiftUI presentation, which gives the Attention Engine a much cleaner integration seam than the 6.x `RootViewController` architecture.

The domain engine should still remain pure/testable behind protocols so that backend work can survive V7 churn or temporarily run in 6.x if qualification fails. Avoid investing in substantial 6.x UIKit/storyboard UI.

### 03 — other diabetes apps

The core product idea is differentiated, but many pieces already exist separately:

- xDrip+ validates direction-aware smart snoozing and pre-emptive “already treated” suppression;
- Loop validates separating meal state from insulin state and includes missed-meal detection;
- Trio/iAPS validate unannounced-meal logic and treatment-state reasoning;
- Trio explicitly supports external insulin entering IOB, highly relevant for MDI;
- AAPS validates multi-signal context, short re-alert cycles and data-quality gating.

The opportunity is not to invent new diabetes physiology, but to combine proven contextual signals into a simpler MDI-focused system whose job is **“does this need my attention?”** rather than automated dosing.

### 04 — low-friction logging

The strongest interaction architecture is **one shared domain-action layer with multiple system adapters**.

- Proactive actions such as `Ate` and `Log insulin` should be App Intents.
- Reactive actions should primarily live on the Attention notification itself.
- Notification text-input actions can collect insulin amounts without opening the app and can also work via Apple Watch dictation.
- iOS 18+ WidgetKit Controls are a better locked-phone one-tap surface for `Ate` than ordinary interactive widgets.
- Ordinary widget/Live Activity buttons require authentication/unlock and cannot resolve missing parameters at tap time, making them weaker for arbitrary insulin entry.
- V7 already has the core seams: App Intents, central notification handling, AlertManager actions, Watch notification UI and WidgetKit.

A useful product challenge emerged: `Handling it` may be too vague to expose everywhere once more specific states exist (`insulin logged`, `no insulin needed`, `waiting for recovery`, `remind me`). Fewer, more meaningful actions should reduce cognitive load.

### 05 — iOS background constraints

The Attention Engine is technically viable on iOS **if it is event-driven rather than timer-driven**.

- Fresh glucose should be the primary physiological re-evaluation clock.
- Treatment logs, notification actions and app startup/reconciliation are additional triggers.
- A time-based defer should persist its deadline and schedule an OS-owned local notification as a fallback, while every fresh glucose reading can resolve, retain or escalate the episode before that deadline.
- Background refresh is discretionary and must not sit on the correctness path for real-time attention logic.
- Direct Bluetooth CGM mode is promising because V7 declares `bluetooth-central`; follower modes have separate delivery characteristics and need independent qualification.
- Stale/missing glucose is its own state and must never be interpreted as evidence that a prior rise/fall simply continued.
- Attention Episode state must be persisted and evaluation must be idempotent across suspension, relaunch and duplicate callbacks.

The key operating principle is: **react to fresh evidence, persist context, schedule conservative time guards, and never rely on periodic background execution.**

## Working product hypothesis

The central problem is not simply calculating insulin. It is deciding **when diabetes genuinely needs the user's attention** and handling the rest with as little cognitive overhead as possible.

A future Attention Engine should be able to consider signals such as:

- current glucose
- rate and persistence of rise/fall
- acceleration/deceleration
- recent meal / Ate event
- recent insulin and approximate IOB
- low/recovery state
- exercise/activity context
- prior alerts and acknowledgements
- whether the user has explicitly said they are handling the situation

and translate them into a simple attention decision rather than exposing every raw signal.
