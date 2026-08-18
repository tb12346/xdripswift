# Product Spec — Chunk 3.1: Ate Interaction

**Status:** Locked  
**Parent spec:** `PRODUCT_SPEC.md`

`Ate` is a proactive, low-friction context action. It should be available through a quick surface such as a widget/control rather than as a routine alert-response action.

## Core interaction

- One tap on **Ate** immediately records a meal at the current time.
- No confirmation screen or Save button is required.
- The event is useful immediately to the Attention Engine.
- The common action must remain one tap even when optional enrichment exists.

## Immediate follow-up

After recording, show a lightweight transient confirmation with the most useful next action:

- **Log insulin** — promoted because Ate + insulin commonly happen together.
- **Add details** — optional enrichment.
- **Undo** — correction path.

Do not nag with a required question such as “Did you take insulin?”. `Log insulin` should be offered as a fast optional continuation.

## Optional enrichment

`Add details` may expose:

- meal size: Small / Medium / Large;
- free-text meal description/note;
- time correction/backdating.

No enrichment is required for the original `Ate` event to count.

The exact UI can evolve, but progressive enrichment is required: the first tap captures the essential fact, later taps add context only if the user wants to.

## Timing and provenance

A time correction updates `occurredAt` while preserving the original `recordedAt`.

Quick relative choices are preferred before a full time picker, e.g. Now / 15m ago / 30m ago / 45m ago / 1h ago / More… . Exact options may be tuned during interaction design.

## Semantics and trust

- `Ate` means approximately “I ate at this time.”
- Meal size is coarse subjective context.
- Free-text description may support historical interpretation or later personalisation, but deterministic core safety behaviour must not depend on extracting precise carbohydrate truth from it.
- `Ate` does not imply a carb amount, insulin action, or adequate treatment.

## Alert surfaces

`Ate` should not occupy a primary reactive Attention-notification action slot in MVP. It is primarily a proactive quick action/widget/control.

If a future need emerges for logging an omitted meal from an alert, that can live behind a secondary `More…` path without burdening the default notification UI.

## Repeated meals and corrections

Do not aggressively deduplicate multiple nearby `Ate` events: the user may genuinely have eaten more than once.

Accidental duplicates must be easy to Undo/correct. Missing or duplicate uncertainty must not be silently rewritten into false certainty.
