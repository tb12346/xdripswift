# Research 11 — Future Personalised Data Model

**Status:** Complete  
**Research date:** 2026-08-17

## Executive conclusion

**Record durable facts, actions, provenance and corrections from day one; derive physiological features later.**

The app should not become a warehouse of precomputed features such as “IOB at 12:35”, “recent-exercise score = 0.72” or “meal impact = high”. Those values depend on models and assumptions we will change. Instead, preserve the underlying event history well enough that future versions can recompute new features and replay old episodes using improved logic.

The recommended architecture is an **append-oriented local Attention event ledger** alongside xDrip's existing canonical glucose and treatment stores:

```text
existing xDrip facts
- BgReading
- TreatmentEntry
        │
        ├───────────────┐
        ↓               ↓
Attention event ledger  imported context
- Ate                  - HealthKit workout summary
- manual exercise      - future sleep summary
- acknowledgement     - meal-vision estimate
- defer/recovery      - other external observations
- correction
        │
        ↓
AttentionEpisode
        │
        ↓
AttentionDecisionEvent
        │
        ↓
derived/replay features
(IOB, exercise recency, meal response, personal patterns, future ML)
```

Three principles matter most:

1. **Facts and estimates must never be conflated.** A user-confirmed 45 g meal is different from an AI estimate of 35–55 g. A manually logged workout is different from a HealthKit workout import. Missing insulin is different from known zero insulin.
2. **Occurred time and recorded time must both be retained.** This is especially important because meals, insulin and exercise may be logged after they happened.
3. **Provenance and revisions are future training data.** Record where an event came from, what model/policy produced it, and what the user later corrected.

For this project, manual exercise is explicitly first-class. HealthKit workout import is useful enrichment, not a prerequisite. Likewise, HealthKit glucose read-back is unnecessary for the core architecture because xDrip/Zukka is already the glucose source that writes to HealthKit; reading the same values back would add duplication and source ambiguity without useful new information.

---

## Questions for this pass

- What should we start storing from the first prototype so future personalization is possible?
- What should remain in existing xDrip/Core Data versus a new Attention store?
- Which fields are essential for replay, debugging and later ML?
- How should manual and automatically imported events coexist?
- How do we preserve uncertainty instead of turning missing data into false facts?
- Which derived features should *not* be persisted?
- How should corrections, model versions and policy changes be represented?
- What can we safely avoid collecting until later?

---

## 1. Do not duplicate xDrip's canonical glucose and treatment history

The existing xDrip model is already the canonical local store for:

- CGM/BG readings;
- insulin treatments;
- carb treatments;
- exercise treatments;
- notes and BG checks;
- Nightscout synchronization identifiers/state.

Those records should remain the source of truth for standard diabetes facts.

The new Attention layer should **reference** those facts rather than copy every glucose reading and treatment into a second database.

Example:

```text
AttentionEpisode
- id
- openedAt
- relatedTreatmentIDs[]
- relevant glucose time range
```

Rather than:

```text
AttentionEpisode
- copied glucose value 1
- copied glucose value 2
- copied insulin amount
- copied carb amount
- ...
```

Why:

- avoids sync divergence;
- avoids double storage of high-volume CGM data;
- preserves compatibility with xDrip/Nightscout;
- makes upstream V7 changes easier to merge;
- lets replay query the original history.

### HealthKit glucose

Do **not** add HealthKit glucose as another canonical input merely because permission is available.

In the intended setup, xDrip/Zukka writes the glucose into HealthKit. Reading it back gives us another copy of data that originated in the same app family, often with additional synchronization delay.

Potential later exception:

- if a genuinely independent glucose source is ever used through HealthKit, add it deliberately behind source/deduplication rules.

For the current product, the useful direction is:

```text
xDrip glucose → optional HealthKit export
```

not:

```text
xDrip → HealthKit → xDrip Attention Engine
```

---

## 2. Add a separate Attention store rather than stretching TreatmentEntry

Standard clinical/treatment facts should stay in `TreatmentEntry`.

Do not force app-specific semantics such as:

- Ate, carbs unknown;
- no insulin needed;
- waiting for recovery;
- remind me in 10 minutes;
- handling it;
- notification acknowledgement;
- AI meal estimate;
- Attention Episode lifecycle;
- policy decision trace;

into treatment rows.

These are not all treatments and many have different revision/retention semantics.

### Recommended persistence boundary

Prefer a separate local persistence component:

```text
AttentionStore
```

implemented behind a protocol and using its own schema/persistent store.

A separate Core Data container/model is a reasonable implementation candidate because xDrip already uses Core Data, but the product-spec requirement should be persistence-technology agnostic.

Benefits:

- minimal changes to upstream xDrip data model;
- less merge conflict with V7 development;
- explicit ownership of app-specific state;
- easier versioned export/backup;
- easier unit/replay testing with an in-memory store;
- Attention data can evolve rapidly without destabilizing CGM/treatment persistence.

V7 backup/export should eventually include this store.

---

## 3. Use a common event envelope

Every new event should share a small set of fields regardless of payload.

Illustrative model:

```swift
struct AttentionEventEnvelope {
    let id: UUID
    let type: AttentionEventType

    // when the real-world event happened
    let occurredAt: Date

    // when the app first learned/recorded it
    let recordedAt: Date

    // useful for time-of-day personalization, especially during travel
    let timeZoneIdentifierAtOccurrence: String?
    let utcOffsetSecondsAtOccurrence: Int?

    let provenance: EventProvenance
    let certainty: EventCertainty

    let linkedTreatmentID: String?
    let linkedEpisodeID: UUID?
    let externalIdentifier: String?

    // correction/revision relationship
    let supersedesEventID: UUID?

    let schemaVersion: Int
}
```

### Why `occurredAt` and `recordedAt` both matter

Example:

```text
18:00 workout actually happened
19:10 user remembers and logs it
```

For physiology and personalization, the event belongs at 18:00.
For evaluating logging behaviour and the Attention experience, we also need to know that the system only became aware at 19:10.

Collapsing those into one timestamp creates retrospective look-ahead leakage.

The same distinction applies to:

- delayed insulin logging;
- back-entered meals;
- manually added exercise;
- corrections;
- imported HealthKit workouts arriving after completion.

---

## 4. Provenance should be structured, not a free-text `enteredBy`

Future learning will need to distinguish events that look identical numerically but have different reliability.

Suggested provenance categories:

```text
EventSource
- app
- notification
- siri
- shortcut
- widget
- control
- watch
- manualHistoricalEntry
- healthKit
- nightscout
- mealVision
- import
- systemDerived
```

And useful associated fields:

```text
sourceAppIdentifier?
sourceRecordIdentifier?
sourceVersion?
deviceIdentifierClass?   // not advertising/device fingerprinting
provider?
modelName?
modelVersion?
```

For HealthKit imports specifically, preserve the HealthKit object UUID plus relevant source revision/device provenance where available. Apple exposes `sourceRevision` on HealthKit objects specifically to identify the app/device source and its version.

HealthKit also has sync identifier/version metadata for applications that need stable replacement semantics. We do not need to copy Apple's model blindly, but the pattern validates keeping **stable origin ID + revision/version** rather than relying on timestamps alone.

For Nightscout, preserve stable remote identifiers used by its treatment/history APIs when importing or linking remote records.

---

## 5. Certainty is a data type, not UI copy

Avoid representing uncertainty with nullable values alone.

Useful high-level distinction:

```text
EventCertainty
- observed
- userReported
- userConfirmed
- externallyReported
- modelEstimated
- derived
- unknown
```

Examples:

```text
CGM reading
→ observed

manual Exercise
→ userReported

HealthKit workout
→ externallyReported

photo says 35–55 g carbs
→ modelEstimated

user taps "looks right" at 45 g
→ userConfirmed

calculated IOB
→ derived
```

This prevents future models from treating an AI guess as equal to a user-confirmed treatment.

### Missing is not zero

Never collapse these states:

```text
no insulin record found
no exercise record found
no carb quantity supplied
HealthKit returned no workout
```

into:

```text
insulin = 0
exercise = false
carbs = 0
```

They should remain **unknown / not observed** unless the user explicitly states the fact.

---

## 6. Manual exercise is first-class and should survive later HealthKit enrichment

The app cannot assume workouts will be logged on Apple Watch.

Manual exercise should therefore work independently:

```text
ManualExerciseEvent
- occurredAt
- endedAt? / duration?
- kind?        // optional
- intensity?   // optional
- notes?       // optional, not required
- source surface
```

The lowest-friction action may initially record simply:

```text
exercise happened around now
```

and allow optional enrichment later.

### HealthKit should enrich, not replace

If a HealthKit workout later appears that overlaps a manual exercise event:

```text
manual event: 18:05 "exercise"
HealthKit: 18:00–18:46 outdoor run, 420 active kcal
```

we should reconcile/link them rather than create two independent physiological exercise episodes.

Recommended model:

```text
ExerciseEpisode
- id
- userReportedEventID?
- healthKitWorkoutID?
- canonical start/end
- normalized kind
- normalized intensity
- provenance/confidence per field
```

Do not delete the original manual event; it tells us that the user intentionally supplied context and when the app learned it.

### Dedupe/reconciliation rule

Start conservatively:

- overlap in time;
- plausible duration;
- compatible exercise type where known;
- never silently merge clearly distinct workouts.

If ambiguous, retain both rather than invent certainty.

---

## 7. Meals should preserve the entire estimate → confirmation path

A meal is not one number.

Recommended structure:

```text
MealEvent
- Ate occurredAt
- photo asset reference?
- user text/description?
- linked carb TreatmentEntry?

MealEstimate
- mealEventID
- createdAt
- provider/model/version
- food components
- portion assumptions
- estimated carb low/high
- confidence / uncertainty reason

MealConfirmation
- mealEventID
- confirmed carb amount/range?
- confirmed/edited components?
- timestamp
- source surface
```

If confirmed carbs become a xDrip carb `TreatmentEntry`, link that treatment ID back to the meal.

### Keep the original estimate after correction

Suppose:

```text
AI estimate: 70–90 g
user corrects: 48 g
```

Do not overwrite the AI result with 48 g.

That correction is extremely valuable future supervision:

```text
model thought 70–90
user said 48
```

Later we can evaluate provider quality and personalize repeated meals.

The photo asset should have its own retention lifecycle so a user can delete old images without destroying the structured meal history.

---

## 8. Insulin/treatment edits should be observable corrections where practical

Existing `TreatmentEntry` is mutable because users can edit/delete treatments.

The canonical current value should stay there.

But when treatment changes happen through our shared domain action layer, record a lightweight correction event:

```text
TreatmentCorrectionEvent
- linkedTreatmentID
- occurredAt / recordedAt
- field(s) changed
- prior value?
- corrected value?
- source
```

Why useful:

- identifies logging mistakes;
- prevents treating known bad historical values as high-confidence supervision;
- lets us evaluate whether quick-entry surfaces cause errors;
- supports debugging an alert that was based on an earlier value.

This does not need to become a full general-purpose audit database for every imported Nightscout mutation in MVP.

---

## 9. Attention Episode is persistent state; user events are its history

Keep the episode model small.

Illustrative:

```text
AttentionEpisode
- id
- type
- openedAt
- lastEvaluatedAt
- currentState
- deferUntil?
- resolvedAt?
- resolutionReason?
- pendingNotificationID?
```

Possible state:

```text
open
acknowledged
waitingForRecovery
deferred
resolved
```

Do **not** stuff every historic action into the episode row.

Instead:

```text
Episode
  ↳ AttentionUserEvent
  ↳ AttentionDecisionEvent
  ↳ treatment links
  ↳ meal/exercise links
```

This preserves both current state and full history.

---

## 10. Record decision-changing Attention events, not every computed feature

Every CGM reading may trigger an engine evaluation.

We do not necessarily need to persist a giant snapshot after every evaluation because historical replay can recompute decisions from raw input.

Persist an `AttentionDecisionEvent` when something meaningful changes, for example:

```text
episode opened
urgency changed
notification scheduled
notification cancelled
notification re-scheduled
user-facing alert selected
episode resolved
```

Suggested fields:

```text
AttentionDecisionEvent
- id
- episodeID
- evaluatedAt
- trigger
  - glucose
  - treatment
  - userAction
  - healthContext
  - appReconciliation
  - deferDeadline
- decision
  - quiet
  - remind
  - escalate
  - resolve
- reasonCodes[]
- attentionPolicyRevisionID
- appBuild/version
```

Reason codes should be structured, e.g.:

```text
persistentRise
recentInsulin
activeInsulinEarlyPhase
recentMeal
recentExercise
postLowRecovery
staleGlucose
userDeferred
```

This gives explainability without serializing the entire object graph on every reading.

---

## 11. Notifications need their own lifecycle events

The system should distinguish what we *know* happened.

Useful events:

```text
notificationScheduled
notificationCancelled
notificationActionReceived
notificationOpened
```

Do not automatically infer:

```text
notification was seen
notification was read
user ignored notification
```

because iOS does not always give the app enough evidence to make those claims.

Important fields:

```text
notificationID
episodeID
category/context
scheduledAt
intendedFireAt?
actionType?
actionValue?
interactionSurface?  // phone/watch where determinable
```

For personalization, the highest-quality signal is usually the explicit user action, not whether a banner was probably displayed.

---

## 12. Preserve policy/model revisions

This is easy to miss and matters enormously for retrospective learning.

Imagine the user gets an alert in January, then in March we change:

- rise persistence threshold;
- alert cooldown;
- IOB action duration;
- insulin peak;
- exercise lookback;
- stale-data threshold.

If historical decision rows simply say:

```text
policyVersion = "current"
```

we can no longer accurately explain January's behavior.

Recommended immutable revision objects:

```text
AttentionPolicyRevision
- id
- createdAt
- engineVersion
- configurationVersion
- relevant config payload/hash

InsulinModelRevision
- id
- model family
- activity duration
- peak
- effectiveFrom
```

Each persisted decision references the revision used at that moment.

Meal AI outputs similarly need:

```text
provider
model
model version where supplied
prompt/schema version
```

External inference is not guaranteed to be reproducible later.

---

## 13. Store local timezone context for circadian personalization

Time-of-day may become a meaningful personal feature.

UTC timestamp alone is insufficient when travelling.

For key user/context events, retain the local timezone identifier or offset applicable at occurrence.

Example:

```text
2026-11-15 08:10 breakfast in New York
```

should remain a breakfast-time event even if replay later happens in London.

Do not store GPS location merely to solve this problem. Timezone is enough.

---

## 14. Future personalization should consume derived feature snapshots, not own the raw store

When we later build personal prediction, construct a feature vector at a historical timestamp from the durable facts.

Example:

```text
FeatureSnapshot(t)
- glucose
- short delta
- medium delta
- trend persistence
- acceleration
- minutes since Ate
- confirmed/estimated carbs if known
- minutes since insulin
- modeled IOB
- insulin activity phase
- recent exercise kind / minutes since
- low-recovery state
- time of day / weekday
- later: sleep vs baseline
```

These are **views of history**, not canonical records.

Most should be recomputed on demand or in an analysis cache.

Why:

- models will change;
- features will change;
- IOB assumptions will change;
- exercise windows will change;
- old values can be recomputed consistently under new candidate policies.

This is essential for fair backtesting.

---

## 15. What to record from day one

### P0 — must-have durable data

**Existing xDrip**

- glucose readings and source/time;
- insulin amount/time;
- carb amount/time when entered;
- treatment edits/deletions via existing model;
- manual xDrip exercise treatment if used.

**New Attention layer**

- every `Ate` event;
- manual exercise events/context;
- `no insulin needed`;
- `waiting for recovery`;
- defer/remind action + deadline;
- other explicit acknowledgement actions;
- occurredAt + recordedAt;
- action source/surface;
- episode linkage;
- Attention Episode open/change/resolve;
- decision-changing engine events;
- reason codes;
- notification scheduling/cancellation/actions;
- policy/config revision;
- app/engine version.

**Imported/enriched context**

- normalized HealthKit workout summary when available;
- HealthKit origin UUID/source provenance;
- meal AI estimate with provider/model/version;
- user confirmation/correction;
- links to resulting TreatmentEntry.

### P1 — useful after first prototype

- optional exercise kind/intensity;
- workout active energy and average/max HR;
- optional meal components / repeated-meal identity;
- optional episode-quality feedback;
- source-level data quality flags;
- correction history for quick insulin/carb logging.

### P2 — collect only when we actually use it

- sleep duration/regularity;
- resting-HR/HRV longitudinal context;
- coarse passive movement;
- richer meal embeddings;
- other recovery/illness signals.

---

## 16. What *not* to store as canonical data

Avoid treating these as permanent facts:

- calculated IOB at every glucose reading;
- COB at every glucose reading;
- a generic “insulin sensitivity score”;
- “exercise effect = -20%”;
- hard-coded meal impact classes;
- predicted glucose values unless they were actually shown/used and need audit;
- daily TIR aggregates;
- every derived delta/acceleration;
- duplicated HealthKit glucose;
- every raw heart-rate sample;
- every sleep-stage sample;
- every model feature vector.

All can be recomputed or added later from more fundamental history if needed.

Exception:

**Persist an inference/derived output if it materially affected a user-facing decision and cannot be guaranteed reproducible.**

Examples:

- meal AI result shown to user;
- future prediction that caused an alert;
- external model output used by an Attention decision.

Store its provider/model/config provenance.

---

## 17. Data quality should travel with the data

Candidate engine inputs should be able to distinguish:

```text
fresh glucose
stale glucose
backfilled glucose
estimated meal
confirmed meal
manual exercise
HealthKit exercise
possibly duplicated exercise
insulin log later corrected
```

Do not let an adapter erase those distinctions before the engine/personalization layer can use them.

Possible common quality structure:

```text
DataQuality
- freshness
- completeness
- certainty
- originalSource
- wasBackfilled
- wasCorrected
```

This does not mean the MVP needs a complex numeric confidence score.

Prefer discrete, explainable states first.

---

## 18. User behavior itself is valuable supervision

We do not need to constantly ask:

> Was that alert useful?

Structured actions already provide labels.

Examples:

```text
alert → log insulin
```

strong evidence that attention led to recorded action.

```text
alert → no insulin needed
```

evidence the situation was intentionally reviewed.

```text
alert → waiting for recovery
```

important contextual label.

```text
reminder → glucose resolves before next alert
```

useful policy outcome.

```text
AI carbs 75 g → user changes to 45 g
```

supervision for meal estimation.

```text
manual exercise logged after alert
```

potential evidence that the system was missing context.

Later, selectively asking the user to review 30–50 representative historical episodes may provide higher-quality labels than interrupting every episode with feedback prompts.

---

## 19. Versioned export is part of the data model

Future personalization and replay should not depend on direct access to a live Core Data database.

Define a versioned private export representation for the new Attention data.

Illustrative:

```json
{
  "schemaVersion": 1,
  "exportedAt": "...",
  "events": [...],
  "episodes": [...],
  "decisionEvents": [...],
  "policyRevisions": [...]
}
```

This should be included in V7 backup/export later.

Benefits:

- migration safety;
- local analysis;
- historical replay;
- portability if xDrip architecture changes;
- ability to inspect/debug without putting raw health data in Git.

Raw personal exports remain private and outside the repository.

---

## 20. Privacy / minimization

The future model does **not** require collecting everything Apple Watch can observe.

The strongest personalization dataset is likely to come from:

- glucose;
- treatments;
- meal occurrence and confirmed estimates;
- exercise occurrence/type/timing;
- explicit Attention interactions;
- outcome trajectory;
- later selected longitudinal context.

Avoid collecting data merely because it is accessible.

Specific recommendations:

- no raw location history;
- no always-on raw HR archive in our own database;
- no duplicated HealthKit CGM feed;
- no raw provider payload retention unless needed;
- photos separate from structured records and deletable;
- credentials remain in Keychain;
- raw exports stay out of Git.

---

## Reuse / adapt / build / ignore

### Reuse

- xDrip `BgReading` as canonical glucose history;
- xDrip `TreatmentEntry` as canonical standard treatment history;
- Nightscout identifiers/history for interoperable treatment linkage;
- HealthKit object UUID/source provenance for workout imports;
- V7 backup architecture later for Attention export.

### Adapt

- existing treatment logging so shared domain actions can emit correction/linkage events;
- HealthKit workout import into a normalized exercise episode;
- notification response path to emit structured Attention user events.

### Build

- separate `AttentionStore`;
- common event envelope;
- `AttentionUserEvent`;
- `ExerciseEpisode` reconciliation model;
- `MealEvent` / estimate / confirmation linkage;
- persistent `AttentionEpisode`;
- `AttentionDecisionEvent`;
- immutable policy/model revisions;
- versioned private export.

### Ignore / defer

- duplicated glucose from HealthKit;
- raw Watch sensor archive;
- broad “collect everything for ML” approach;
- storing every derived feature permanently;
- complex numerical confidence scoring before simple states prove insufficient.

---

## Open questions

1. Should the separate Attention store use a dedicated Core Data persistent container or a smaller explicit SQLite layer? Prefer minimum dependency/merge cost; decide in product/technical spec.
2. Exactly how much treatment correction history is worth recording beyond edits made through our own action layer?
3. How should manual + HealthKit exercise reconciliation be exposed if the automatic match is ambiguous?
4. Do we retain meal photos indefinitely for personal repeated-meal retrieval, or make retention configurable?
5. How much decision trace should be retained on-device before pruning/compaction? The volume is small enough that this may not matter initially.
6. Which settings need full revision history versus being embedded in an Attention policy configuration snapshot?
7. Should `no insulin needed` expire automatically as an acknowledgement while preserving the historical event? Likely yes; safety pass should define the lifecycle.

---

## Product-spec implications

1. **Existing xDrip glucose/treatment stores remain canonical.**
2. **Manual exercise is first-class**, not a fallback after HealthKit fails.
3. HealthKit workouts enrich/reconcile manual exercise and can also stand alone when logged automatically.
4. HealthKit glucose read-back is not required for the first product.
5. Add a separate local `AttentionStore` rather than overloading `TreatmentEntry`.
6. Every Attention event records both `occurredAt` and `recordedAt`.
7. Every event records structured provenance/source.
8. Estimated, confirmed, observed, reported and derived data remain distinguishable.
9. Missing data remains unknown; never silently convert it to zero/false.
10. Keep original AI estimates when a user corrects them.
11. Link confirmed carbs to the resulting xDrip treatment rather than duplicating treatment truth.
12. Persist episode state separately from its event history.
13. Persist decision-changing events/reasons, not every derived feature on every reading.
14. Version Attention policy, insulin model and external model configuration so old behavior can be reconstructed.
15. Retain timezone-at-occurrence for key events without collecting location.
16. Build personalization features from the event history and recompute them under new models.
17. Versioned private export/backup is required before meaningful long-term data collection.
18. Raw personal health history, exports and media remain out of Git.

---

## Bottom line

The future personalized system should be built on an **honest history of what happened and what the app knew at the time**, not on a pile of fixed assumptions.

The durable data layer therefore needs to answer:

```text
What happened?
When did it actually happen?
When did the app learn about it?
Where did the information come from?
Was it observed, user-reported, estimated or confirmed?
Was it later corrected?
Which episode/decision did it affect?
Which policy/model version was in force?
```

If we preserve those facts from day one, future glucose-response modeling can change dramatically without requiring us to redesign the history underneath it.

The practical principle is:

> **Store facts, uncertainty and provenance; derive intelligence later.**

---

## Primary sources / evidence

- Apple HealthKit `HKObject.sourceRevision` / `HKSourceRevision`: provenance of the app/device and version that created HealthKit data.
- Apple HealthKit `HKMetadataKeySyncIdentifier` and `HKMetadataKeySyncVersion`: stable identifier + revision semantics for synchronized objects.
- Apple HealthKit Metadata Keys: supports application-specific metadata where appropriate.
- Nightscout API v3: stable collection/history mechanisms for treatment data and compatibility with existing treatment records.
- Current V7 xDrip research: existing `BgReading`/`TreatmentEntry`, HealthKit glucose-export-only manager, treatment editor and backup/data-management architecture.
