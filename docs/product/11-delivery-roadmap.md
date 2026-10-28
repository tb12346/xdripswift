# Delivery roadmap: live companion to a working personal app

Updated: 2026-10-28. Owner: the active implementation chat. This is the execution
plan and handoff index; update it after each meaningful slice.

## Destination

Receive useful contextual Attention prompts, respond or request a short reminder,
log once with reliable sync, and understand missing data or failures. Reduce actual
interruptions and taps. No dose advice; silence never means glucose is safe.
Replacing Zukka requires a separate explicit decision after prospective use.

## Current baseline

- The phone has the V7 UI, real quick logging, Ate/Exercise context and frozen C-v1
  engine, in an isolated free-account Attention Trial. Nightscout supplies live
  readings; Zukka retains its sensor connection and independent safety alerts.
- Live glucose and recorded-insulin context reach the local journal. Restart and
  a physical lock/unlock cycle passed. Upstream IOB is separate from the frozen
  timing-only policy; model confirmation remains a setup check.
- Saved history remains protected while locked. Observations buffered in memory
  can be lost if iOS terminates the process. Sustained reliability is unverified.
- New Attention alerts are off; reminders are not implemented. The current readable
  token does not establish treatment-upload access.
- Free signing needs renewal before 2026-11-04; update over the existing app, never
  delete it to renew. This variant omits NFC scanning and Watch packaging.

Published evidence: [phone integration](10-phone-companion-build.md).
Additional local worktree references: `09-frozen-evaluation-and-v7-shadow-integration.md`,
`03-2-interaction-prototype-feedback.md`, `03-3-attention-context-data.md`.

## Milestones

Statuses: **todo**, **in progress**, **implemented**, **verified**, **enabled**.
Code existing, checks passing, installation and user opt-in are separate facts.

| ID | Outcome | Status | Completion gate |
| --- | --- | --- | --- |
| M0 | Isolated live phone companion | verified | Signed install, live journal, restart and one lock/unlock cycle passed. Longer observation belongs to M4. |
| M1 | Optional Attention notifications + Open into quick logging | implemented | 39 targeted simulator tests and 11 packaging checks pass; signed build installed/launched on phone, history and live recording preserved. Visible OS notification/Open check awaits Mac unlock. Live opt-in remains off. |
| M2 | Remind 10m + reassessment/cancellation | todo | Existing UX choice, not a clinical threshold. OS-owned backstop; worsening can interrupt earlier; convincing recovery cancels; stale-data wording cannot imply a fresh assessment. Persist reminder intent across restart. |
| M3 | One-app logging and reliable Nightscout sync | todo | Appropriate write access; create/edit/undo/retry/deduplication verified with mocks, then an explicitly authorized real treatment round trip. Preserve Ate/Exercise mirroring. No fake real-account records. |
| M4 | Reliable daily companion and usefulness review | todo | Longer foreground/background/locked/network-loss/reboot use; honest gap/storage indicators; battery, permissions, renewal and notification overlap checked. Review actual interruptions and response effort. |
| M5 | Everyday-use release / possible Zukka replacement | todo | M1–M4 evidence reviewed with user; independent low/urgent/stale safety behaviour, source/fallback, logging/IOB and installation plan verified. User explicitly decides alert ownership and replacement. |

M1–M3 form the first usable alert-and-response release. Do not restart research
or demand more historical data merely to implement it. Later Lock Screen controls,
Live Activity actions, paid membership and Watch support are enhancements whose
priority follows use of the core response loop; they are not implied by M1.

## M1 checklist

- [x] App-side delivery adapter outside the immutable engine; explicit default-off switch.
- [x] Fresh live Notify/Escalate only; no startup replay or old prompt on opt-in.
- [x] Deduplicate callbacks/restarts; handle opt-out during an in-flight request.
- [x] Cancel only Attention-owned notifications on opt-out, recovery or stale input at the next running evaluation.
- [x] Preserve existing categories, core alert routing, sounds and snooze settings.
- [x] Authenticated Open routes to existing quick logger without prefilling an amount or writing a treatment; visible OS route check remains below.
- [x] Show scheduling/permission/error status without claiming confirmed delivery.
- [x] Fake-transport and policy tests, simulator build/run, signed phone update.
- [ ] Verify visible OS notification/Open interaction; record physical notification validation and user opt-in separately. Installed with delivery off.

## Boundaries and architecture

- Never recommend, calculate or automate insulin doses. Retain upstream IOB;
  do not change the frozen policy under the same name. Missing logs mean unknown.
- Existing low, urgent, stale-data and other core safety alerts remain independent.
  No automatic changes to Zukka, sensor pairing, uploads or overlapping alerts.
- Use an Attention notification namespace and narrow upstream hooks. Never call
  global notification removal. Notification text requests attention, not insulin.
- Preserve complete file protection and expose unsaved/buffered status. Document
  any future privacy/durability trade-off explicitly before changing it.
- Synthetic notifications must be labelled. No fake treatments on real Nightscout.
- No credentials, raw health data or signing/device identifiers in Git.

## Handoff protocol

1. Read this plan, PRODUCT_SPEC.md and the newest milestone entry first.
2. Product/research: `xdripswift`, branch `research/v7-readiness`. Actual app:
   sibling `xdripswift-v7-review`, branch `review/v7-ui`. Do not build the old UI.
3. Inspect both dirty worktrees; preserve existing logging/UX work. No reset,
   blanket staging, overwrite or rebase. Most app integration is still local and
   uncommitted: publishing this roadmap does not publish its code dependencies.
4. Use app `Tools/AttentionTrial/README.md` and `build.sh`, free variant and existing
   simulator. Signing uses the local user's Personal Team, not the repo default.
5. Keep `Packages/AttentionReplay` fingerprint-identical. New policies need a
   separately named/evaluated research version.
6. Continue routine implementation/testing autonomously; involve the user only
   for necessary phone/account actions and product choices.
7. Before ending, update milestone status and append dated evidence: what is
   implemented, tested, installed, enabled, blocked, and the next concrete task.
8. Publish plan/status updates to the user's repo. Before another-machine handoff,
   curate a reproducible V7 code checkpoint including its currently local dependencies.

## Progress and decisions

### 2026-10-28 — Plan accepted; M1 started

User requested a persistent end-to-end plan and implementation of the first step.
M1 delivers optional live notifications with Open into the existing quick logger.
M2 adds the agreed reminder interaction, then M3 completes one-app logging.
Nothing enables notifications or uploads just by installing. Zukka remains the
independent safety layer. Next concrete task: implement/verify M1, record evidence,
then proceed to M2.

### 2026-10-28 — M1 code and automated checks

Added `AttentionNotificationDelivery.swift` outside the frozen package. Runtime
publishes new final-capture evidence; restoration never delivers, earlier backfill
prompts are filtered and unknown input cancels eligibility. Requests use isolated
identifiers and generic text without glucose/dose/context payloads. A persisted
attempt watermark prevents resending the same decision, including after restart.
Permission failure or scheduling error is visible; it is not reported as delivery.
The same failed decision is not retried automatically; a new engine decision may
try again. App suspension still prevents proactive cancellation until it runs.

Existing AlertManager registration adds the Attention category without replacing
the existing set. Narrow coordinator routing recognizes only Attention identifiers.
Open requests the existing logger after authentication; it creates no health event.
An explicit, clearly labelled setup notification tests OS delivery without enabling
live prompts or inventing glucose/treatment events.

All 39 selected simulator tests passed (delivery, shadow runtime and home interaction;
no skips). Frozen fingerprints remain unchanged. Simulator launch succeeded, but
the Mac lock blocked visible OS notification/Open verification; user unlock was
requested while signed build work continued. M1 is implemented, not yet fully
device-verified or enabled. Next: complete these checks, then M2 reminders.

### 2026-10-28 — M1 phone update installed; visible check pending

Signed build, bundle isolation and signature verification passed; 11 packaging
checks also passed. The app was updated in place and launched on the phone.
A private journal check confirmed the complete previous prefix was preserved
and fresh observations continued. Attention opt-in was not enabled. The labelled
setup test can be invoked in Treatments → waveform → Attention notifications;
it is independent of live opt-in and writes no health events.

The Mac remained locked, so real banner rendering and tapping Open into the
logger could not be visually verified. Automated routing tests are not a
substitute for that check. User unlock was requested; finish it before claiming
M1 verified. Then implement M2, not another research/packaging detour. App source
changes remain in the V7 dirty worktree; only curated product docs are published.
