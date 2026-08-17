# Diabetes Attention App — MVP Product Specification

**Status:** In progress  
**Product-spec phase:** Chunk 1 of 7 locked  
**Source research:** `docs/research/` on this branch

This is the canonical living product specification for the personal diabetes attention-management project. Product decisions should be captured here as they are locked. Implementation detail should remain subordinate to the product behaviour and safety contract.

---

## 1. Product contract

### Goal

**Help improve long-term health outcomes while reducing the cognitive load of managing diabetes, by identifying when diabetes genuinely needs attention and staying quiet when it does not.**

The product must optimise both sides of that goal:

- fewer interruptions are not success if glucose management worsens;
- better glucose metrics are not enough if achieving them requires more conscious attention.

The Attention Engine is the mechanism, not the goal. Its job is to remember the situation over time, incorporate what the user has already told or done, and decide whether the current situation warrants attention.

### Core product test

> **Compared with the existing setup, can the user safely think about diabetes less?**

Success therefore requires both:

1. **Lower attention burden** — fewer unnecessary interruptions, repeated nags and manual checks, and less need to remember that a situation has already been handled.
2. **Preserved or improved awareness/control** — meaningful worsening still becomes visible promptly; acknowledgements cannot create indefinite silence; stale, missing or conflicting data cannot create false reassurance.

The exact clinical and cognitive-load metrics used to evaluate this contract will be defined in the success-criteria section of this specification rather than assumed here.

---

## 2. First usable product

The MVP is a **personal, shadow-mode Attention Engine for unresolved rising/high glucose situations**.

It should:

- recognise a persistent rising/high situation as one ongoing **Attention Episode**, rather than treating each glucose reading or alarm as an independent event;
- remember relevant meal, insulin, exercise, recovery and user-action context;
- decide whether to **stay quiet, interrupt, remind, escalate or resolve**;
- know when relevant action has been recorded and give that action time to work when the evidence supports doing so;
- remain able to re-escalate when fresh evidence materially worsens;
- resolve an episode quietly when persistent evidence shows that the situation is recovering.

A representative desired experience is:

```text
Ate
→ glucose begins rising
→ engine watches
→ persistent concerning rise
→ one useful prompt
→ insulin is logged
→ engine knows action was recorded
→ stays quiet while the response remains plausible
→ glucose recovers
→ episode resolves without another interruption
```

A user acknowledgement is never an exemption from future evidence:

```text
Ate
→ rise
→ prompt
→ No insulin needed
→ engine backs off
→ glucose materially worsens
→ acknowledgement is overridden
→ engine prompts again
```

The MVP is therefore **bounded contextual awareness**, not merely a more sophisticated fixed snooze system.

---

## 3. Situations owned by the MVP

### Attention Engine owns

- persistent rising glucose;
- high or approaching-high situations where context changes whether attention is useful;
- meal-associated (`Ate`) rises;
- whether recorded insulin/action suggests that a situation is already being handled;
- active/recent exercise context when it changes the usefulness or urgency of a contextual high/rise interruption;
- recovery/improvement;
- re-alerting and escalation of an unresolved Attention Episode.

### Attention Engine does not own

- core low-glucose alarms;
- urgent/severe glucose safety alarms;
- lost/stale sensor safety alerts;
- replacement of xDrip/Zukka's existing core safety machinery.

Low/recovery state may be **context** for an Attention Episode without the Attention Engine taking ownership of detecting or treating the low itself.

During shadow-mode MVP use, existing Zukka/core safety behaviour remains independent. Experimental Attention notifications do not replace it.

---

## 4. MVP context

The usable MVP should be capable of evaluating:

### Required context

- current and recent CGM readings;
- glucose trajectory and persistence;
- data freshness/quality;
- `Ate` events;
- recorded insulin;
- approximate IOB / insulin activity derived from **recorded** insulin;
- manually logged active/recent exercise;
- low/recovery context;
- active Attention Episode state;
- previous Attention actions and defer state.

Approximate IOB is an attention-context signal, not a dosing system. Missing insulin records remain unknown and must never be interpreted as proof that no insulin was taken.

### Exercise context

Manual exercise is part of the MVP because it can materially change whether a high/rising situation deserves attention. In particular, the user may intentionally prefer to run somewhat high during exercise rather than receive contextual prompting that could encourage unnecessary attention toward correction while activity may increase hypoglycaemia risk.

This preference must **not** become an absolute rule such as `high + exercise = never alert`. Exercise modifies contextual Attention urgency; fresh material worsening and the independent core safety path retain authority to surface important conditions.

Exercise logging follows a **progressive enrichment** principle:

```text
Exercise
→ essential event is recorded immediately
→ optional type
→ optional duration
→ optional timing correction / started earlier
```

Every completed step saves useful information. The user is never required to finish the enrichment flow for the original Exercise event to count.

Exact exercise categories, duration choices and the interaction for backdating an exercise event are intentionally deferred to the User Actions + Interaction Model section.

### Later context, not MVP

- automatic Apple Health / Apple Watch workout detection and enrichment;
- meal-photo recognition or carb estimation;
- classical COB modelling;
- sleep or continuous heart-rate context;
- learned glucose prediction;
- personalised ML;
- similar-historical-episode retrieval.

---

## 5. MVP user vocabulary

### Proactive actions

- **Ate**
- **Log insulin**
- **Exercise**

### Reactive Attention actions

- **Log insulin**
- **Remind me**
- **No insulin needed**
- **Waiting for recovery**, when contextually relevant

`Handling it` may remain in the internal domain vocabulary but is **not exposed in the MVP UI** unless later interaction design reveals a concrete situation not covered by the more specific actions above.

### Progressive logging principle

**The first action should capture the essential fact; additional actions may enrich context but should not be required for the original action to count.**

Examples:

- `Ate` is useful immediately without a carb count.
- `Exercise` is useful immediately without type or duration; type, duration and timing can enrich it.
- Insulin is different because the amount is fundamental to a valid insulin treatment record.

---

## 6. Explicit MVP non-goals

The MVP does **not** include:

- insulin-dose recommendations;
- automated insulin delivery;
- replacement of existing core CGM safety alarms;
- meal recognition or photo analysis;
- required carb counting;
- COB modelling;
- learned glucose forecasting;
- ML/personalisation;
- automatic Apple Health/Watch physiological context;
- sleep/heart-rate modelling;
- a sophisticated Watch treatment/exercise app;
- Live Activities as a core Attention surface;
- public distribution;
- a polished redesign of xDrip;
- a general-purpose configurable smart-alarm builder.

The MVP is also **not an AI diabetes assistant**. Future learned or AI components may provide bounded evidence, but the product is an attention-management system whose core behaviour must remain understandable, testable and safe without them.

---

## 7. MVP versus implementation slices

The MVP is the smallest version worth using, not necessarily the first piece of software implemented.

Implementation should be sliced more narrowly where useful. For example:

- glucose-only episode detection may exist before `Ate` is wired in;
- recent recorded-insulin timing may precede the full IOB model;
- manual Exercise may be added after the core episode state machine;
- individual system surfaces may arrive after their shared domain actions.

These intermediate slices are development milestones, **not reasons to weaken the definition of the usable MVP**.

---

## 8. Decisions intentionally deferred from Chunk 1

The following belong in later product-spec chunks:

- exact Attention Episode opening/resolution rules;
- persistence, worsening and recovery definitions;
- exact quiet/remind/escalate policy;
- maximum contextual silence ceilings;
- exercise type/duration vocabulary and backdating interaction;
- exact notification actions by episode state;
- safety thresholds and invariant policy;
- detailed persistence/data model;
- final MVP surfaces;
- success metrics and qualification gates.

Where a numeric policy cannot be justified confidently now, it should be selected through historical replay and then validated prospectively rather than invented for the specification.
