# First phone companion build

## Phone setup check — 2026-10-28

The phone is now connected and paired, and Xcode visibly lists the user's Apple
Account with a **Personal Team**. Developer Mode is now enabled. The user explicitly
authorized development signing and phone registration; Xcode created the Apple
Development identity. The first signed build reached a Watch complication
provisioning failure because no Watch was registered. The free trial now excludes
Watch build dependencies and embedding through build settings, preserving those
targets for normal V7 and standard trial builds. Both iOS extensions, HealthKit
and isolated App Groups remain. Eleven packaging checks pass, including strict
phone-only bundle checks. The main app's embedded development profile includes
the connected phone and expires on 4 November 2026. After the user completed
the macOS Keychain prompt, the signed phone build succeeded. The app and both
iOS extensions passed signature, identity, entitlement, device inclusion and
profile-expiry checks. **Attention Trial is installed on the physical phone.**
First launch was denied by iOS with its signature/entitlement/developer-trust
security message. After the user completed developer trust, the launch retry
succeeded and a subsequent device process check confirmed the installed app and
its widget were running. This verifies phone startup, not foreground UI state,
live follower setup, sustained background operation or alert delivery. No
credentials were copied, health data requested or Zukka settings changed.
The GitHub research head still matches the checked-out research base; existing
uncommitted V7 and research work was preserved. Previous build results below are
from October 20, not a fresh phone test.

The fresh phone-only simulator build also passed, installed and launched on the
existing iPhone 17 simulator. Treatments and the Attention inspection screen
were visually verified, including Free-account trial, frozen C-v1, shadow-only,
and no configured companion source. The first simulator attempt found stale
test-framework products from the previous Xcode version; regenerable products
were moved outside the output folder and the freshly assembled retry passed.
Installed simulator data and all source work were preserved. No full simulator
test-suite rerun is claimed for this packaging-only change; eleven packaging
checks and the frozen-source guard passed today.

### Live follower and restart check — 2026-10-28

The user configured the trial as a Nightscout follower with a readable token and
reported success. A narrowly scoped read of the trial's local shadow journal
confirmed recent glucose observations and recorded-insulin context had reached
the frozen C-v1 recorder. This is live integration evidence, not just a successful
credential test. No insulin quantities or credentials were copied into this doc.

The agent restarted only Attention Trial and re-read the journal. Its previous
event sequence was preserved exactly, new observations were appended, and the
latest persisted capture was recent. This verifies persistence and resumption
after an app restart. It does not yet verify phone reboot, locked/background
recording, sustained follower freshness, IOB model configuration, or treatment
uploads. The two diagnostic journal copies remain in a restricted temporary
folder outside Git; no app preferences, credentials or broader database export
was copied. Zukka remained untouched and experimental Attention delivery stayed
off.

### Initial locked-phone check — 2026-10-28

With the user reporting the phone locked, reading the journal was denied by iOS
file protection. The app process remained present, but journal size and modification
time did not advance during the observed interval, reaching more than four minutes
since its last save. Database-file metadata also showed no recent writes during
an intermediate check. Process presence alone does not prove background execution;
these observations do not distinguish suspension, no new source data, and a blocked
journal write. The user was asked to unlock and open the trial without force-quitting
so resumption can be checked. Continuous locked/background recording is **not yet
verified**.

Code review confirms the journal currently uses complete file protection and
any persistence error permanently pauses that runtime instance. This is a
potential recovery issue, not yet proof that a write failed during this particular
lock interval. Apple's [file-protection documentation](https://developer.apple.com/documentation/foundation/nsdata/writingoptions/completefileprotection)
states that this protection level prevents reads and writes while locked. The
protection level and all follower/core-alert settings were left unchanged for
the diagnostic check.

### Unlock recovery fix — 2026-10-28

After the user reopened the original trial without force-quitting, two journal
reads more than a minute apart were identical and the latest saved capture was
over seven minutes old. The observed foreground stall matches the code's permanent
pause on a write error; no runtime error string was extracted from the phone, so
the specific triggering exception is not independently confirmed.

The recorder now treats unavailable protected data as temporary. A loaded session
may keep observations in bounded memory while locked, retaining their actual
availability times. It writes that pending sequence atomically after protected
data becomes available, including when the next input snapshot is unavailable.
Foreground entry also retries this path. A launch while locked defers restoring
history and does not start an empty replacement session. The entire saved event
sequence is validated before replay. Corruption, policy mismatch, conflicting
readings and genuine unlocked write failures still stop recording and preserve
the previous file.

Complete file protection, backup exclusion, frozen C-v1, upstream follower code,
IOB calculation, alert settings and upload settings are unchanged. Pending memory
observations are explicitly labelled unsaved and can be lost if the process ends
before unlock. This does not claim durable locked recording or keep iOS from
suspending the app.

All 16 targeted simulator runtime tests passed with no skips, including five new
cases for buffering/flush, lock during a write, locked launch, genuine disk failure,
and invalid later events. The first run's only failure was an assertion checking
an on-device protection attribute that Simulator does not expose; that unsupported
assertion was removed, while the behavioural checks remained and passed. Physical
file-protection validation subsequently passed as described below. The frozen-source guard and signed
bundle/identity checks passed. Simulator launch succeeded; the final visual
inspection could not run because the Mac was locked.

The signed update was installed over the existing phone app. A private journal
check confirmed its entire previously saved prefix was preserved and fresh captures
and newer glucose observations resumed with the existing connection. No personal
readings, treatment identifiers or credentials are in Git.

### Physical unlock recovery verified — 2026-10-28

The second phone check passed. Two attempts to read the protected journal while
the phone was locked were denied, confirming the update did not expose its saved
contents while locked. Database metadata showed activity during that interval.
After the user unlocked and opened the trial without force-quitting, the saved
journal included glucose observations whose capture/availability timestamps fell
inside the observed locked interval. Its complete previous event prefix was
preserved, and a recent capture after unlock confirmed saving had resumed.
Process checks before and after unlock matched: recovery happened in the same
running app, without reinstalling or restarting it.

This establishes background capture and automatic unlock persistence for this
observed cycle. It does not establish overnight reliability, durability if iOS
terminates the process while observations are buffered, phone-reboot recovery,
or network-interruption behaviour. The readable token does not validate treatment
upload access, and experimental Attention delivery remains off. The next delivery
milestone is still the opt-in actionable prompts/reminders described below.

## Free-account trial decision — 2026-10-20

The user chose trying a free Apple Personal Team before deciding whether paid
membership is worthwhile. V7 now has an explicit free build variant, layered
over the existing isolated Attention Trial configuration. It removes only the
main app's NFC entitlement and prevents NFC session creation; an attempted scan
explains the build restriction instead of claiming the phone lacks NFC. Normal
V7 and standard trial NFC behaviour are unchanged. No frozen engine, safety
alert, logging, sync or Nightscout account settings were changed.

HealthKit, isolated App Groups and the four existing embedded bundles are
retained, not preemptively cut. Apple's current capability table lists HealthKit
and App Groups for free accounts. NFC, APNs push and Time Sensitive Notifications
are not available there; the latter two entitlements are not in the existing
trial. Ordinary local notification actions remain a possible implementation,
not evidence that Attention delivery has been enabled. No phone reliability is
implied by an available capability.

The inspection screen labels this build **Free-account trial** and explains
the seven-day provisioning expiry. Free and standard trial builds use the same
identity for a given team; switching teams can change storage and must be
reviewed before migration. Never delete the trial to renew it. Zukka's App Store
installation remains independent. Build tooling requires an explicit team for
signed builds and never performs automatic provisioning/enrollment.

The 38 frozen engine tests, 10 packaging checks and 85 focused simulator tests
passed (133 total, no failures/skips). The built app and four embedded bundles
passed the metadata isolation check. Resolved free settings retain both trial
flags and target-specific extension entitlements; a separate normal V7 settings
check resolves to `.v7review`, its original entitlement file and DEBUG only.
The free app was installed and launched on the existing iPhone 17 simulator.
Visual inspection confirmed the V7 Treatments screen, Free-account trial label,
seven-day/NFC guidance, notifications off, retained history, Quiet with unavailable
glucose and unknown IOB. No health events or source/alert settings were changed.

The initial unsigned-device compilation failed with **No space left on device**;
the first simulator build was interrupted to avoid filling the disk. The failed
device build's 42 MB temporary folder was removed (regenerable products only).
Once the Mac reported about 4 GB available, the simulator retry passed as above.
The sequential unsigned-device retry also succeeded; its app and four embedded
bundles passed the metadata isolation check. This proves compilation, not signing
or phone operation. Build/test logs are outside Git in `/private/tmp`.

No physical phone or Apple Development signing identity was available at this
turn's checks. Personal Team signing and the
actual extension profiles remain unverified until the user's Apple Account is
available in Xcode and their phone is connected/trusted. Compilation is not an
installable signed build. No membership purchase or agreement was performed.

References: [Apple capabilities](https://developer.apple.com/help/account/reference/supported-capabilities-ios)
and [Personal Team limits](https://developer.apple.com/help/account/basics/about-your-developer-account).

## Previous milestone — 2026-10-05

**2026-10-05 — self-contained simulator/unsigned-device build verified; signing
and physical-phone trial pending.** The user approved preparing a V7 companion phone
build, then actionable Attention notifications/reminders, with Zukka retained
as the independent safety layer until explicit review. This is not permission
to disable existing safety alerts, silently enable uploads, or pair another CGM.

## Build slice

The actual V7 checkout (`review/v7-ui`) now contains an immutable C-v1 package
under `Packages/AttentionReplay`. Its source/manifest bytes match the original
pre-evaluation fingerprints. The build rejects source changes or additional
Swift sources. The research checkout remains the policy-authoring authority;
the V7 snapshot is a pinned distribution, not a separate policy fork. This
replaces the sibling-checkout dependency described in milestone 09 without
altering the original freeze, tests, health data or V7 UX work.

`Config/AttentionTrial.xcconfig` and `Tools/AttentionTrial/build.sh` explicitly
select a separate **Attention Trial** identity for app and extensions. Normal
V7 builds retain their identity. Entitlement groups now follow their existing
build-setting values; the trial supplies separate groups and disables outward
OS-AID sharing using upstream's existing build flag. Its URL scheme and widget
link are distinct so they cannot claim Zukka's `xdripswift://` links. No new
Live Activity behaviour was added; only the existing link's build identity is
parameterized. No sensor, credentials, alert or upload settings are changed.

The trial flag also permits the shadow recorder in an explicitly selected trial
build independently of DEBUG. It never enables notification delivery. The
inspection screen explains companion setup, the separation of historical
replay/live follower credentials, and that existing core xDrip alerts have
separate settings. A shadow-only engine is not an all-alerts-off app.
The trial-only first-launch instructions explicitly say not to pair a second
sensor, replacing the normal direct-sensor onboarding text for this build only.

## Observer regression found during the broader test pass

The initial trial suite exposed a deadlock: the shadow observer registered a
Core Data save notification on `OperationQueue.main`. NotificationCenter waits
for such delivery, so a background save could wait for main while main waited
for that saving context during a glucose fetch. Enqueuing again inside that
observer did not remove the initial synchronous wait.

The recorder now receives notifications on the posting queue and explicitly
enqueues the read on main, without handing managed objects across queues. The
same rule covers context/provider/foreground notifications. A regression test
holds main while a background notification posts and asserts that posting can
finish independently, then verifies deferred main delivery. No core safety
alert or upstream IOB calculation was changed.

An existing OS-AID test assumed sharing was compiled on and expected cleanup
writes. It now checks the opposite, intended contract when `DISABLE_LOOP_SHARE`
is selected: shared storage remains untouched. Its normal-build assertions are
preserved; the test was not skipped to hide an isolation failure.

## Live-data/logging plan, not yet verified on a phone

- Preserve Zukka's current sensor connection. Verify an appropriate follower
  source for the companion before using it; do not infer latency/reliability
  from historical downloads.
- Preserve approved context mirroring. The old historical read-only token cannot
  support one-app logging with treatment uploads. Verify write access and the
  canonical treatment round trip without duplicate records before relying on it.
- No fake treatment is to be uploaded to the real account for testing. Use mocks
  and a separately authorized real-world verification step.
- Exercise locked/background/network-loss/restart behaviour on the phone.
  Complete file protection on the current journal can restrict locked access;
  resolve that lifecycle behaviour before claiming continuous shadow recording.
- Next implementation after packaging: opt-in, namespaced Attention delivery,
  direct opening into logging, short explicit reminders, cancellation/reassessment,
  and a shadow-only switch. Review overlapping routine-high alerts with the user;
  do not change low, urgent, stale-data or other core safety settings automatically.

## Verification and blocker

The frozen package's 38 tests and six new packaging checks passed. The final
focused simulator suite passed all 84 tests with no failures or skips (engine/journal,
observer regression, isolation/onboarding, follower settings, background
keep-alive, gap-fill and home interactions). Simulator and unsigned iPhone
builds both succeeded after the observer fix. Metadata checks passed for the
app and all four embedded bundles on both platforms; ordinary V7 build settings
still resolve to their previous identity/groups. The original research freeze
still verifies. Private data remains ignored/untracked.

The trial installed and launched on the existing iPhone 17 simulator. Its first
notification permission was explicitly declined for this shadow-only check.
An informational licence notice was dismissed, not a source setup completed.
After the user unlocked the Mac, the final trial identity was relaunched and
visually checked. Treatments → blue waveform opened the companion setup screen,
with the expected follower/setup guidance and explicit Attention-notifications-off
label. With no live source, the engine showed no episode, Quiet, and unavailable/
stale glucose rather than a recovery claim; IOB remained unconfirmed/unknown.
All 18 in-app synthetic results rendered, while the original retained Quiet
history remained present after relaunch. No fake records were logged and no
connection/model/alert settings were enabled during this check.

No connected physical phone was discoverable. Signed build failed because trial
app/extension development profiles are absent; no automatic provisioning or
Apple account change was attempted. The user confirmed Zukka is an App Store
installation published by another developer. That does not grant signing access
for this custom trial; leave it untouched and establish a separate authorized
Apple development account/team for our installation. Membership choice and
account sign-in remain user actions; no enrollment, purchase or agreement was
performed by the agent.

Simulator tests cannot verify physical-phone notification delivery, file
protection, background reliability or server-side sync. No daily-use readiness
or replacement of Zukka is claimed.
