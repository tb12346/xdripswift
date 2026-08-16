# Research 10 — Apple Health + Watch Context

**Status:** Complete  
**Research date:** 2026-08-16

## Executive conclusion

**Use Apple Health primarily for recent completed-workout context first. Do not make raw heart rate, sleep stages, HRV, or passive movement part of the first Attention Engine.**

HealthKit can give us useful exercise history with enough structure to materially improve context: workout type, start/end, duration, active energy, distance and workout-associated heart-rate statistics. HealthKit can also wake the iPhone app when newly saved samples arrive if the app enables HealthKit background delivery.

However, HealthKit is not a guaranteed real-time feed from Apple Watch to an arbitrary iPhone app. Some sample types have system-enforced background-delivery limits, Watch-to-iPhone synchronization can be delayed, and truly live high-frequency workout data requires owning an active `HKWorkoutSession` / `HKLiveWorkoutBuilder` on Apple Watch. We should not add that complexity just to make the Attention Engine aware of exercise.

The recommended product sequence is therefore:

1. **P0 — completed workout context** from HealthKit:
   - workout type/group;
   - start/end;
   - duration;
   - time since workout;
   - active energy if available;
   - average/max heart rate if associated with the workout.
2. **P0 fallback — manually logged Exercise** using xDrip's existing treatment model when real-time context matters or HealthKit is unavailable/delayed.
3. **P1 — coarse recent movement** such as recent active energy / exercise time, only if replay shows it adds value beyond explicit workouts.
4. **P2 — sleep duration/regularity** as a later personalization feature, not an acute alert trigger.
5. **P2+ — resting HR / HRV / other recovery signals** only as weak longitudinal features after we have enough personal data to validate them.

The Attention Engine should use exercise as **context that changes confidence and urgency**, not as a direct insulin-dose modifier.

---

## Questions for this pass

- What Apple Health / HealthKit data can we actually read?
- How fresh and reliable is it in the background?
- Can completed Apple Watch workouts wake the iPhone app?
- Can we infer live exercise from Apple Watch without building a workout app?
- Which exercise signals matter for type 1 diabetes attention decisions?
- Do sleep, resting heart rate and HRV deserve early product complexity?
- How does V7 currently use HealthKit, and what would need to change?
- What should be stored locally versus left in HealthKit?

---

## 1. Exercise is a materially useful diabetes-context signal

The 2026 ADA Standards of Care are clear that physical activity can alter glucose behaviour in type 1 diabetes, and that the response varies with activity type, intensity, duration, starting glucose and circulating insulin.

Important points for this product:

- aerobic exercise commonly lowers glucose;
- high-intensity interval or resistance exercise can behave differently and can sometimes raise glucose acutely;
- all major exercise modes can influence glucose after the session;
- post-exercise hypoglycemia can occur for several hours because insulin sensitivity can remain increased;
- individual variation is substantial.

This means a simple boolean such as:

```text
exerciseRecently = true
```

is not enough to justify a deterministic physiological conclusion.

A better model is:

```text
RecentExerciseContext
- workout category
- endedAt
- duration
- active energy?
- average HR?
- max HR?
- data source / freshness
```

The Attention Engine can then treat exercise as a **confidence modifier**.

Example:

```text
glucose rising
+ insulin logged recently
+ meaningful recent aerobic workout

→ be more conservative about nagging for additional action
→ continue watching trajectory closely
```

But:

```text
glucose rising
+ recent high-intensity / resistance workout

→ do not assume glucose should be falling
```

No exercise-derived signal should itself generate an insulin-dose recommendation.

---

## 2. The strongest first signal is a completed `HKWorkout`

HealthKit workouts are structured objects rather than loose sensor samples.

A workout provides or can be associated with:

- `workoutActivityType`;
- start and end dates;
- duration;
- active energy;
- distance;
- workout events;
- associated quantity samples such as heart rate;
- workout-level statistics.

Apple's current HealthKit API lets an app calculate statistics for a workout from quantity samples associated with it. This is a clean way to obtain average/max heart-rate context without attempting to interpret the user's entire daily heart-rate stream.

### Recommended first-pass workout grouping

Do not expose the full HealthKit activity taxonomy to the Attention Engine.

Normalize it into a smaller domain enum:

```text
ExerciseKind
- aerobic
- resistance
- mixed
- highIntensity
- flexibilityLowIntensity
- other
- unknown
```

Illustrative mappings:

- running / walking / cycling / swimming → aerobic;
- traditionalStrengthTraining / functionalStrengthTraining → resistance;
- crossTraining → mixed;
- HIIT → highIntensity;
- yoga / stretching → flexibilityLowIntensity.

The exact mapping should remain data-driven and editable later.

---

## 3. Completed workouts are more valuable than raw step count

Step count is easy to access but weak as an early Attention Engine signal.

Problems include:

- steps do not capture cycling, swimming, rowing or resistance exercise;
- a large step count can accumulate gradually rather than represent a physiologically meaningful bout;
- HealthKit may coalesce samples;
- Apple explicitly documents that iOS step-count background delivery has an hourly maximum frequency.

Therefore:

**Do not use step count as the first exercise detector.**

A recorded workout has much better semantics.

Potential later fallback:

```text
no explicit workout
+ unusually high active energy / exercise time in the last N hours
→ weak recent-movement context
```

This should have lower confidence than an explicit workout.

---

## 4. Active energy is useful, but mainly as workout magnitude

`activeEnergyBurned` measures energy above resting expenditure and Apple Watch records it automatically.

For this project, its strongest early use is not:

```text
250 kcal = X% more insulin sensitive
```

That would create false precision.

Instead use it as one of several signals describing workout magnitude:

```text
45 min run
+ 430 kcal active energy
+ elevated workout HR
→ meaningful recent exercise episode
```

Later, personal historical replay may show that duration alone performs just as well. If so, remove energy from the decision path.

---

## 5. Heart rate: use inside workout context, not as a generic acute signal

HealthKit exposes ordinary heart rate, resting heart rate, walking heart-rate average, HRV and other cardiovascular metrics.

The first version should **not** feed arbitrary raw heart-rate samples directly into the Attention Engine.

A high heart rate can reflect:

- exercise;
- stress;
- caffeine;
- illness;
- heat;
- dehydration;
- posture;
- many other causes.

That is too nonspecific to alter a diabetes alert by itself.

### Better early use

Use average/max heart rate associated with a recorded workout as a proxy for workout intensity.

HealthKit supports workout-associated statistics and predicates for fetching objects associated with a workout.

Possible normalized feature:

```text
ExerciseIntensity
- low
- moderate
- high
- unknown
```

Initially, avoid a generic age-based formula if we do not need it. Personal historical HR distributions may later be more useful than a population estimate of maximum heart rate.

---

## 6. Live Apple Watch workout data is possible, but not worth P0 complexity

Apple Watch can provide high-frequency heart-rate samples during an active `HKWorkoutSession`.

An app that owns a workout session can:

- continue running on Watch in the background;
- receive sensor updates;
- use `HKLiveWorkoutBuilder` / `HKLiveWorkoutDataSource`;
- mirror workout state to iPhone.

This is the correct architecture for a dedicated workout app.

It is **not** the right first architecture for this project.

Reasons:

1. Apple Watch only runs one workout session at a time.
2. We do not want to compete with or replace Apple's Workout app merely to detect exercise.
3. It adds Watch UI, lifecycle, permissions, background modes and synchronization complexity.
4. The Attention Engine's core value does not require second-by-second heart rate.

### Decision

**Do not start a workout session purely for diabetes context.**

If a future version adds a genuinely valuable diabetes-specific workout experience, revisit this separately.

---

## 7. HealthKit background delivery helps, but is not real-time infrastructure

Apple provides `HKObserverQuery` plus `enableBackgroundDelivery(for:frequency:)`.

With the HealthKit Background Delivery entitlement enabled, HealthKit can wake the app after matching samples are saved.

Important constraints:

- the requested frequency is a **maximum** delivery frequency, not a service-level guarantee;
- some sample types have stricter system-enforced maximum frequencies;
- on iOS, Apple explicitly documents `stepCount` as having an hourly maximum;
- an observer callback only says that something changed; the app should run an anchored/sample query to fetch actual changes;
- the app must call the observer completion handler or HealthKit applies backoff and can eventually stop background delivery;
- device testing is required; Simulator does not model background server queries.

### Best first use

Register background delivery for `HKWorkoutType`.

Then:

```text
HealthKit saves completed workout
→ observer wakes app when available
→ anchored query fetches new workout(s)
→ normalize to RecentExerciseContext
→ persist derived context
→ AttentionEngine.evaluate() if relevant
```

This does **not** mean the engine learns that exercise began in real time.

---

## 8. Manual Exercise remains an important fallback

V7 already supports `.Exercise` as a treatment type.

That provides a useful low-tech path for cases where context matters immediately:

```text
user begins / finishes meaningful exercise
→ one-tap or normal Exercise log
→ Attention Engine knows immediately
```

Longer term we can make this low-friction through the same App Intent/control architecture as `Ate`.

This is preferable to making real-time correctness depend on Watch synchronization.

A completed HealthKit workout can later reconcile or enrich that manual event rather than create a duplicate episode.

---

## 9. Sleep should be collected later, not used for acute alert logic

HealthKit exposes sleep-analysis samples including:

- awake;
- core;
- deep;
- REM;
- asleep unspecified;
- in-bed where available.

The 2026 ADA Standards now explicitly recommend attention to sleep health in diabetes, and sleep/glycemia associations are real.

But the evidence does not justify a simple acute rule such as:

```text
slept 5h 42m
→ increase high-alert urgency by 20%
```

That would be pseudo-precision.

A 2026 randomized trial in adults with type 1 diabetes improved sleep health without a clear improvement in TIR or glycemic variability across the whole sample.

### Recommended later use

Treat sleep as a **personalization feature**, for example:

```text
lastNightSleepDuration
sleepDurationVsPersonalBaseline
sleepMidpoint
sleepRegularity
```

Then test whether these features explain the user's own repeated glucose-response patterns.

Do not use individual REM/deep/core percentages in the first model unless they prove predictive in personal data.

---

## 10. Resting HR and HRV are interesting but weak early signals

HealthKit exposes:

- `restingHeartRate`;
- `heartRateVariabilitySDNN`.

Apple notes that resting heart rate is an estimate derived from sedentary samples and may be revised during the current or previous day as the estimate improves.

That makes it unsuitable for a real-time diabetes rule.

HRV is also influenced by many factors and varies substantially between people.

Potential future use:

```text
7-day / 30-day personal baseline
vs
current-day deviation
```

as a weak feature for illness, stress or recovery context.

Do not let either one suppress or trigger an Attention alert in an early version.

---

## 11. V7 already has HealthKit, but only for exporting glucose

Current V7 `HealthKitManager.swift` is narrowly focused on writing xDrip blood-glucose readings into HealthKit.

It currently:

- creates only the `.bloodGlucose` quantity type;
- checks sharing/write authorization for that type;
- writes glucose samples;
- has no workout, activity, heart-rate or sleep queries;
- has no observer/anchored query system for reading health context.

Current app entitlement:

```text
com.apple.developer.healthkit = true
```

but V7 does **not** currently declare:

```text
com.apple.developer.healthkit.background-delivery
```

The Watch target currently has no HealthKit entitlement at all.

The iPhone Info.plist already includes `NSHealthShareUsageDescription`, but its copy says only that HealthKit is used to store blood-glucose readings. That description would need to be updated if we request workout/activity read access.

### Architecture recommendation

Do not turn the existing glucose-export class into a giant all-purpose HealthKit object.

Prefer:

```text
HealthKitManager
  → existing BG export responsibility

HealthContextProvider
  → read workouts / later activity / sleep
  → normalize + persist derived context
  → expose snapshots to AttentionEngine
```

Both can share one `HKHealthStore` abstraction later if useful.

---

## 12. HealthKit read permissions have an important privacy behaviour

HealthKit read access is deliberately privacy-preserving.

The app cannot reliably distinguish:

```text
user denied read access
```

from:

```text
there is no data
```

for a particular type.

Apple intentionally makes denied read access appear like no matching data exists.

This means Attention logic must model:

```text
exercise context unavailable
```

rather than interpreting an empty query as:

```text
user definitely did not exercise
```

That is analogous to our existing rule:

> no insulin logged does not prove no insulin was taken.

### Permission UX consequence

Request only the data types we actually use.

P0 request set could be as small as:

- workouts;
- heart rate;
- active energy burned.

Do not ask for sleep, HRV, steps, temperature and every other Health type just because they might someday be interesting.

---

## 13. Proposed normalized domain model

Illustrative research model:

```swift
struct RecentExerciseContext: Equatable, Sendable {
    enum Kind {
        case aerobic
        case resistance
        case mixed
        case highIntensity
        case flexibilityLowIntensity
        case other
        case unknown
    }

    let source: Source
    let kind: Kind
    let startedAt: Date
    let endedAt: Date
    let duration: TimeInterval
    let activeEnergyKcal: Double?
    let averageHeartRate: Double?
    let maximumHeartRate: Double?
    let observedAt: Date
}
```

with source such as:

```text
healthKitWorkout
manualTreatment
futureWatchSession
```

The Attention Engine receives the normalized domain value rather than importing HealthKit.

---

## 14. Do not duplicate raw HealthKit data into Core Data

HealthKit should remain the canonical raw store for Watch/Health data.

Persist only the small derived context necessary for deterministic behaviour and replay/debugging.

For example:

```text
HealthContextSnapshot
- calculatedAt
- sourceWorkoutUUID?
- workoutKind
- workoutStartedAt
- workoutEndedAt
- duration
- activeEnergy?
- averageHR?
- maxHR?
- freshness
```

Do not copy every heart-rate, step or sleep-stage sample into xDrip Core Data.

For historical personalization we can query HealthKit for the relevant date range, normalize it, and create a local research dataset outside Git.

---

## 15. Attention Engine behaviour should remain conservative

Potential safe-ish contextual effects to test in replay:

### Recent aerobic workout + active recorded insulin

Possible effect:

- reduce confidence that a rising glucose necessarily means more immediate action is needed;
- maintain short reassessment cadence;
- increase sensitivity to a subsequent downward turn.

### Recent exercise + falling glucose

Possible effect:

- treat the downward trajectory as more important context;
- avoid unnecessary high-glucose escalation following a transient post-exercise rise.

### High-intensity / resistance exercise

Possible effect:

- do not apply the same expected-downward assumptions as aerobic exercise.

### Unknown exercise type

Possible effect:

- mild context only; avoid strong suppression.

These are hypotheses for replay and later prospective testing, **not dosing rules**.

---

## 16. Replay should prove whether exercise adds value

Pass 06 gives us a clean way to test this.

Candidate policies:

```text
A. glucose + meal + insulin context only
B. A + time since completed workout
C. B + workout category
D. C + duration / active energy
E. D + workout HR intensity
```

Evaluate whether richer exercise context:

- reduces alerts shortly after reasonable insulin action;
- reduces repeated nags around exercise;
- improves low/falling-risk awareness;
- delays useful high alerts too often;
- adds predictive value beyond the glucose trend itself.

If B performs as well as E, choose B.

The product should prefer the smallest contextual model that materially improves attention outcomes.

---

## 17. Product priority

### P0

- HealthKit read authorization for workouts, heart rate and active energy.
- Separate `HealthContextProvider`.
- Query recent completed workouts.
- Normalize workout type.
- Capture duration/time-since-workout.
- Optionally extract workout average/max HR and active energy.
- Persist derived snapshot.
- Feed recent-exercise context into historical replay.

### P1

- HealthKit background delivery for workouts.
- Reconcile manual Exercise events with HealthKit workouts.
- Test coarse active-energy / exercise-time fallback when no workout exists.

### P2

- Sleep duration + regularity for personalization research.
- Resting HR / HRV baseline deviation.

### Not early

- dedicated live workout session;
- raw continuous Watch HR stream;
- sleep-stage-driven alert rules;
- universal exercise-to-insulin formulas;
- copying raw HealthKit history into xDrip storage.

---

## 18. Reuse / adapt / build / avoid

| Area | Call |
|---|---|
| Existing HealthKit capability | **Reuse** |
| Existing `HealthKitManager` BG export | **Keep / reuse** |
| Workout querying | **Build** |
| `HealthContextProvider` abstraction | **Build** |
| HealthKit workout background delivery | **Adapt / add** |
| Existing xDrip Exercise treatment | **Reuse as fallback** |
| Workout-associated HR statistics | **Use selectively** |
| Raw daily HR as acute signal | **Avoid early** |
| Step count as primary exercise detector | **Avoid** |
| Sleep as acute alert input | **Avoid early** |
| HRV/resting-HR acute alert rules | **Avoid early** |
| Dedicated Watch workout session | **Defer** |

---

## 19. Product-spec implications

1. **Recent exercise should become a first-class optional Attention context.**
2. Exercise absence must have an `unknown/unavailable` state; an empty HealthKit query cannot prove no exercise occurred.
3. Workout type matters. Do not model all exercise as glucose-lowering.
4. Completed HealthKit workouts are the preferred automatic source in the first version.
5. Existing manual Exercise logging remains the immediate/reliable fallback.
6. HealthKit background delivery is a convenience trigger, not a real-time correctness dependency.
7. The engine should consume normalized `RecentExerciseContext`, not HealthKit objects.
8. Raw health samples remain in HealthKit; only small derived snapshots should be persisted locally.
9. Request only minimum HealthKit read permissions initially.
10. Sleep and cardiovascular recovery signals belong in personalization research rather than first-version alert logic.
11. Exercise context modifies attention urgency/confidence and downward-risk awareness; it does not calculate insulin doses.
12. Historical replay must demonstrate incremental value before additional HealthKit signals are kept.

---

## 20. Open questions handed forward

### Pass 11 — future personalized data model

- Which derived HealthKit features should be recorded alongside every Attention decision for future learning?
- Do we persist only `RecentExerciseContext`, or also a feature snapshot such as `minutesSinceWorkout`, `exerciseKind`, `exerciseLoadBand`?
- How do we version derived features so replay remains reproducible?

### Pass 12 — prediction + personalization

- Does recent exercise materially improve personalized glucose-response prediction?
- Is workout category enough, or do duration/HR/energy materially improve results?
- Does sleep-duration deviation explain useful day-to-day sensitivity changes for this user?

### Pass 13 — safety/failure modes

- How should the engine behave when HealthKit permission is missing or data is delayed?
- When should automatic workout context expire?
- How should manually logged exercise and a subsequently synced HealthKit workout be deduplicated?

---

## Primary references reviewed

### Apple Developer

- Configuring HealthKit access
- Authorizing access to health data
- Executing Observer Queries
- Reading data from HealthKit
- `HKHealthStore.enableBackgroundDelivery(for:frequency:)`
- `HKUpdateFrequency`
- `HKWorkout`
- `HKWorkout.statistics(for:)`
- `HKQuery.predicateForObjects(from:)`
- Running workout sessions
- `HKWorkoutSession`
- `activeEnergyBurned`
- `restingHeartRate`
- `heartRateVariabilitySDNN`
- `HKCategoryValueSleepAnalysis`

### Diabetes evidence

- American Diabetes Association, *Standards of Care in Diabetes—2026*, Section 5: Physical Activity and Sleep Health
- American Diabetes Association, *Understanding Blood Glucose and Exercise*
- 2026 Diabetes Care randomized trial: *Sleep Optimization to Improve Glycemic Targets in Adults With Type 1 Diabetes*

### xDrip V7 source reviewed (read-only)

- `xDrip/Managers/HealthKit/HealthKitManager.swift`
- `xDrip/xdrip.entitlements`
- `xDrip/Supporting Files/Info.plist`
- `xDrip Watch App/xDrip Watch App.entitlements`

---

## Bottom line

The useful Apple Health integration is narrower than “ingest all Watch data.”

The first valuable version is:

> **Know that a meaningful workout happened recently, know roughly what kind it was, and let that make the Attention Engine more context-aware without pretending exercise has a deterministic glucose effect.**

That gives us most of the likely benefit with a small permission surface and modest technical complexity.