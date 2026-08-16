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
| 01 | xDrip upstream PR + issue archaeology | Not started | What existing alert, snooze, logging, widget and Watch work can we reuse? |
| 02 | V7 readiness + migration path | Not started | Stable 6.x, V7, or a deliberately portable hybrid? |
| 03 | Other diabetes apps: attention + alert patterns | Not started | What should we borrow from Loop, Trio/iAPS, AAPS, xDrip+ and related tools? |
| 04 | Low-friction meal/treatment logging on iOS | Not started | What is the fastest reliable way to log Ate, insulin and acknowledgements? |
| 05 | iOS notification + background constraints | Not started | What attention-engine behaviours are actually possible on iPhone and Watch? |
| 06 | Historical replay + backtesting | Not started | How can we tune alert logic against real historical CGM/treatment data? |
| 07 | Insulin-on-board + carbs-on-board | Not started | How early should approximate IOB/COB become part of context? |
| 08 | Meal photo recognition + carb estimation | Not started | Open source, API, or multimodal model for a personal prototype? |
| 09 | Apple Health + Watch context | Not started | Which exercise, sleep, HR and activity signals are useful and accessible? |
| 10 | Future personalised data model | Not started | What should we start recording from day one to support later learning? |
| 11 | Prediction + personalisation approaches | Not started | What existing forecasting approaches are worth adapting later? |
| 12 | Safety + failure modes | Not started | Where must the system become conservative or avoid false certainty? |
| 13 | Personal-use deployment | Not started | How do we make builds easy to install, update and run alongside Zukka? |
| 14 | Licensing + future distribution boundary | Not started | What changes if this moves beyond private personal use? |

## Findings already worth carrying forward

### Fork scan

The first fork scan found that most forks are snapshots, upstream syncs, or build/distribution variants rather than substantially different products. The high-signal findings were:

- **`mpereiragu/xdripswift-predict`** — useful prediction architecture, predictive alerts, local/remote separation, learning/evaluation logic, and especially walk-forward historical backtesting.
- **Historical local Insulin-on-Board branch** — useful reference for on-device IOB based on logged boluses and configurable insulin activity parameters; too old to merge directly, but relevant to the future Attention Engine.
- **Paul Plant calibration-assistant experiment** — useful interaction pattern: turn multiple noisy glucose/context signals into a simple recommendation rather than making the user interpret all inputs manually.
- No scanned fork appeared to implement the intended core Attention Engine: unresolved meal state + action awareness + glucose trajectory + adaptive escalation/snoozing + explicit user acknowledgement.

### V7

V7 is active development on Paul Plant's fork rather than the current main upstream release line. It is useful to study, but should not yet be assumed to be a stable sole foundation. A promising strategy is to keep new domain logic portable so it can start on stable 6.x if needed and migrate to V7 later without being rewritten.

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
