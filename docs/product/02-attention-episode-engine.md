# Product Spec — Chunk 2: Attention Episode + Engine

**Status:** Locked  
**Parent spec:** `PRODUCT_SPEC.md`

This section defines the MVP's core Attention Episode abstraction and decision behaviour. Numeric policy constants are intentionally deferred to historical replay and prospective validation where evidence is required.

---

## 1. Attention Episode definition

An **Attention Episode** is a continuous period in which glucose and recent context indicate a potentially meaningful situation that the engine should keep track of until there is sufficient evidence that the situation has resolved.

An episode is **internal memory, not an alert**. Most candidate episodes should be allowed to exist and resolve without ever interrupting the user.

### Episode principles

1. A meaningful developing rise can start an episode **before glucose is high**. The candidate threshold should be permissive enough to let the engine watch useful developing situations silently, while persistence/data-quality requirements protect against one noisy sample.
2. One continuous unresolved situation is **one episode**, not a new event per glucose reading or notification.
3. Relevant context can predate the episode. Recent `Ate`, insulin, modeled IOB/activity, exercise and recovery context are gathered when relevant even if they occurred before the episode opened.
4. New or corrected context immediately re-evaluates an existing episode. A backdated exercise event, for example, can change the interpretation of a currently active episode.
5. An episode does not resolve because of one favourable reading. Resolution requires persistent evidence/hysteresis.
6. A short recently-resolved linkage window should allow an immediate reversal to continue/reopen the prior situation rather than behave as an unrelated new episode. The duration is replay-tuned.
7. **Episode existence answers “is there something unresolved worth watching?” The Attention decision answers “does the user need to know about it now?”**

A high/rising episode may therefore remain completely quiet when recent insulin, exercise, recovery or improving trajectory makes another interruption unhelpful.

---

## 2. Episode lifecycle and Attention decisions

Keep episode lifecycle separate from the context attached to it.

### Lifecycle

- **Active** — a meaningful situation exists and remains unresolved. The engine evaluates it on every relevant event. Active does not imply a notification.
- **Resolving** — persistent evidence is moving in the desired direction, but there is not yet enough evidence to close the episode. Reversal returns it to Active.
- **Resolved** — sufficient persistent evidence says the situation has cleared. Pending contextual notifications/reminders are cancelled and the episode becomes historical.

The recently-resolved linkage window is retained internally but does not need to be a user-visible lifecycle state.

### Decision on each evaluation

- **QUIET** — keep watching without interrupting.
- **NOTIFY** — the current unresolved situation warrants attention.
- **ESCALATE** — prior context/defer that justified quieting is no longer sufficient because the situation has materially changed or worsened.
- **RESOLVE** — persistent recovery evidence is sufficient to close the episode.

`Ate`, insulin/IOB, Exercise, recovery context, `No insulin needed`, defer deadlines and notification history are **context**, not mutually exclusive lifecycle states.

A notification being sent also does not create an `Alerted` episode state; notification history is part of the context used by subsequent decisions.

---

## 3. Evaluation model

The engine is event-driven. Relevant events include:

- fresh glucose;
- treatment logged/edited/deleted;
- user action;
- Exercise start/end/correction;
- app launch/reconciliation;
- explicit reminder/defer boundary where applicable.

Every fresh glucose reading re-evaluates an active/resolving episode. Important context changes also cause immediate re-evaluation.

---

## 4. First-interruption policy

The first notification should not be driven by one static high threshold.

> **Notify when the current evidence suggests there is a meaningful unresolved situation where the user's attention could be useful.**

### Inputs

The MVP first-attention policy considers:

- current glucose position/severity;
- trajectory, persistence and robust acceleration/deceleration evidence;
- recent `Ate` context;
- recorded insulin timing;
- modeled IOB / insulin activity from recorded insulin;
- active/recent exercise;
- automatically derived low/recovery context;
- data freshness/quality;
- previous user/Attention context where relevant.

Fresh glucose severity and persistent worsening progressively dominate mitigating contextual assumptions as the situation becomes more concerning.

### Three broad paths to first attention

#### Early intervention opportunity

A convincing developing rise may warrant attention before glucose becomes conventionally high when attention now is plausibly more useful than waiting for a later threshold crossing.

The bar should be higher while glucose remains in range: persistent convincing movement is required rather than a single trend arrow/sample.

#### Unresolved high

An already-high situation may warrant attention even without extreme rate-of-rise when there is insufficient evidence that the situation is already being handled or resolving.

Missing treatment data must be described neutrally: `no recent recorded insulin` is not proof that insulin was not taken.

#### Material deterioration

A situation can warrant attention despite mitigating context when fresh glucose evidence becomes sufficiently concerning. Recorded insulin, Exercise, recovery context and prior acknowledgements buy contextual patience, not immunity.

### `Ate`

`Ate` alone never generates an Attention notification.

A moderate early post-meal rise is expected context and should generally increase patience rather than urgency. The meal becomes more relevant as a persistent/worsening rise remains unresolved, particularly when there is no recorded insulin evidence.

### Recorded insulin / modeled activity

Recent recorded insulin and modeled activity can delay or reduce contextual attention. They never permanently satisfy an episode.

The engine is deciding whether **another interruption is useful yet**, not whether an insulin dose was correct or sufficient.

### Exercise

Active/recent exercise should **substantially raise the contextual high/rise Attention bar**. The user may deliberately prefer to run somewhat high during exercise rather than receive contextual prompting that could encourage unnecessary attention toward correction while exercise may increase subsequent hypoglycaemia risk.

This is not an absolute `high + exercise = never alert` rule. Fresh material worsening and the independent core safety path remain able to surface important conditions.

### Low/recovery context

The user is **not required to press a `Waiting for recovery` button**. The engine already has the glucose sequence and should automatically derive obvious low/recovery context.

A rapid rise immediately out of a low should not be treated like an ordinary fast-rise/high situation. As glucose position, time and trajectory move materially beyond plausible recovery, normal contextual Attention policy can progressively resume.

### Uncertainty

Noisy/ambiguous glucose evidence should generally lead the contextual engine to watch for more evidence rather than manufacture a strong inference, where doing so remains within independent safety bounds.

Stale glucose is different: stale data invalidates current trajectory inference and is handled by the independent data-health safety path.

### Policy implementation

Do **not** begin with an opaque weighted `Attention Score`.

Use explicit deterministic rules that can be explained, replayed and tested. Exact glucose boundaries, persistence windows, slope definitions, material-worsening definitions, insulin patience and Exercise modifiers are selected through replay and validated live.

---

## 5. User actions change context, not physiological truth

Every user action writes structured context and immediately re-evaluates the episode.

General invariant:

> **User actions change what the engine knows about the user's actions or intent; they do not establish physiological truth.**

Therefore:

- `Ate` ≠ known carb amount;
- Exercise ≠ glucose will fall;
- insulin logged ≠ sufficient insulin;
- `No insulin needed` ≠ insulin is medically unnecessary;
- `Remind me` ≠ ignore new evidence.

### Log insulin

When insulin is logged:

1. write/update the canonical treatment;
2. recompute modeled IOB/activity;
3. immediately re-evaluate the episode;
4. normally give the recorded treatment appropriate time to act;
5. allow material fresh deterioration to override that patience.

The engine never decides whether the logged amount was appropriate.

### Remind me

An explicit reminder is a user request to hear again at a chosen time if the situation still meaningfully needs attention.

Contract:

> **Notify again at the requested time if the situation remains meaningfully unresolved. Notify sooner if it materially worsens. Cancel the reminder only if there is convincing evidence that it has resolved or is clearly resolving at a sufficient rate.**

Weak or marginal improvement is **not** enough to silently cancel an explicit reminder.

The common reminder duration should be available in one tap. Less-common durations should be available through a low-friction `More…` path; exact interaction and intervals belong in Chunk 3.

A reminder uses an OS-owned local notification as its time backstop while fresh glucose continues to drive live re-evaluation.

### No insulin needed

Semantic meaning:

> **The user has seen the current situation and deliberately does not intend to take insulin for it right now.**

It is not a medical claim that insulin is unnecessary.

The engine should stop asking the same question repeatedly, become substantially more patient, retain the episode, continue evaluating fresh glucose, and override the acknowledgement if the situation materially changes/worsens. Its influence is bounded and expires.

### Exercise

Exercise is a contextual fact, not an acknowledgement that an Attention Episode is handled.

Logging or changing Exercise context immediately re-evaluates an active episode and may materially raise the contextual high/rise Attention bar.

Exercise has a lifecycle:

- live Exercise can be started and later explicitly stopped;
- if expected duration is supplied, that estimate must not silently assert that Exercise definitely ended;
- retrospective Exercise should allow start and end times to be recorded;
- backdated/corrected Exercise context can re-evaluate an existing episode.

Exact tap flow, exercise types, durations and timing controls belong in Chunk 3.

### Waiting for recovery — removed from MVP UI

Do not expose a `Waiting for recovery` action in MVP. Requiring the user to explain a low-recovery trajectory that the engine can reliably infer would add unnecessary cognitive load.

Product principle:

> **Never ask the user to provide context the app can reliably derive itself. Explicit input is reserved for facts or intent the system cannot observe.**

### Undo / Edit

Low-friction actions must be easy to correct.

- lightweight Attention events should support practical Undo/correction;
- insulin requires clear Undo/Edit because an incorrect amount/time changes modeled IOB and subsequent Attention decisions;
- corrections trigger immediate re-evaluation.

---

## 6. Re-alerting and escalation

The engine should avoid repeatedly telling the user something they already know while preventing an unresolved situation from disappearing indefinitely.

### Automatic re-alerts

Absent an explicit reminder request, do **not** repeat a contextual notification merely because a fixed snooze timer elapsed.

Automatic re-attention is earned by:

1. **meaningfully new evidence** relative to the last notification/user action; or
2. a **bounded unresolved-duration safety ceiling** that causes the engine to reconsider an episode that has remained unresolved for too long even without dramatic deterioration.

Meaningful new evidence may include:

- materially greater glucose severity;
- meaningfully faster/persistent rise;
- insulin context becoming less persuasive with elapsed action time and continued worsening;
- Exercise ending while the high/rise remains unresolved;
- recovery reversing;
- another decision-relevant context change.

Do not notify on every slightly worse reading. Escalation is relative to what the user already knows.

### User actions change the subsequent interruption bar

After `No insulin needed`, the engine should not notify again merely because the original first-notification condition remains true. It requires meaningful change or the bounded unresolved-duration backstop.

After insulin is logged, the subsequent question is whether renewed attention is useful **despite** the recorded treatment and modeled activity.

Exercise ending is a decision-relevant context change and triggers immediate re-evaluation; it does not automatically force a notification.

### Explicit reminders are different

An explicit `Remind me in N` request has stronger semantics than an automatic re-alert.

Default at the requested deadline: **deliver the reminder if the episode remains meaningfully unresolved**.

Cancel it only when convincing persistent evidence shows that the situation has resolved or is clearly resolving at a sufficient rate. Marginal improvement is not enough.

Material worsening can ESCALATE before the requested reminder time.

### Maximum contextual silence

Every unresolved episode that has previously warranted attention must remain subject to a bounded maximum quiet interval appropriate to its context/severity. Exact ceilings are replay-selected and safety-validated rather than invented here.

---

## 7. Resolving and resolution

Improvement should make the engine quieter **before** the episode is fully resolved.

### Resolving

When persistent evidence begins moving convincingly in the desired direction:

- lifecycle becomes Resolving;
- contextual notification urgency drops sharply;
- automatic re-alerts should generally remain quiet;
- explicit pending reminders may be cancelled only when recovery is convincing enough to satisfy the stronger explicit-reminder contract;
- reversal returns the episode to Active.

Do not send congratulatory/recovery notifications by default. Silent recovery is the desired product behaviour.

### Resolution

Resolution requires persistent evidence based on **both glucose position and trajectory over time**, not a single favourable arrow/sample.

An episode may resolve because:

- glucose has returned to an acceptable region with convincing stable/improving trajectory; or
- the original developing-rise condition clearly dissipated before becoming meaningfully high.

A still-materially-high value should not instantly resolve merely because it has begun falling.

Candidate episodes may open, remain completely silent and resolve without ever notifying the user. That is potentially the product working exactly as intended.

---

## 8. Parameters to select through replay/live validation

Do not hard-code false precision into the product specification. Replay should compare candidate policies and live use should validate cognitive-load effects.

Parameters requiring evidence include:

- candidate-episode glucose/trajectory boundaries;
- persistence requirements;
- robust rate/acceleration definitions;
- material-worsening definitions;
- first-notification boundaries by context;
- insulin activity/patience policy;
- Exercise modifiers;
- automatic recovery-context decay;
- maximum contextual quiet ceilings;
- Resolving persistence requirements;
- final resolution criteria;
- recently-resolved linkage window;
- default and alternative explicit reminder durations.

These parameters remain constrained by the Safety Contract defined in the later product-spec chunk.
