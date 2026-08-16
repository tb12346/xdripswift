# 04 — Low-friction meal and treatment logging on iOS

Status: **Complete**  
Research date: 2026-08-16

## Question

What is the fastest reliable way to record the small pieces of context the Attention Engine needs — especially `Ate`, insulin taken, `handling`, `no insulin needed`, `waiting for recovery`, and short defer/remind actions — without forcing the user to open the main app?

## Executive conclusion

**Build one shared domain-action layer, expose it through App Intents, and use different system surfaces for proactive versus reactive actions.**

The best interaction model is not one universal widget.

### Reactive actions: put them on the alert itself

When the app is already asking for attention, the lowest-friction place to answer is the notification:

- `Log insulin` — text-input action for amount;
- `Remind 10 min` — one tap;
- `No insulin needed` — one tap when relevant;
- `Waiting for recovery` — one tap in post-low contexts;
- `Handling it` — one tap when a general acknowledgement is useful.

Apple's UserNotifications framework can deliver custom notification actions directly to the app in the background without opening the foreground UI. Text-input notification actions accept typed or dictated text. Notifications forwarded to Apple Watch retain actionable controls, and text input can be dictated on the Watch.

This is stronger than the original assumption that most logging would need a widget.

### Proactive actions: App Intents first

For actions the user initiates before any alert exists:

- `Ate` should be a zero-parameter App Intent;
- `Log insulin` should be an App Intent with a numeric amount parameter;
- later optional intents can cover meal photo, exercise context or other explicit state.

The same intents can then be reused by Siri, Shortcuts, Spotlight, compatible widgets, controls, the Action Button and future system surfaces.

### Best one-tap proactive surface on modern iOS

For iOS 18+ devices, **WidgetKit Controls** are a better locked-phone action surface than ordinary interactive widgets. Controls can live in Control Center, on the Lock Screen and on the Action Button, and execute App Intents. Their authentication policy can be selected per action.

A Lock Screen `Ate` control is therefore a strong candidate.

### Important limitation

Ordinary interactive widgets and Live Activity buttons are **not** truly one-tap from a locked phone: Apple documents that their controls are inactive until the user authenticates/unlocks.

Also, widgets do not resolve missing App Intent parameters at tap time. A widget button must already have the parameter values it needs. That makes widgets excellent for zero-parameter actions such as `Ate`, but weak for arbitrary insulin amounts unless the amount is preconfigured.

### Modern iOS 26-era enhancement

Current App Intents also support interactive snippets in Siri, Spotlight and Shortcuts. These can show confirmation/result UI and follow-up buttons without opening the app. That could make a future `Log insulin` Shortcut flow significantly nicer, but it should be progressive enhancement rather than the core dependency because the xDrip base still supports older iOS versions.

---

## 1. Build actions once, then expose them everywhere

The fundamental technical decision should be:

```text
system surface
(notification / Siri / widget / control / Watch / app UI)
        ↓
shared domain action
        ↓
TreatmentLoggingService / AttentionEpisodeService
        ↓
Core Data + Attention Engine
```

Do **not** implement separate business logic for notification logging, widget logging and in-app logging.

Illustrative actions:

```swift
recordMealEvent()
recordInsulin(amount: Double)
recordAttentionAcknowledgement(.handling)
recordAttentionAcknowledgement(.noInsulinNeeded)
recordAttentionAcknowledgement(.waitingForRecovery)
deferAttention(minutes: Int)
```

App Intents and notification callbacks should be thin adapters around these actions.

This matters because the Attention Engine must see the same event regardless of whether it came from:

- the app;
- the Lock Screen;
- Siri;
- Apple Watch;
- a notification.

The event should also record its **source surface** for later UX analysis, without changing its clinical meaning.

---

## 2. Existing V7 groundwork is unusually good

V7 already contains several useful seams.

### Existing App Intent

`xDrip/GlucoseIntent.swift` already defines an `AppIntent` that:

- creates a Core Data manager;
- reads glucose history;
- performs without requiring the main app UI;
- returns dialog and a SwiftUI result view;
- currently uses `IntentAuthenticationPolicy.alwaysAllowed`.

This proves App Intents are already part of the project and can access the existing data layer.

### Existing notification-action architecture

`AlertManager` already:

- registers notification categories;
- registers a `Snooze` `UNNotificationAction`;
- receives notification responses;
- mutates snooze state/Core Data in response.

`RootApplicationCoordinator` is already the central `UNUserNotificationCenterDelegate` in V7 and forwards alert responses into `AlertManager`.

So adding semantic Attention actions does not require a second notification framework.

### Existing Watch notification UI

The Watch target already has a `WKUserNotificationHostingController` (`NotificationController`) for xDrip alerts.

Apple allows Watch notification interfaces to expose actionable notification buttons, text-input actions, dictated input and suggested text responses.

This gives us a realistic route to responding to Attention notifications from the wrist without first building a new full Watch logging UI.

### Existing widget bundle

The V7 WidgetKit bundle already contains the glucose widget and Live Activity. Modern controls can be added to the WidgetKit architecture as an availability-gated extension rather than requiring a new product architecture.

---

## 3. Notification actions — highest-value reactive surface

### What iOS supports

Actionable notifications can contain multiple `UNNotificationAction`s. Selecting a normal action causes iOS to launch/wake the app in the **background** and deliver the response to the notification-center delegate without bringing the full app UI forward.

A `UNTextInputNotificationAction` can collect text by typing or dictation and include it in the response.

Banner notifications display only the **first two** registered actions, although expanded notification interfaces can expose more.

### Design consequence: actions must be contextual

Do not attach six generic actions to every alert.

Instead define a small number of static notification categories and choose the appropriate category based on Attention Episode state.

Example:

#### Rising / no recent insulin logged

Primary banner actions:

1. `Log insulin`
2. `Remind 10 min`

Additional expanded action, if useful:

- `No insulin needed`

#### Post-low recovery rise

Primary actions:

1. `Waiting for recovery`
2. `Remind 10 min`

#### Ambiguous unresolved episode

Primary actions:

1. `Handling it`
2. `Remind 10 min`

This is more usable than a generic Snooze button because the response becomes structured context for subsequent re-evaluation.

### `Log insulin` via notification text input

A text-input notification action is a practical way to log arbitrary insulin amounts without opening the app.

Potential flow:

```text
Attention alert
→ tap “Log insulin”
→ type/dictate “1.5”
→ validate
→ create insulin TreatmentEntry
→ link/update Attention Episode
→ re-evaluate urgency
→ confirmation / updated notification state
```

Validation must be strict:

- numeric only;
- locale-aware decimal parsing;
- positive value;
- sensible upper bound configured conservatively;
- invalid input must not silently become zero or another value.

The system must never infer an insulin amount from vague free text.

### Apple Watch advantage

If an iPhone notification is forwarded to Apple Watch, actionable notification controls remain available. Watch supports text input by dictation, and a Watch notification controller can provide suggested responses.

This suggests a useful future experiment for insulin-sensitive users who use a small recurring set of amounts:

- suggested numeric responses such as configured recent/favourite amounts;
- still allow dictation for another amount.

This must be treated as **logging**, not a dosing recommendation. Suggested values should be user-configured/recent logging shortcuts, never generated dose advice.

---

## 4. App Intents — the common proactive API

Create App Intents for the actual user actions, not for UI concepts.

### `AteIntent`

Requirements:

- no carb amount required;
- timestamp defaults to now;
- records a lightweight meal-context event;
- updates/creates the relevant Attention Episode;
- returns a terse confirmation;
- optional undo path.

This should be one of the highest-priority intents because it directly addresses the user behaviour we are designing for.

### `LogInsulinIntent`

Required parameter:

- insulin amount as a numeric value.

Optional later parameters:

- timestamp;
- insulin type if the user ever uses more than one relevant insulin profile.

App Intents can ask the user for unresolved required parameters in system experiences such as Siri/Shortcuts, and the framework can resolve spoken numeric input into numeric parameter types.

Examples:

```text
“Hey Siri, log one unit of insulin”
“Log 0.5 units in [app]”
```

The exact phrases belong in product/implementation design later, but the framework supports the required interaction model.

### App Shortcuts

Expose the high-value intents as App Shortcuts so they become available across:

- Siri;
- Shortcuts;
- Spotlight;
- supported Action Button flows.

A zero-parameter `Ate` shortcut can run immediately. An amount-based insulin intent can collect the missing parameter through supported system interaction.

---

## 5. WidgetKit Controls — best modern proactive one-tap surface

Apple introduced WidgetKit Controls in iOS 18.

Controls can appear in:

- Control Center;
- Lock Screen;
- Action Button configuration.

They are explicitly designed for discrete app actions that should happen without navigating through the main app.

### Strong candidates

#### `Ate`

Excellent fit:

- discrete action;
- no input required;
- very frequent;
- should not open the app.

#### `Handling it`

Potential fit, but less useful globally because it needs an active Attention Episode. It may be better as a notification action.

#### `Waiting for recovery`

Same: technically possible, but context-dependent and therefore stronger on a notification than as a permanent Lock Screen control.

#### `Log insulin`

A generic control is a weaker fit because controls are buttons/toggles rather than arbitrary numeric-entry interfaces.

Possible alternatives:

- control opens an App Intent/Shortcut flow that gathers amount elsewhere;
- user-configured fixed-amount controls;
- open a minimal quick-log app route.

Do not make a collection of hard-coded dose buttons the default UX. Incorrect insulin logging has much greater downstream consequence than an `Ate` event.

### Authentication

App Intents support three authentication policies:

- always allowed, including when locked;
- authentication required;
- local-device authentication required.

Controls can use those policies and can redact sensitive state on the Lock Screen.

We should choose policy **per action**, based on the consequence of an accidental/unauthorised event, rather than globally.

This decision belongs partly in the later safety pass. At minimum, false insulin logs require substantially more caution than a lightweight `Ate` context marker.

---

## 6. Ordinary interactive widgets — useful, but not the primary Lock Screen solution

Interactive WidgetKit buttons/toggles can perform App Intents without launching the app.

They are useful for a Home Screen glucose/context widget containing actions such as:

- `Ate`;
- `Handling it` if an episode is active;
- `Remind me` if an episode is active.

However:

1. Apple documents that widget/Live Activity buttons are inactive on a locked device until the user authenticates/unlocks.
2. Widget intents **cannot resolve missing parameters at tap time**; required inputs must already have values.

Therefore:

- `Ate` is a great widget action;
- arbitrary `Log insulin` is not naturally a one-tap widget action;
- WidgetKit Controls are better for a permanent Lock Screen action on iOS 18+.

V7's widget extension currently supports older iOS versions, so interactive functionality should be availability-gated rather than forcing the entire app's minimum OS upward unnecessarily.

---

## 7. Live Activities / Dynamic Island — secondary, not core logging UI

V7 already has Live Activity infrastructure.

An Attention Episode could eventually have a Live Activity showing a compact unresolved state, for example:

```text
Rising • insulin logged 12m ago
Monitoring
```

or:

```text
Waiting for recovery
```

Interactive buttons are technically possible using App Intents, but locked-device interaction has the same authentication/unlock limitation as widgets.

**Recommendation:** do not make a Live Activity part of the first Attention Engine prototype. It risks adding persistent visual diabetes presence — exactly the cognitive load the product is trying to reduce.

Consider it later only if testing shows that a visible ongoing episode actually reduces repeated checking.

---

## 8. Interactive snippets — powerful progressive enhancement

Current App Intents support interactive snippets in Siri, Spotlight and Shortcuts.

They can:

- show state/results;
- contain follow-up buttons/toggles;
- request confirmation;
- update without opening the full app.

Apple explicitly describes confirmation snippets where a person adjusts values and confirms an action.

This could eventually make a polished system-level insulin log flow:

```text
invoke “Log insulin”
→ system gathers amount
→ snippet: “Log 1.0 U now?”
→ Confirm / Cancel
→ result: “1.0 U logged”
```

Important limitations:

- Control Center controls cannot display snippets;
- current snippet UI is not a watchOS surface;
- this is newer platform capability and should not be required for the first implementation.

Treat snippets as a modern enhancement over a simpler App Intent/Shortcuts foundation.

---

## 9. Home Screen Quick Actions — compatibility fallback

V7 still retains UIKit Home Screen Quick Action handling through `AppDelegate` and `QuickActionsManager`.

These are useful for older supported iOS versions or as a fallback, but are not the preferred primary interaction because they require:

```text
long-press app icon → choose action
```

and commonly bring the app into an active lifecycle.

If cheap to adapt, an `Ate` Quick Action is worthwhile. Do not center the architecture on it.

---

## 10. Apple Watch strategy

### Phase 1: notification actions

Highest value and lowest implementation cost.

When an Attention alert reaches the Watch:

- show concise context;
- expose the relevant acknowledgement/defer actions;
- support dictated insulin amount through text input when appropriate.

V7 already has custom Watch notification presentation, so we are extending an existing seam.

### Phase 2: proactive `Ate`

Options include:

- a small Watch app quick-log control;
- Smart Stack / Control integration on supported newer watchOS versions;
- complication tap into a minimal logging route.

The best choice should be tested against actual tap count and reliability rather than assumed.

### Do not begin with a complex Watch treatment editor

The core use case is deliberately simpler than Loop/Trio treatment entry. `Ate` should remain extremely lightweight.

---

## 11. Interaction ranking

| Surface | Best use | Opens app? | Locked-phone value | Arbitrary insulin amount | Watch value | Priority |
|---|---|---:|---:|---:|---:|---:|
| Notification actions | Respond to active Attention Episode | No, background by default | **High** | **Yes, text/dictation** | **High** | **P0** |
| App Intents / Siri | Proactive Ate + insulin logging | Usually no | High depending auth | **Yes** | Medium-high | **P0** |
| iOS 18+ Controls | Proactive one-tap Ate | No | **Very high** | Weak unless preconfigured | Growing | **P1** |
| Interactive Home widget | Ate + active episode buttons | No | Medium; unlock needed | Weak | Widget/complication separate | **P1** |
| App interactive snippets | Confirmation/result flows | No | High in supported system flows | **Good** | Not watchOS | **P2** |
| Watch notification actions | Alert acknowledgement + insulin text | No foreground UI | N/A | **Yes, dictation** | **Very high** | **P1** |
| Home Screen Quick Actions | Compatibility fallback | Usually foreground lifecycle | Low | Could deep-link | N/A | **P2** |
| Live Activity buttons | Ongoing episode interaction | No | Unlock limitation | Weak | Some ecosystem value | **Later** |
| Full Watch treatment editor | Detailed treatment entry | Watch app | N/A | Yes | High but higher friction | **Later** |

---

## 12. Recommended first interaction set

### Proactive

1. **`Ate` App Intent**
2. **`Log insulin` App Intent with amount**
3. Home/widget button for `Ate`
4. iOS 18+ Lock Screen / Control Center `Ate` Control
5. Siri/App Shortcut exposure for both intents

### Reactive Attention notification

Register context-specific categories containing no more than two truly primary banner actions.

Start with:

- `Log insulin` — text input;
- `Remind 10 min`;
- `No insulin needed` where appropriate;
- `Waiting for recovery` where appropriate;
- `Handling it` only if it proves semantically useful beyond the more specific choices.

The notification action should write structured state and immediately re-run the Attention Engine rather than simply dismissing the alert.

---

## 13. Important product challenge: do we need both “Handling it” and specific states?

This pass raises a useful challenge to the initial action list.

`Handling it` may be too vague if we already support:

- insulin logged;
- no insulin needed;
- waiting for recovery;
- remind me in 10 minutes.

A generic acknowledgement is useful only when none of those describe reality.

**Recommendation:** keep it in the domain vocabulary for now, but do not automatically expose it on every surface. Test whether the specific actions cover most real situations.

Fewer choices mean lower cognitive load.

---

## 14. Data-model consequence

Every low-friction interaction should produce a structured event, not just a UI-side flag.

Example conceptual model:

```text
AttentionUserEvent
- id
- timestamp
- type
  - ate
  - insulinLogged
  - handling
  - noInsulinNeeded
  - waitingForRecovery
  - defer
- optional amount
- source
  - app
  - notification
  - siri
  - shortcut
  - widget
  - control
  - watch
- linkedTreatmentID?
- linkedEpisodeID?
```

Insulin itself should still be stored as the existing xDrip `TreatmentEntry`; the event can link to it if useful.

This source/event structure will later let us answer questions such as:

- which surface is actually used;
- which action most often resolves an episode;
- whether `Ate` events are frequently followed by insulin;
- how often a “remind me” becomes a later escalation;
- whether Watch actions materially reduce missed responses.

---

## 15. Safety / trust considerations handed forward

Low-friction input increases the cost of accidental input.

Important distinctions:

- a false `Ate` event changes context but should never be treated as proof of insulin;
- a false insulin log may suppress/escalate alerts incorrectly and corrupt later IOB/personalization;
- `no insulin needed` is an explicit user judgement, not a physiological fact;
- `waiting for recovery` should expire/re-evaluate as glucose evidence changes;
- `remind me` must not become an indefinite fixed snooze.

The later safety pass should define authentication, confirmation and undo requirements for each event type.

A useful design principle is:

> the easier an action is to perform, the easier it should be to see and undo — and the less the engine should infer from a single action without corroborating glucose/treatment evidence.

---

## Reuse / adapt / build / avoid

| Area | Call |
|---|---|
| Existing V7 `GlucoseIntent` App Intent pattern | **Adapt** |
| Existing Core Data/treatment storage | **Reuse** |
| Existing `AlertManager` notification categories/actions | **Adapt** |
| V7 `RootApplicationCoordinator` notification delegate | **Reuse / extend** |
| Existing Watch notification controller | **Reuse / extend** |
| Existing WidgetKit bundle | **Reuse / extend** |
| Shared treatment logging domain service | **Build / extract** |
| Shared Attention user-action service | **Build** |
| iOS 18+ Lock Screen Control for Ate | **Build** |
| Separate business logic per surface | **Avoid** |
| Generic six-button alert | **Avoid** |
| Fixed-dose controls as primary insulin UX | **Avoid** |
| Live Activity as default Attention UI | **Defer / likely avoid initially** |

---

## Product-spec implications

1. **App Intents are first-class product infrastructure**, not an optional integration.
2. **Notification actions are the primary reactive UI** for Attention Episodes.
3. `Ate` should be a true one-step domain action with no required carb amount.
4. `Log insulin` should accept a precise amount and write the existing treatment store.
5. Use **context-specific notification categories**, with at most two primary banner actions.
6. All surfaces must call the same domain services and create the same structured events.
7. Controls/widgets/Watch are adapters, not owners of business logic.
8. Progressive enhancement is preferable to raising the whole app's minimum OS merely for newer system surfaces.
9. Keep `handling` as a possible domain state, but challenge whether it needs prominent UI once more specific responses exist.
10. Record source surface for interaction research and later personalization.
11. After every user action, the live Attention Episode should be immediately re-evaluated.
12. Authentication/confirmation/undo policy should be defined per action in the safety pass, especially for insulin logging.

---

## Open questions passed forward

### Pass 05 — iOS notification/background constraints

- After a notification action is handled in background, how much work can we reliably perform before suspension?
- Can we immediately schedule/cancel/rewrite the next attention notification reliably?
- How does CGM source/background mode affect re-evaluation cadence?
- What parts must be event-driven from new glucose versus timers?
- How reliable are condition-based “remind me when this changes” semantics under iOS scheduling?

### Pass 07 — IOB

- What exact treatment fields and insulin model does `LogInsulinIntent` need so a newly logged injection updates approximate IOB immediately?

### Pass 12 — safety

- Which actions require authentication?
- Which require confirmation?
- What is the undo window?
- How should the engine behave after a potentially erroneous insulin log?

---

## Key primary references

Apple Developer documentation:

- App Intents — `developer.apple.com/documentation/appintents`
- Adding parameters to an app intent
- App Shortcuts
- IntentAuthenticationPolicy
- WidgetKit: Adding interactivity to widgets and Live Activities
- WidgetKit Controls / Creating controls to perform actions across the system
- UserNotifications: Declaring actionable notification types
- UserNotifications: Handling notifications and notification-related actions
- `UNTextInputNotificationAction`
- watchOS: Adding actions to notifications on watchOS
- App Intents: Displaying static and interactive snippets

xDrip V7 source reviewed:

- `xDrip/GlucoseIntent.swift`
- `xDrip/Managers/Alerts/AlertManager.swift`
- `xDrip/Managers/Application/RootApplicationCoordinator.swift`
- `xDrip Watch App/DataModels/NotificationController.swift`
- `xDrip Widget/XDripWidgetBundle.swift`
