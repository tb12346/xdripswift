# 05 — iOS notification + background constraints

Status: **Complete**  
Research date: 2026-08-16

## Question

Can the proposed Attention Engine reliably re-evaluate glucose context, handle notification actions, shorten/escalate reminders, and implement ideas such as “remind me in 10 minutes unless I’m improving” under real iOS background-execution constraints?

## Executive conclusion

**Yes — if the engine is event-driven rather than timer-driven.**

The key architectural constraint is that iOS does not give a normal app a reliable “wake me every five minutes and run my rules” scheduler. Background refresh is discretionary, and `BGTaskRequest.earliestBeginDate` means “not before this date,” not “run at this exact date.” Short UIKit background tasks only let already-running work finish; they do not schedule future execution.

Fortunately, xDrip already has the right natural clock: **new glucose readings**.

The Attention Engine should re-evaluate on meaningful app events:

```text
new glucose ───────────────┐
treatment logged ──────────┤
user action ───────────────┼──> AttentionEngine.evaluate()
app launch/relaunch ───────┤          │
expired/defer state ───────┘          ↓
                                decision / episode
                                      │
                           schedule/cancel notification
```

For a request such as:

> Remind me in 10 minutes unless glucose starts recovering.

we should:

1. persist the defer state and deadline;
2. schedule a 10-minute local notification as an **OS-owned time guard**;
3. re-evaluate on every fresh glucose reading before that deadline;
4. cancel the fallback if the episode resolves;
5. escalate sooner if a fresh reading shows material worsening;
6. if no fresh data arrives, let the fallback fire with cautious wording rather than pretending we know the trajectory continued.

This is both technically realistic and a better safety model than trying to keep an app-level timer alive.

---

## 1. The Attention Engine should have no background polling loop

Do **not** build:

```text
Timer every 5 minutes
→ wake app
→ read latest glucose
→ AttentionEngine.evaluate()
```

That mental model conflicts with iOS background execution.

Apple's BackgroundTasks framework is designed for opportunistic refresh/processing. The system chooses when to launch a task, and setting `earliestBeginDate` only guarantees that the task will not start *before* that date — it does not guarantee execution at the requested time.

Likewise, `UIApplication.beginBackgroundTask(...)` extends the lifetime of work that is already underway. It is useful for finishing a small database write or evaluation after the app backgrounds, but it is not a future wake-up mechanism.

### Product consequence

The Attention Engine should be written as a pure evaluator that can execute quickly whenever the app gets a legitimate event. It should not own timers or assume continuous process lifetime.

---

## 2. New glucose readings are the physiological clock

The most useful reevaluation event is a fresh CGM reading because it gives the engine new evidence rather than merely elapsed time.

V7 already processes new glucose centrally in `RootApplicationCoordinator`. The direct-transmitter path calls `processNewGlucoseData(...)`, and the post-processing path subsequently runs the existing alert pass via `checkAlertsCreateNotificationAndSetAppBadge()` / `AlertManager.checkAlerts(...)`.

That gives us a natural seam:

```text
new reading arrives
→ existing xDrip storage/post-processing
→ build AttentionContext
→ AttentionEngine.evaluate()
→ persist episode decision
→ existing/new notification adapter
```

The engine can therefore reassess on the same event that xDrip already uses for glucose alerts.

### Why this is better than clock polling

A 5-minute timer asks:

> Has time passed?

A CGM event asks:

> Has the physiological evidence changed?

For this product, the second is what matters most.

---

## 3. Direct Bluetooth CGM mode is the strongest background path

V7's main app `Info.plist` currently declares:

```text
UIBackgroundModes
- audio
- bluetooth-central
```

Apple's Core Bluetooth background model allows apps that declare the central-role background mode to be woken from suspension for relevant BLE events. Core Bluetooth can also support state preservation/restoration when an app opts into that API.

This makes direct BLE CGM delivery a strong fit for event-driven Attention evaluation: when xDrip receives a new transmitter reading in the background, the Attention Engine can run as part of the same short processing pass.

### Important limitations

`bluetooth-central` does **not** mean unlimited background runtime.

Apple still constrains background Bluetooth behaviour and can suspend/terminate the process. State-restoration relaunch also has specific requirements and exceptions, including that user force-quit can prevent relaunch.

We did **not** independently confirm during this pass that every xDrip transmitter implementation opts into Core Bluetooth state restoration using a restore identifier. That should be verified during the V7 device/build qualification rather than assumed.

### Qualification test

On the actual CGM used for this project, test:

- screen locked for several hours;
- app backgrounded;
- app removed from memory by normal system pressure if reproducible;
- Bluetooth interruption/reconnection;
- phone restart if relevant;
- user force-quit behaviour documented separately.

Measure whether each real CGM reading results in the expected xDrip processing/alert path.

---

## 4. Follower/network modes are a different reliability class

V7 has a centralized `FollowerBackgroundKeepAliveManager`. Its own source describes a retained silent-audio mechanism with Normal, Aggressive and Continuous modes. It deliberately keeps networking outside the keep-alive manager: each follower source still owns its authentication, downloads, polling, retries and heartbeat behaviour.

That means the Attention Engine should **not** depend on the silent-audio timer or on a particular follower keep-alive strategy.

Instead:

```text
follower successfully obtains new reading
→ existing follower processing
→ AttentionEngine.evaluate()
```

If follower delivery is delayed, the Attention Engine cannot manufacture fresh glucose evidence. It should retain the unresolved episode and degrade to a stale/unknown-data state when appropriate.

### Product consequence

We should test direct-CGM and follower configurations separately. “Works in xDrip” is not enough evidence that they have identical background cadence.

---

## 5. Local notifications are time guards, not physiology evaluators

Apple's UserNotifications framework lets the app schedule a local notification and hand responsibility for delivery to the system. The request stays pending until its trigger occurs or the app explicitly cancels it. The app does not need to remain running for the system to interact with the user.

This is ideal for **fallback time boundaries**.

It is not equivalent to running the Attention Engine at that moment.

When we schedule:

```text
10-minute fallback notification
```

we are preparing its content *now*. If the app receives no execution event before the trigger, arbitrary new Attention logic does not execute at the notification's fire time merely because the notification is delivered.

Therefore the fallback copy must remain valid even without fresh glucose.

Bad fallback wording:

> You’re still rising.

unless we actually reevaluated a fresh reading.

Better:

> You asked me to check again. I haven’t seen enough fresh data to confirm this has resolved.

If fresh readings arrive before the deadline, we can cancel or replace that notification based on the latest context.

---

## 6. Exact design for “Remind me in 10 min unless improving”

### At user action

```text
user taps Remind 10 min
→ persist episode.deferUntil = now + 10m
→ persist last known context/reason
→ schedule fallback local notification for +10m
→ return quickly
```

### On each new glucose reading

```text
load active episode
→ evaluate fresh glucose + treatment/user context

if resolved / clearly recovering:
    cancel fallback
    resolve or downgrade episode

if materially worse:
    cancel/replace fallback
    notify earlier if policy says attention is now needed

if still unresolved but not worse:
    keep fallback pending
```

### If no new glucose arrives

The OS-owned fallback can still be delivered. Its job is to restore attention, not to claim that a physiological condition has continued.

On the next actual app execution event, immediately recalculate the episode using the latest data.

### Important distinction

`Remind 10 min` is therefore not a traditional “snooze everything for 10 minutes.”

It means:

> Do not bother me merely because time is passing; continue watching fresh evidence, resolve quietly if improving, escalate early if new evidence warrants it, and use 10 minutes as the backstop if nothing else happens.

---

## 7. Notification actions give us a short, legitimate background execution event

Apple's actionable-notification flow is well suited to the pass-04 interaction model.

When a user selects a custom notification action, iOS launches/wakes the app in the background and invokes the `UNUserNotificationCenterDelegate` response handler without bringing the app to the foreground.

That execution should be treated as a **short transaction**:

```text
receive action
→ validate input
→ persist treatment/user event
→ update episode
→ run AttentionEngine once
→ cancel/schedule notification as needed
→ finish callback
```

Do not perform long analytics, model training, large network syncs or other optional work before completing the notification response.

If a little extra runtime is genuinely needed to finish critical work that is already in progress, UIKit can request temporary background execution time, but system conditions can end it and it is not a scheduling mechanism.

---

## 8. BGTaskScheduler should not drive attention timing

`BGAppRefreshTask` can be useful for non-urgent maintenance such as:

- eventual cleanup;
- aggregate analytics;
- refreshing non-critical ancillary data;
- opportunistic future personalization work.

It should **not** be used to implement:

- re-alert in exactly 5 minutes;
- reevaluate an unresolved high at exactly 20:15;
- wake at every CGM interval;
- “if still rising in N minutes” physiology logic.

The OS decides when a background refresh actually runs.

### Recommendation

Attention logic should work correctly even if a `BGAppRefreshTask` never runs during an episode.

---

## 9. Persist episode state across process lifetime

Because the app process is not guaranteed to stay alive, the Attention Engine must not keep important state only in memory.

Persist at least:

- active episode ID;
- episode type/reasons;
- opened/last-evaluated timestamps;
- last processed glucose timestamp/ID;
- user acknowledgement state;
- defer deadline;
- linked treatment IDs / latest known treatment context;
- pending notification identifier;
- resolution state.

On startup, notification-action wake, or other reactivation:

```text
load active episode
→ read current glucose/treatments
→ reconcile scheduled notification state
→ evaluate once
```

This makes process termination a recoverable lifecycle event rather than a broken episode.

---

## 10. Evaluation must be idempotent

Background callbacks can overlap, duplicate or arrive after state has already changed.

The engine and adapters should tolerate:

- duplicate glucose callbacks;
- a notification action arriving near a new CGM reading;
- repeated treatment synchronization;
- app relaunch after the same reading was previously processed;
- stale scheduled notification identifiers.

A useful rule is:

> The same input snapshot should not create multiple user-facing alerts simply because evaluation was invoked twice.

Use stable episode IDs, reading timestamps/IDs and deterministic notification identifiers where practical.

---

## 11. Stale data is its own context, not evidence of continued rise

This is a critical safety/product implication.

If the engine last saw:

```text
9.8 mmol/L and rising
```

and then receives no glucose for 20 minutes, it must not reason:

```text
9.8 rising for another 20 min
```

The correct state is:

```text
latest glucose is stale / trajectory unknown
```

A stale-data state can itself require attention, but it is a **data-quality problem**, not evidence that the high/rise continued.

This also protects the product against follower/network gaps and Bluetooth interruptions.

---

## 12. Failure-mode matrix

| Situation | Expected Attention behaviour |
|---|---|
| Fresh direct-BLE glucose arrives in background | Re-evaluate immediately in existing reading-processing event. |
| Fresh follower glucose arrives | Re-evaluate on successful follower delivery. |
| Follower reading is late | Do not extrapolate indefinitely; retain episode and transition to stale/unknown if needed. |
| User logs insulin from notification | Persist treatment + event, re-evaluate once, update/cancel fallback, finish background callback quickly. |
| User selects `Remind 10 min` | Persist deadline + schedule OS local notification; keep evaluating on fresh readings. |
| Glucose improves before defer deadline | Resolve/downgrade and cancel pending fallback. |
| Glucose worsens before defer deadline | New reading may escalate before timer expires. |
| No glucose arrives before deadline | Fallback may alert using stale-data-safe wording; do not claim continued rise. |
| App process is suspended/terminated | Episode state remains persisted; pending local notification remains system-owned; reconcile on next execution. |
| Duplicate callbacks | Idempotent evaluation must avoid duplicate alerts/events. |
| Notification permission disabled | Attention notifications cannot be relied on; surface degraded notification capability prominently when app is active. |
| BG refresh delayed/throttled | No safety-critical Attention behaviour should depend on it. |
| User force-quits app | Do not assume Core Bluetooth will relaunch it; this must be documented/tested as a degraded operating state. |

---

## 13. Recommended technical boundary

Illustrative shape — still research, not final product spec:

```swift
protocol AttentionEvaluating {
    func evaluate(_ context: AttentionContext) -> AttentionDecision
}

protocol AttentionEpisodeStoring { ... }
protocol AttentionNotificationScheduling { ... }
protocol GlucoseHistoryProviding { ... }
protocol TreatmentHistoryProviding { ... }
protocol AttentionClock { ... }
```

App/service adapters invoke evaluation from:

```text
CGM/follower new-reading callback
TreatmentLoggingService
Attention notification response
app startup/reconciliation
foreground refresh
```

`AttentionNotificationScheduling` owns local notification requests/cancellation. `AttentionEngine` itself should not import UserNotifications or depend on a running Timer.

---

## 14. What we should reuse / adapt / build / avoid

| Area | Call |
|---|---|
| xDrip new-reading processing/callbacks | **Reuse as trigger** |
| Existing `AlertManager` / UserNotifications plumbing | **Reuse / adapt** |
| V7 `RootApplicationCoordinator` event wiring | **Reuse / extend** |
| `bluetooth-central` background mode | **Reuse** |
| Follower keep-alive | **Leave owned by follower subsystem** |
| Persisted Attention Episode | **Build** |
| Notification scheduler adapter | **Build** |
| Fast reconciliation on app wake/relaunch | **Build** |
| BGTaskScheduler for exact Attention timing | **Avoid** |
| App-level repeating background timer | **Avoid** |
| Attention-specific silent-audio keep-alive | **Avoid** |
| Extrapolating stale CGM as continued trend | **Avoid** |

---

## 15. Product-spec implications

1. **Attention is event-driven.** New glucose is the primary physiological reevaluation clock.
2. Time-based defer should schedule a **fallback local notification**, not a future background compute task.
3. A defer deadline must never prevent earlier escalation from a new worsening reading.
4. A new improving reading can silently cancel a scheduled reminder.
5. Scheduled fallback wording must remain valid if no fresh glucose was received.
6. Stale/missing CGM is an explicit state and cannot be treated as continued rise/fall.
7. Episode state must survive app suspension/termination and be reconciled on every execution entry point.
8. Notification actions should do minimal synchronous/transactional work and finish promptly.
9. Direct BLE and follower modes need separate on-device reliability qualification.
10. The Attention Engine should not depend on xDrip's follower silent-audio keep-alive.
11. Background refresh may support maintenance but is not part of the correctness path for real-time attention decisions.
12. We should record trigger/source (`newGlucose`, `treatment`, `notificationAction`, `launch`, etc.) for debugging and backtesting.

---

## 16. Open questions handed forward

### Pass 06 — historical replay / backtesting

The event-driven model suggests replay should use the same triggers as production:

```text
ordered glucose events
+ ordered treatment/user events
+ synthetic timer/defer-boundary events where relevant
```

Can the same evaluator and episode state machine reproduce live behaviour deterministically?

### Pass 07 — IOB

When a treatment is logged, what minimal IOB state should be recomputed synchronously so the next Attention decision reflects it immediately?

### Pass 12 — safety / failure modes

- What notification-permission state is acceptable for using Attention features?
- How should we handle user force-quit expectations?
- When does stale data require a separate alert?
- What confirmation/undo model is needed after background insulin logging?

### Pass 13 — personal-use deployment

The V7 qualification plan should explicitly test background reliability with the user's real CGM/data-source mode rather than merely confirming the app launches and receives foreground readings.

---

## 17. Key primary references

Apple Developer documentation reviewed:

- **Scheduling a notification locally from your app** — UserNotifications
- **Handling notifications and notification-related actions** — UserNotifications
- **Choosing Background Strategies for Your App** — BackgroundTasks
- **`BGTaskRequest.earliestBeginDate`** — BackgroundTasks
- **Extending your app's background execution time** — UIKit
- **Core Bluetooth Background Processing for iOS Apps**
- **TN3115: Bluetooth State Restoration app relaunch rules**

xDrip V7 source reviewed (read-only):

- `xDrip/Supporting Files/Info.plist`
- `xDrip/Managers/Application/RootApplicationCoordinator.swift`
- `xDrip/Managers/Followers/FollowerBackgroundKeepAliveManager.swift`

## Bottom line

The proposed Attention Engine is technically viable on iOS, but the implementation model matters enormously.

The robust pattern is:

> **React to fresh evidence; persist context; schedule conservative time guards; never rely on periodic background execution.**

That architecture is also well aligned with V7's existing new-reading and notification seams.