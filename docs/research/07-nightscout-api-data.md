# 07 — Nightscout API + data storage

Status: **Complete**  
Research date: 2026-08-16

## Question

Given that the user already has Nightscout, how much of the data needed by the future Attention Engine can Nightscout store and retrieve reliably? In particular, can we persist meals, insulin, exercise, notes and future context alongside historical CGM data, and where should app-specific states such as `Ate`, `No insulin needed`, `Waiting for recovery`, acknowledgements and Attention Episode lifecycle live?

## Executive conclusion

**Use Nightscout as the durable interoperability/history layer for standard diabetes events, but not as the sole canonical store for the Attention Engine's private state.**

Nightscout is more capable than a glucose-only archive:

- CGM readings live in `entries`;
- insulin, carbs and other treatment events live in `treatments`;
- treatment documents support standard fields including `eventType`, carbs, protein, fat, insulin, notes and `enteredBy`;
- `food` is a first-class collection for reusable food/nutrition definitions;
- `activity` is a first-class Mongo collection with a v1 REST API;
- the current activity and food write paths preserve the submitted document rather than projecting it through a narrow fixed schema;
- current treatment storage explicitly preserves client fields outside the small set it normalises;
- API v3 provides identifiers, deduplication, filtering, paging and incremental history for its supported collections.

However, Nightscout is not a good single source of truth for every internal product event:

- API v3's generic collection set is limited to `devicestatus`, `entries`, `food`, `profile`, `settings` and `treatments`; `activity` currently remains a separate v1 API;
- arbitrary extra fields often survive today, but those fields are not a stable cross-client contract;
- there is no supported arbitrary `attention_events` collection exposed through the standard API;
- remote availability must not sit on the real-time alert correctness path;
- app-internal events would pollute Nightscout treatment history if every acknowledgement, defer and episode transition were represented as a treatment.

Recommended ownership:

```text
existing xDrip Core Data
├── BgReading                  ← canonical local glucose history
├── TreatmentEntry             ← canonical local insulin/carbs/manual exercise
│
├── AttentionEvent             ← NEW: Ate, defer, recovery wait, acknowledgement, etc.
└── AttentionEpisode           ← NEW: persistent episode lifecycle/state
          │
          ├── local Attention Engine reads synchronously/offline
          │
          └── selected interoperable events are mirrored to Nightscout
                    ↓
Nightscout
├── entries                    ← CGM history / replay source
├── treatments                 ← insulin, carbs, notes, standard treatment events
├── food                       ← reusable food/nutrition definitions
└── activity                   ← optional exercise/activity summaries
```

The guiding rule is:

> **Nightscout should make our data more durable and interoperable; it should not become a network dependency for deciding whether diabetes needs attention right now.**

---

## 1. What Nightscout stores today

Current Nightscout uses MongoDB and exposes several distinct collections. Its documented core environment variables include collections for:

- `entries`
- `treatments`
- `devicestatus`
- `profile`
- `food`
- `activity`

API v3 exposes generic CRUD/search/history operations for:

```text
devicestatus
entries
food
profile
settings
treatments
```

`activity` is real and current, but is not in that API v3 collection enum; it has a separate v1-style REST endpoint.

This distinction matters because “Nightscout can physically store it” is not the same as “Nightscout defines a stable modern API contract for it.”

---

## 2. CGM readings — excellent fit

Nightscout's `entries` collection remains the obvious historical source for glucose replay.

For our purposes we care about:

- glucose value;
- timestamp;
- trend/direction where supplied;
- noise/data-quality fields where available;
- stable identity/deduplication metadata.

The v1 API exposes `/api/v1/entries` and supports Mongo-style date/query filtering. The project README explicitly documents larger historical queries using `count`, `date`, `dateString` and related filters.

API v3 adds a better general synchronization model:

- filters;
- `limit` and `skip` paging;
- ascending/descending sort;
- selective returned fields;
- stable `identifier` handling;
- collection history endpoints for incremental synchronization.

For a first 1–3 month historical replay, Nightscout is therefore a very good source.

### Practical extraction note

Do not copy the old prediction backtest's “request N days × ~300 readings in one call” approach as our permanent importer.

At 5-minute cadence:

```text
30 days  ≈ 8,640 readings
90 days  ≈ 25,920 readings
```

API v3 commonly caps a single page at about 1,000 documents (`API3_MAX_LIMIT` is configurable). Our importer should page chronologically and checkpoint progress.

---

## 3. Treatments — the main standard event stream

Nightscout treatment documents are flexible and are the best place for standard diabetes actions.

The documented treatment schema includes:

```text
eventType
timestamp / created_at
carbs
protein
fat
insulin
glucose / glucoseType
notes
enteredBy
```

Current server code also handles fields used by AID clients such as duration, percent, absolute values, identifiers and other client-specific metadata.

The current treatment storage implementation is particularly relevant: it normalises known numeric/date fields but deliberately preserves other client fields. A source-code comment added for UUID compatibility says that fields such as `syncIdentifier`, `uuid` and other client fields are preserved as-is.

That means treatment records are currently flexible enough for modest namespaced metadata if we need it.

### Good Nightscout treatment uses for this project

- insulin taken;
- carbs eaten when the user actually records a carb amount;
- protein/fat if a future meal estimator produces them and the user chooses to retain them;
- manual exercise treatment where we want to stay compatible with existing xDrip treatment semantics;
- notes useful to the user outside this app.

### Bad uses

Do not turn every internal Attention state into a treatment merely because the database accepts arbitrary JSON.

Examples that should remain app-domain state by default:

- notification shown;
- user acknowledged alert;
- `Remind me in 10 minutes`;
- reminder deadline cancelled;
- episode escalated;
- episode resolved;
- policy reason code;
- candidate confidence;
- policy/model version.

Those are application events, not treatments.

---

## 4. Food collection — useful, but it means “food definition”, not “I ate”

Nightscout API v3 defines a `food` schema containing nutritional information such as:

```text
name
portion
unit
carbs
fat
protein
energy
gi
```

Nightscout also supports quick-pick/compound-food concepts.

This can be useful later for:

- reusable known meals;
- common portions;
- confirmed nutrition estimates;
- matching repeated meals over time.

But the semantic distinction is crucial:

```text
Food record       = “this food/portion exists and has these properties”
Meal event        = “I ate something at this time”
```

A food record alone does not tell the Attention Engine that a meal just happened.

For the user flow, the immediate one-tap `Ate` action should therefore create an **AttentionEvent** immediately. If a carb/nutrition estimate becomes available later, it can be linked to or produce a standard TreatmentEntry/Nightscout treatment.

Do not force a carb value simply to make the meal fit Nightscout. “Ate, carbs unknown” is real information and should remain representable.

---

## 5. Activity collection — real and flexible, but API maturity is uneven

Nightscout currently has:

- a configurable `MONGO_ACTIVITY_COLLECTION`;
- `/api/v1/activity` read/write/delete/update routes;
- dedicated `api:activity:*` permissions;
- an indexed `created_at` field;
- current batch-write support.

The storage implementation is unusually permissive: it adds `created_at` when missing and then writes the submitted activity document as the Mongo replacement document. It does not impose a narrow activity schema before storage.

That makes it capable of storing exercise summaries and future contextual fields.

### Important limitation

`activity` is **not** part of API v3's generic collection enum today.

That makes it less attractive as the only long-term schema for Apple Health/Watch context. We should treat it as an optional interoperable mirror rather than forcing every HealthKit sample into it.

### Recommended exercise split

```text
Manual “I exercised” / exercise treatment
    → existing xDrip TreatmentEntry
    → Nightscout treatment sync where compatible

Detailed workout/activity context
    → HealthKit remains source
    → local context adapter / derived summary
    → optionally mirror useful summaries to Nightscout activity
```

The Attention Engine probably needs simple derived context such as:

- exercise occurred recently;
- start/end time;
- duration;
- broad type/intensity;
- perhaps recent HR/activity magnitude later.

It does not need every raw heart-rate or motion sample copied into Nightscout.

---

## 6. Can we store arbitrary custom fields?

### Short answer

**Current server implementations often preserve them, but we should not treat arbitrary fields as a guaranteed public contract.**

Evidence from current Nightscout source:

- `activity` writes the supplied document wholesale after adding `created_at` where needed;
- `food` writes the supplied document wholesale;
- treatment storage normalises selected known fields and explicitly preserves other client fields;
- API v3's OpenAPI models do not declare an arbitrary custom collection mechanism.

This is useful flexibility, but there are several failure modes if we make custom fields essential:

1. another Nightscout client may GET → edit → PUT a record without preserving fields it does not understand;
2. future Nightscout validation may become stricter;
3. API v1/v3 paths do not expose exactly the same collections;
4. reports and third-party tools will not understand our custom semantics;
5. a hosting provider may run a different Nightscout version or retention configuration.

### If we mirror custom metadata

Use a namespace and version it, for example conceptually:

```json
{
  "identifier": "attentionapp-<uuid>",
  "eventType": "Note",
  "created_at": "...",
  "enteredBy": "attention-app",
  "notes": "Meal logged",
  "attentionApp": {
    "schemaVersion": 1,
    "eventId": "...",
    "type": "ate"
  }
}
```

But the rule remains:

> Losing `attentionApp` from a remote record must never make the app unable to reconstruct or safely run its local episode state.

For that reason I would **not mirror every AttentionEvent to Nightscout by default**. Mirror only events whose presence there is actually useful.

---

## 7. No supported arbitrary custom collection

It would be tempting to create:

```text
attention_events
attention_episodes
```

inside the same Mongo database.

Nightscout's standard API does not currently expose arbitrary collection names. API v3 explicitly enumerates its supported collections, and activity itself needs a dedicated route.

We could fork Nightscout or access Mongo directly to add our own collection, but that creates exactly the coupling we are trying to avoid:

- custom server deployment;
- migration burden;
- hosting-provider incompatibility;
- another codebase to maintain;
- additional security surface.

**Do not do this for the personal app.**

If we ever need true remote app-specific storage beyond Nightscout mirrors/backups, evaluate a separate small sync layer at that point rather than silently turning Nightscout into our custom backend.

---

## 8. Recommended local data model

We should reuse xDrip's existing Core Data wherever the concept already exists and add only the missing domain records.

### Existing source-of-truth entities

```text
BgReading
    current/historical glucose

TreatmentEntry
    insulin
    carbs
    exercise
    BG check
    note
```

Do not create duplicate insulin or carb tables.

### New AttentionEvent

Conceptually:

```swift
AttentionEvent
- id: UUID
- occurredAt: Date
- recordedAt: Date
- type: AttentionEventType
- episodeId: UUID?
- relatedTreatmentId: String?
- source: ActionSource
- payloadVersion: Int
- payload: typed fields as required
```

Possible event types:

```text
ate
noInsulinNeeded
waitingForRecovery
remindLater
acknowledged
mealEstimateAdded
exerciseContextAdded
```

The event log should represent things the **user did or told the app**, not every internal evaluation.

### New AttentionEpisode

Conceptually:

```swift
AttentionEpisode
- id: UUID
- startedAt: Date
- resolvedAt: Date?
- status
- trigger / reason
- lastEvaluatedAt
- currentDeferDeadline
- linked user events
- policyVersion
```

A separate decision trace can be retained selectively for research/debug builds rather than turning every evaluation into permanent user data.

### Why local first

- available synchronously when a CGM callback arrives;
- works offline;
- no network race before suppressing/escalating an alert;
- easy deterministic unit/replay tests;
- natural fit with xDrip's existing persistence;
- can later be included in V7 backup/export.

---

## 9. Mapping the product concepts

| Product concept | Local canonical record | Nightscout representation | Recommendation |
|---|---|---|---|
| CGM reading | existing `BgReading` | `entries` | Reuse both |
| Insulin taken | existing `TreatmentEntry` | `treatments.insulin` | Reuse/sync |
| Carbs entered | existing `TreatmentEntry` | `treatments.carbs` | Reuse/sync |
| Protein/fat | treatment metadata if added | treatment fields | Optional later |
| Reusable known food | optional local/cache | `food` | Useful later |
| `Ate`, carbs unknown | `AttentionEvent.ate` | optional note/custom mirror | **Local first** |
| Manual exercise | existing `TreatmentEntry` + optional event link | treatment and/or activity | Reuse |
| Detailed workout | HealthKit + derived local context | optional `activity` summary | Do not copy raw HealthKit wholesale |
| `No insulin needed` | `AttentionEvent` | usually none | Local only |
| `Waiting for recovery` | `AttentionEvent` + episode state | usually none | Local only |
| `Remind me later` | `AttentionEvent` + persisted deadline | none | Local only |
| Alert acknowledgement | `AttentionEvent` | none | Local only |
| Attention Episode | `AttentionEpisode` | none | Local only |
| Policy decision/reason | transient/debug trace | none | Local/debug |
| Meal photo | file + local metadata/reference | no image payload; optional text/ref only | Keep outside Nightscout documents |
| Photo carb estimate | local event + confirmed treatment if accepted | confirmed treatment/note | Sync confirmed user data |
| Watch/Health context | HealthKit + derived context | optional activity summary | Local/derived |

---

## 10. Meal-photo storage

Do not put image binaries/base64 into Nightscout treatment or activity documents.

Reasons:

- Mongo document size/bandwidth waste;
- poor interoperability;
- backups become unnecessarily large;
- image lifecycle differs from event lifecycle;
- Nightscout is not an object/image store.

A future meal photo should instead have:

```text
local photo file / managed media asset
        ↓
MealPhotoMetadata
- id
- capturedAt
- linked AttentionEvent / TreatmentEntry
- model/provider used
- candidate foods
- carb range
- confidence
- user-confirmed result
```

If we later need cross-device photo persistence, choose an actual file/object-sync mechanism then. Nightscout can retain a reference or human-readable note, not the photo itself.

---

## 11. API authentication and permissions

Nightscout supports:

- `API_SECRET` with effectively full administrative access;
- access tokens/subjects with roles;
- bearer JWT derived from access-token authorization;
- query-token compatibility for clients that use it.

Current default roles include:

```text
admin                 → full access
readable              → read access
careportal            → create treatments
devicestatus-upload   → create device status
activity              → create activity
```

Nightscout's user documentation explicitly warns that knowing the API secret gives full access and recommends role-based tokens when access needs to be controlled.

### Recommendation for our app

**Do not embed or routinely use the user's API_SECRET.**

Create a dedicated app token/subject with the minimum permissions we actually require and store it in iOS Keychain.

Likely eventual needs:

```text
read entries
read/create/update treatments
read activity
create activity       (only if we choose to mirror workouts)
read food             (only if we use it)
create/update food    (only if we build reusable meal definitions)
```

For initial replay, a read-only token is enough.

The exact permission set can be finalised against the user's deployed Nightscout version when we implement the connector.

---

## 12. Privacy consequence of storing richer context

Today the user's Nightscout contains health data already, but adding:

- meal descriptions;
- exercise;
- notes;
- behavioural acknowledgements;
- perhaps future contextual metadata

makes the dataset richer and more personally revealing.

Before the app starts uploading new context, confirm the Nightscout deployment is not accidentally world-readable.

Nightscout documentation notes that `AUTH_DEFAULT_ROLES=readable` permits anyone who knows the URL to view the site and recommends securing the site where appropriate.

For this project:

- prefer `AUTH_DEFAULT_ROLES=denied` or otherwise secured access once we rely on richer records;
- use HTTPS;
- use a dedicated least-privilege token;
- store secrets in Keychain;
- never put the Nightscout URL/token/API secret into Git;
- never commit raw CGM/treatment exports to Git.

This continues the privacy rule from pass 06.

---

## 13. Retention is configurable — do not assume “Nightscout = forever”

API v3 supports per-collection autopruning. Its documentation gives examples such as retaining entries for 365 days and treatments for 120 days; the documented default is only automatic pruning for device status, but deployments can change this.

Therefore the connector should inspect/measure actual available history rather than assuming a fixed retention period.

For replay reports, record:

```text
first available glucose date
first available treatment date
first available activity date
number/duration of data gaps
```

For app-specific AttentionEvents, local retention/backup should be explicit and under our control.

---

## 14. Historical replay with Nightscout

The user's existing Nightscout materially simplifies pass 06.

Recommended first replay pipeline:

```text
Nightscout entries ──────┐
                         │
Nightscout treatments ───┼─→ normalisation → ReplayEvent stream
                         │
(optional activity) ─────┘

future local AttentionEvent export ────────────┘
```

### Phase 1 — existing historical data

Use what already exists:

- entries;
- treatments;
- possibly historical exercise if it was logged as treatment/activity.

This lets us evaluate glucose trajectory and treatment-aware policies now.

### Phase 2 — prospective richer logging

Once our app is running, also record:

- Ate;
- intentional no-insulin decision;
- waiting-for-recovery state;
- reminder/defer action;
- user acknowledgement;
- derived exercise context.

Future replay can then answer the much more valuable question:

> Given what the user actually knew/did at the time, did the Attention Engine interrupt at the right moment?

---

## 15. API v1 vs API v3 recommendation

Do not force one version globally.

### Prefer API v3 where it is mature

Especially for replay/sync of:

- entries;
- treatments;
- food if used.

Reasons:

- explicit stable identifiers;
- deduplication;
- paging;
- field filters;
- incremental history endpoints;
- better concurrency/update semantics.

### Use v1 where Nightscout currently requires it

Most notably:

- `activity`.

Also retain v1 compatibility because xDrip and existing Nightscout deployments may already use it heavily.

Build the Nightscout connector behind protocols so API-version details do not leak into the Attention Engine.

Conceptually:

```swift
protocol GlucoseHistorySource { ... }
protocol TreatmentHistorySource { ... }
protocol ActivityHistorySource { ... }

NightscoutV3HistoryClient
NightscoutV1ActivityClient
```

The engine should see normalised domain events, not REST endpoints.

---

## 16. Idempotent uploads

If our app uploads anything to Nightscout, give each syncable local event a stable UUID/identifier.

API v3's `identifier` field is specifically designed for document identity and deduplication. Recent Nightscout releases have also improved UUID handling for Loop/Trio/AID clients.

This is important because iOS background execution and network retry behaviour can cause the same write to be attempted more than once.

Desired property:

```text
save locally
→ enqueue NS mirror
→ upload succeeds but response is lost
→ retry later with same identifier
→ Nightscout updates/deduplicates instead of adding a second meal/insulin record
```

Local save should always happen first; Nightscout sync is asynchronous.

---

## 17. Reuse / adapt / build / avoid

| Area | Decision |
|---|---|
| Nightscout CGM `entries` | **Reuse** as historical/replay source |
| Nightscout `treatments` | **Reuse** for insulin/carbs/standard treatment events |
| Existing xDrip TreatmentEntry sync | **Reuse/adapt**; keep standard data source of truth |
| Nightscout `food` | **Optional reuse later** for reusable foods/confirmed nutrition |
| Nightscout `activity` | **Optional adapt** for exercise summaries; do not make core dependency |
| API v3 history/paging | **Reuse** for robust history synchronization |
| API v3 identifiers | **Reuse** for idempotent mirrors |
| App-specific custom treatment fields | **Mirror only**, namespaced/versioned |
| AttentionEvent | **Build locally** |
| AttentionEpisode | **Build locally** |
| Custom Nightscout Mongo collection | **Avoid** |
| Direct MongoDB access from iOS app | **Avoid** |
| Raw meal images in Nightscout | **Avoid** |
| API_SECRET embedded in app | **Avoid** |
| Dedicated least-privilege token + Keychain | **Build/configure** |

---

## 18. Product-spec implications

1. **Nightscout becomes an explicit integration in the product spec**, not merely a future export target.

2. **Existing Nightscout history is the preferred initial replay dataset** for this user.

3. **Standard diabetes events remain standard records.** Glucose uses `BgReading`/Nightscout entries; insulin/carbs use TreatmentEntry/Nightscout treatments.

4. **`Ate` is not equivalent to `carbs = 0`.** It requires its own local event because the amount may be unknown and insulin may intentionally be deferred.

5. **Attention Episode state is local-first.** Alert correctness must never wait for a Nightscout read/write.

6. **Nightscout synchronization is asynchronous and idempotent.** Stable event identifiers and retry-safe writes are required.

7. **Exercise has two levels:** simple user-recorded exercise can reuse treatment semantics; rich HealthKit activity remains local/derived with optional Nightscout activity summaries.

8. **Food definitions and meal occurrences are separate concepts.** A reusable food record cannot replace a meal event.

9. **Meal photos are files, not Nightscout documents.** Only metadata/confirmed nutrition may be mirrored.

10. **V7 backup/export should eventually be extended to include AttentionEvent/AttentionEpisode data** because local app-specific state must be portable even if Nightscout mirrors are incomplete.

11. **Credentials live in Keychain** and a dedicated Nightscout token should be used instead of embedding the full API secret.

12. **Security of the user's Nightscout deployment should be checked before richer context is uploaded.**

13. **Raw personal data remains outside Git.** Only synthetic/redacted replay fixtures and aggregate results belong in the repository.

---

## 19. Open questions for implementation/spec phase

These do not block the research conclusion:

1. Which Nightscout version/hosting provider is the user's instance currently running?
2. Is `AUTH_DEFAULT_ROLES` currently `readable` or secured?
3. How much historical entry/treatment data is actually retained?
4. How complete is historical insulin logging in the user's Nightscout?
5. Has exercise historically been logged as treatments, activity records, or not at all?
6. Does the user's current xDrip/Zukka sync path already write all TreatmentEntry types we care about?
7. Should `Ate` ever be mirrored to Nightscout for human visibility, or remain local unless carbs/nutrition are added?
8. How much Attention decision trace should be retained after debugging/research versus discarded?
9. Should local Attention data be included in standard V7 backup immediately, or first via a development-only export?

---

## 20. Sources inspected

Primary sources used in this pass:

- Nightscout `cgm-remote-monitor` README / API and environment documentation
- `lib/server/swagger.yaml` — API v1 OpenAPI
- `lib/api3/swagger.yaml` — API v3 OpenAPI
- `lib/api/activity/index.js` — activity REST route
- `lib/server/activity.js` — activity persistence/query behavior
- `lib/api/food/index.js` — food REST route
- `lib/server/food.js` — food persistence behavior
- `lib/server/treatments.js` — treatment normalization, custom-field preservation and identifiers
- `lib/authorization/index.js` — token/JWT/API-secret resolution
- `lib/authorization/storage.js` — default roles/permissions
- Nightscout Admin Tools and configuration documentation
- current Nightscout release notes, including recent UUID/deduplication and activity/food API work
- xDrip V7 Data Management/Nightscout import findings from passes 02 and 06

## Final decision

**Yes, Nightscout should be a major part of the data architecture — but as the interoperable history/sync layer, not the brain of the app.**

The clean split is:

```text
standard diabetes fact
    → existing xDrip entity
    → Nightscout sync

user/app attention fact
    → AttentionEvent / AttentionEpisode locally
    → optional selective Nightscout mirror

large media / rich sensor context
    → proper local/file/HealthKit source
    → derived summaries only where useful
```

This gives us the best of both worlds: the user's existing Nightscout becomes immediately useful for historical replay and durable standard records, while the Attention Engine stays reliable, testable and free to evolve without corrupting or overloading Nightscout semantics.
