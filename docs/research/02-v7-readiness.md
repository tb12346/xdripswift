# 02 — V7 readiness + migration path

Status: **Complete**  
Research date: 2026-08-16

## Question

Should this project build on stable xDrip 6.x, Paul Plant's actively developed V7 branch, or deliberately keep new domain logic portable between both?

## Executive conclusion

**V7 should be our preferred technical base, but only after a short build/device qualification gate.**

The decisive point is architectural, not cosmetic. V7 moves the long-lived application services previously owned by `RootViewController` into `RootApplicationCoordinator`, while SwiftUI owns presentation. The coordinator still owns the backend services we care about: glucose access, treatments, alerts, Nightscout, HealthKit, Watch, Bluetooth, post-processing and related managers.

That is a much cleaner place to integrate an Attention Engine than 6.x, where `RootViewController` mixes application services, glucose callbacks and UIKit presentation.

V7 is still active beta development, though. Its upstream PR remains open, the branch is moving quickly, testers are still using beta builds, and we did not find public successful GitHub Actions runs for `v7-beta` during this pass. So our domain model should not depend on V7 UI types or rapidly changing screen code.

Recommended strategy:

1. Treat **V7 as the intended app foundation**.
2. Before feature work, verify the current V7 head builds, installs under a separate bundle ID and behaves reliably on the user's actual iPhone/CGM setup.
3. Build `AttentionEngine` as **pure, testable Swift domain logic behind protocols**, not inside SwiftUI views, UIKit controllers or notification extensions.
4. If V7 fails that qualification gate, start only backend/domain work on stable 6.x and postpone substantial UI work rather than building screens V7 replaces.

## Current V7 state

Upstream PR: `JohanDegraeve/xdripswift #730 — V7`

As of 2026-08-16:

- PR is open, not a draft and currently mergeable.
- Base is upstream `develop`; head is Paul Plant's `v7-beta` branch.
- A direct branch comparison showed `v7-beta` **331 commits ahead of `develop` and zero behind** at the time of research.
- The branch received commits on 2026-08-16, so it is active rather than abandoned.
- The PR describes a major presentation-layer migration, not a complete backend rewrite.

### What V7 changes

The complete primary iOS interface is rebuilt around SwiftUI, including:

- application entry point and navigation;
- Home, Treatments, Statistics, Devices and Settings;
- glucose charts;
- alarm configuration;
- Snooze and Sensor Management;
- Treatments list/editor;
- retirement of the old storyboard/UIKit presentation paths.

### What stays as the foundation

V7 explicitly retains the established backend services, including:

- CGM and Bluetooth communication;
- transmitter/follower integrations;
- glucose storage and calibration;
- alarms and notifications;
- Nightscout, Dexcom Share, HealthKit and calendar services;
- Apple Watch, widgets and Live Activities;
- Core Data and existing settings.

The PR also states that existing settings, alarms and locally stored data are retained when upgrading from 6.3.1.

## The key architectural seam: `RootApplicationCoordinator`

V7 introduces `xDrip/Managers/Application/RootApplicationCoordinator.swift`.

Its own documentation says it **owns the long-lived application services previously created by `RootViewController`**, while SwiftUI owns the complete root view hierarchy.

The coordinator owns/references, among other things:

- `CoreDataManager`
- `BgReadingsAccessor`
- `TreatmentEntryAccessor`
- `NightscoutSyncManager`
- `AlertManager`
- `HealthKitManager`
- `WatchManager`
- `BluetoothPeripheralManager`
- `BgPostProcessingManager`

The SwiftUI `XdripApp` creates one coordinator and one root state model and starts the service graph once.

For our project, the desirable shape is therefore:

```text
SwiftUI / App Intents / notification actions / Watch
                    ↓
              thin adapters
                    ↓
              AttentionEngine
             ↙      ↓       ↘
      glucose   treatments   episode state
      provider    provider    store
                    ↓
          existing xDrip backend
```

The Attention Engine should be instantiated/wired by the application layer, but its rules should remain independent of `RootApplicationCoordinator`, SwiftUI and UIKit.

## Alert integration seam

V7 still creates and owns the existing `AlertManager`. When a fresh reading is processed, the coordinator calls the shared alert evaluation flow (`alertManager.checkAlerts(...)`) before updating related notification state.

This gives us a natural future integration point:

```text
new glucose reading
      ↓
existing processing/storage
      ↓
AttentionEngine.evaluate(context)
      ↓
attention decision / episode update
      ↓
existing alert + notification machinery
```

We should not bury contextual rules inside SwiftUI views or individual `AlertKind` cases.

## Treatments remain compatible

V7's SwiftUI treatment editor still writes the existing Core Data `TreatmentEntry` model and continues Nightscout sync behaviour.

Current treatment types include insulin, carbs, exercise, BG check and note.

Therefore:

- logged insulin/carbs should continue to use xDrip treatments;
- we should not create a second insulin/carb database;
- project-specific states such as `Ate`, `handling`, `waitingForRecovery`, `noInsulinNeeded` and episode acknowledgement can live in a lightweight adjacent model.

## Build/platform findings

Current V7 project settings found in this pass:

- main iOS app: **iOS 16.2**;
- widget: **iOS 16.2**;
- notification content extension: **iOS 17.4**;
- Watch targets: **watchOS 10.0**.

So adopting V7 does not currently imply requiring iOS 18+ for the main app.

The branch's build workflow currently selects a macOS 26 runner and Xcode 26.2. We need to reproduce/verify the required local toolchain during qualification.

A query for public `build_xdrip.yml` runs on `v7-beta` returned none during this pass. In the V7 PR discussion, Paul Plant welcomed beta testing and referred to current testers, but at least one tester was still being directed toward local Xcode builds rather than a simple browser-build flow.

**Interpretation:** V7 is clearly being actively built/tested, but we should independently validate our own build and device path.

## Stability assessment

### Positive signals

- very active development;
- open upstream PR rather than isolated private work;
- explicit 6.x data/settings migration compatibility;
- backend retained rather than replaced;
- service lifecycle deliberately separated from UI;
- existing beta testers;
- tests added around several V7 areas;
- currently mergeable.

### Caution signals

- still an open beta PR;
- hundreds of commits of divergence;
- treatments, charts, followers and UI details still changing;
- public CI-green status for the beta branch was not independently verified;
- current toolchain needs qualification;
- this is diabetes software, so “beta seems fine” is not enough to make it the sole monitoring path without our own testing.

## Migration matrix

| Work | Portability | Recommendation |
|---|---|---|
| Attention state machine | Very high | Pure Swift; no UI dependencies. |
| Attention decision/reason model | Very high | Pure Swift structs/enums. |
| Episode/event model | Very high | Keep separate from presentation. |
| Historical replay/backtesting | Very high | Independent of app UI. |
| Local IOB calculation | Very high | Depend on treatment-provider protocol. |
| Glucose history abstraction | Very high | Wrap existing accessors. |
| Treatment history abstraction | Very high | Wrap existing storage. |
| Attention persistence | High | Own small store/model with migration tests. |
| Alert-policy adapter | High | Thin adapter to existing alert machinery. |
| App Intents | Medium-high | System API survives; wiring may move. |
| Notification actions | Medium-high | System API survives; integration may move. |
| Widget actions | Medium | Shared state/dependencies may need adaptation. |
| Watch actions | Medium | Shared domain survives; Watch integration evolves separately. |
| V7 SwiftUI screens | V7-specific | Fine if V7 qualifies. |
| 6.x UIKit Home/Treatments/Snooze UI | Low | Avoid investing here. |
| Storyboard navigation changes | Very low | Avoid. |

## Recommended Attention Engine boundary

Illustrative technical shape — not yet the product spec:

```swift
protocol GlucoseHistoryProviding { ... }
protocol TreatmentHistoryProviding { ... }
protocol AttentionEpisodeStoring { ... }
protocol AttentionClock { ... }

struct AttentionContext { ... }
struct AttentionDecision { ... }
enum AttentionReason { ... }

protocol AttentionEvaluating {
    func evaluate(_ context: AttentionContext) -> AttentionDecision
}
```

Later inputs such as approximate IOB, exercise context, meal-photo results and historical response models can sit behind interfaces.

The key rule is:

> **SwiftUI tells the engine what the user did; it does not decide what the diabetes context means.**

That lets the same evaluator run live, in App Intents/notification actions and in historical replay tests.

## V7 qualification gate before feature implementation

### Build/install

- bring current `v7-beta` into a branch in the user's fork;
- build with the required Xcode/toolchain;
- give it a distinct bundle/display name so it can coexist with Zukka;
- verify app, notification extension, widget and Watch targets sign correctly.

### Core monitoring

On the user's actual CGM setup, verify:

- readings arrive reliably;
- background behaviour is acceptable;
- high/low alarms fire correctly;
- snooze works;
- widget/Watch refresh behaviour is acceptable;
- treatments can be added and persist;
- no obvious duplicate/corrupt treatment behaviour.

Keep Zukka installed and functioning during qualification rather than relying on the beta/custom app as the only monitoring route.

### Decision

**If V7 passes:** base the project on V7.

**If it fails materially:** use stable 6.x only as a temporary host for portable backend work while the blocker is resolved. Avoid significant 6.x UIKit work.

## Final recommendation

**Preferred path: V7 + portable domain core.**

Why:

1. V7 is the clear direction of active xDrip development.
2. It preserves the backend services we depend on.
3. `RootApplicationCoordinator` gives us a cleaner integration boundary.
4. SwiftUI/App-Intent-era architecture fits our low-friction interaction goals better.
5. Building substantial UI on 6.x now risks immediate rework.
6. The remaining V7 risk can be tested cheaply with a build/device qualification gate.

## Reuse / adapt / build / avoid

| Area | Call |
|---|---|
| V7 backend managers/accessors | **Reuse** |
| `RootApplicationCoordinator` lifecycle seam | **Adapt / integrate with** |
| V7 SwiftUI shell | **Reuse** |
| V7 `TreatmentEntry` / Core Data model | **Reuse** |
| Existing `AlertManager` delivery machinery | **Reuse / adapt** |
| Attention Engine domain core | **Build separately** |
| Attention Episode persistence | **Build separately** |
| 6.x UIKit Home/Treatments/Snooze work | **Avoid** |
| CGM/Bluetooth rewrites | **Avoid** |

## Product-spec implications

1. Attention Engine is a domain service, not a screen feature.
2. Attention Episode state must survive UI lifecycle/relaunch.
3. The engine should re-evaluate from the new-reading pipeline, not view refresh timers.
4. Logged insulin/carbs should use existing `TreatmentEntry` data.
5. App Intent/widget/notification actions should write through the same domain layer as the app UI.
6. Historical replay should invoke the same evaluator used live.
7. The technical spec should separate domain rules, xDrip data adapters, notification policy/delivery and UI/system-surface adapters.
8. V7 base qualification should be the first practical implementation milestone before Attention Engine coding.

## Open questions handed to later passes

- **Pass 03:** How do Loop, Trio/iAPS, AAPS and xDrip+ model active treatment/expected response and context-aware alerting?
- **Pass 04:** Which App Intents/widget/notification actions should be first-class inputs into the Attention Engine?
- **Pass 05:** How reliably can contextual re-evaluation and interactive notifications run under iOS background constraints?
- **Pass 06:** What interfaces let live evaluation and historical replay use identical logic?
- **Pass 07:** Which local IOB model should the treatment provider expose for MDI use?
