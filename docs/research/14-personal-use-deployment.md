# Research 14 — Personal-Use Deployment

**Status:** Complete  
**Research date:** 2026-08-17

## Executive conclusion

**Use a two-stage personal deployment strategy: direct Xcode installation for active development, then an optional private/internal TestFlight channel once the app is stable enough to use daily. Keep one stable, unique custom bundle identity throughout development, and initially run the custom app alongside Zukka as a Nightscout Follower rather than a second CGM master.**

That gives us the fastest debugging loop without risking the existing glucose path:

```text
                    active development
              Mac + Xcode + paid team
                        |
                        v
             unique stable “lab” app
                        |
                        v
                custom V7 on iPhone
                 /              \
                /                \
     Nightscout Follower        Attention features
              |                 widgets / Watch
              v                 logging / alerts
           live data
              ^
              |
            Zukka
   remains existing CGM/safety path

later, when useful:
GitHub Actions + Fastlane
        |
        v
private/internal TestFlight
(Mac-independent install + recovery)
```

The most important deployment distinction is:

> **Two apps can coexist on one iPhone, but two apps should not casually compete to own the same CGM connection or duplicate the same downstream side effects.**

The first real-device qualification should therefore be a **shadow deployment**, not a cutover.

---

## 1. Recommended deployment channels

### Stage A — development / shadow channel

Use direct Xcode installation while we are building and debugging.

Recommended characteristics:

- paid Apple Developer Program membership;
- current supported Xcode version for the V7 branch;
- Developer Mode enabled on the iPhone;
- stable custom bundle identifier;
- distinct display name and eventually a visibly different icon;
- automatic signing;
- install from `xdrip.xcworkspace`;
- custom app starts in Nightscout Follower mode;
- Zukka remains the existing direct-CGM / safety path during qualification;
- update the same installed custom app in place rather than creating a new identity for every build.

Why this is the best development loop:

- debugger and device logs are available immediately;
- no TestFlight upload/processing cycle for every change;
- signing is managed by Xcode;
- the Watch app and extensions can be tested as part of the same project;
- a stable bundle ID keeps the development app's own sandbox/data across rebuilds;
- the custom build can coexist with Zukka because it is a different app identity.

### Stage B — stable personal beta channel

Once the app is useful enough that losing access while away from the Mac would be annoying, add a private/internal TestFlight build from the fork.

Benefits:

- reinstall from the TestFlight app without the development Mac;
- easier recovery after accidental deletion or phone replacement;
- a stable personal build can be separated from whatever experimental branch is currently being debugged;
- the upstream project already contains GitHub Actions + Fastlane infrastructure for this model.

Costs:

- App Store Connect setup;
- signing/certificate management;
- repository secrets;
- CI configuration;
- TestFlight builds expire after 90 days;
- the current browser-build workflow needs adaptation/qualification for a custom fork branch rather than being assumed safe as-is.

### Ad hoc distribution

A paid developer account can also distribute to registered devices without TestFlight, but for this project it is not the preferred path. Direct Xcode is better for development and TestFlight is better for personal recovery/installation away from the Mac.

---

## 2. A free Apple account is a poor fit

Apple allows a free Personal Team to run apps on personal devices, but its development resources are intentionally short-lived:

- App IDs expire after 7 days;
- registered devices expire after 7 days;
- provisioning profiles expire after 7 days;
- apps must then be rebuilt/reinstalled;
- the Personal Team is limited to three installed development apps per device.

That is already awkward for a glucose app that should remain reliably available.

It is an even worse fit for this codebase because xDrip currently uses capabilities including HealthKit and NFC. xDrip's own build documentation notes that a free-account build requires stripping unsupported capabilities and repeating signing frequently.

### Recommendation

For a phone app intended to run continuously and eventually exercise HealthKit, Watch, widgets and Bluetooth functionality, treat a **paid Apple Developer Program membership as a project prerequisite for serious device work**.

Apple currently lists the program at **99 USD per membership year, or local currency where available**.

We should not design the product around seven-day Personal Team reprovisioning.

---

## 3. Developer Mode is required for the Xcode path

For locally installed development-signed iOS apps, Apple requires Developer Mode.

The flow is broadly:

1. pair the iPhone with the Mac/Xcode;
2. enable Developer Mode under Privacy & Security when prompted;
3. restart/confirm the device;
4. select the iPhone as the Xcode run destination.

Developer Mode is specifically for locally installed development software. It is **not** required for ordinary TestFlight installation.

This is another reason TestFlight becomes useful later as the “stable personal” channel while Xcode remains the development channel.

---

## 4. V7 already has a good personal-signing seam

The current V7 branch uses an xcconfig-based setup rather than requiring us to edit project signing settings everywhere.

Current `xDrip/xDrip.xcconfig` defines defaults such as:

```text
MAIN_APP_BUNDLE_IDENTIFIER = com.$(DEVELOPMENT_TEAM).xdripswift
MAIN_APP_DISPLAY_NAME = xDrip4iO5
XDRIP_CODE_SIGN_STYLE = Automatic
XDRIP_DEVELOPMENT_TEAM = ...

#include? "xDripOverride.xcconfig"
#include? "../xDripConfigOverride.xcconfig"
```

The repository `.gitignore` explicitly ignores:

```text
xDripConfigOverride.xcconfig
```

This is a useful existing convention.

### For local development

Create a local override rather than modifying upstream defaults:

```text
XDRIP_DEVELOPMENT_TEAM = <our Team ID>
MAIN_APP_BUNDLE_IDENTIFIER = <our stable custom bundle ID>
MAIN_APP_DISPLAY_NAME = <distinct dev display name>
```

The exact final name is not important yet. The operational requirement is that it is obviously not Zukka.

### Why this matters

Signing configuration stays local and does not create noisy upstream merge conflicts. A stable custom bundle ID also means each Xcode build updates the same development app rather than creating a fresh sandbox.

---

## 5. One main bundle ID drives the companion targets

The V7 Xcode project derives companion target identifiers from `MAIN_APP_BUNDLE_IDENTIFIER`.

Current patterns include:

```text
main app
$(MAIN_APP_BUNDLE_IDENTIFIER)

widget
$(MAIN_APP_BUNDLE_IDENTIFIER).xDripWidget

Watch app
$(MAIN_APP_BUNDLE_IDENTIFIER).watchkitapp

Watch complication
$(MAIN_APP_BUNDLE_IDENTIFIER).watchkitapp.xDripWatchComplication

notification content extension
$(MAIN_APP_BUNDLE_IDENTIFIER).xDripNotificationContextExtension
```

That is good for our personal fork: choose one unique root identity and preserve the existing hierarchy.

### Product/deployment consequence

Do not randomly change bundle IDs during development. Changing identity should be treated like creating a different installation channel, because it creates a different app sandbox and different extension identities.

---

## 6. A distinct display name/icon is a safety feature, not branding work

Polished branding is still out of scope, but visual distinction is operationally important when Zukka and the experimental app are both installed.

The custom build should eventually have:

- a clearly different display name, e.g. a temporary `xDrip Lab` / `Attention Lab` style name;
- a visibly different icon or development badge;
- distinguishable notification wording where practical.

This reduces the chance of:

- changing settings in the wrong app;
- assuming one app sent an alert when the other did;
- opening the wrong treatment/logging surface;
- confusing Watch complications/widgets during testing.

This is not a request to do final visual design. It is deployment hygiene.

---

## 7. Current V7 capabilities make paid signing the realistic path

The current main-app entitlements include:

```text
HealthKit
NFC reader TAG format
Loop app group
Trio app group
```

The project also contains:

- widget target;
- Watch app;
- Watch complication;
- notification content extension;
- Live Activities;
- Bluetooth central background mode;
- audio background mode.

The current main `Info.plist` includes:

```text
UIBackgroundModes:
- audio
- bluetooth-central
```

and declares HealthKit/Bluetooth/NFC usage descriptions.

The widget and Watch targets use shared app-group entitlements.

### Implication

A “minimal stripped development build” is the wrong long-term target. We want to qualify the actual capability topology that the product will rely on.

---

## 8. App Groups are separate from Zukka

The V7 project currently derives its shared app groups from the Apple development team, including:

```text
group.com.<TEAMID>.loopkit.LoopGroup
group.org.nightscout.<TEAMID>.trio.trio-app-group
```

Those groups allow the app and its own extensions/compatible AID apps under the same developer team to share the intended data.

Because our personal build is signed by our own Apple team and has its own bundle identity, it should not be assumed to share Zukka's private app-group container.

### Consequence

The custom app is a separate installation with a separate sandbox. Nightscout/backup import should be treated as the intentional interoperability/history route rather than trying to share Zukka's local files directly.

---

## 9. Installation coexistence is easy; CGM ownership coexistence is not

A distinct bundle ID means iOS can install the custom build alongside Zukka.

The harder question is what each app does once both are running.

xDrip can operate in two very different roles:

```text
Master
→ connects directly to supported CGM/transmitter over Bluetooth

Follower
→ obtains readings remotely, e.g. from Nightscout
→ does not need to own the direct CGM connection
```

Trying to use two xDrip-family apps as independent masters against the same physical sensor/transmitter can create contention or sensor-specific connection problems.

### Initial recommendation

**Do not make the development build a second CGM master during early qualification.**

Instead:

```text
Zukka
  → keeps existing CGM / safety role
  → continues producing/uploading the user's normal glucose stream

custom V7 build
  → Nightscout Follower
  → consumes live glucose history
  → develops/tests Attention behaviour
```

The user already has Nightscout, which makes this particularly practical.

This gives us live real-world data without changing the existing sensor ownership arrangement.

---

## 10. Shadow mode must also avoid duplicate side effects

Even as a Follower, two apps can still duplicate actions downstream if both are fully enabled.

Potential duplication includes:

- HealthKit glucose writes;
- Nightscout uploads;
- treatment uploads;
- glucose alerts;
- Watch notifications;
- Live Activities;
- calendar/contact side effects.

### Recommended shadow policy

During first development qualification:

**Keep Zukka's existing safety behaviour intact.**

For the custom app:

- consume Nightscout glucose as a Follower;
- disable or avoid duplicate HealthKit glucose writing initially;
- avoid generic Nightscout re-upload of data that already came from Nightscout;
- make treatment-sync behaviour explicit before relying on it;
- keep standard duplicate glucose alerts disabled or carefully controlled while testing distinct experimental Attention notifications;
- label experimental notifications clearly enough that the source is obvious;
- do not make the experimental build the sole safety-alert source.

This is the deployment version of Pass 13's safety-island principle.

### Later cutover

If we eventually want the custom app to become the main/master app, that should be a deliberate qualification step with sensor-specific verification rather than something that happens automatically because both apps are installed.

---

## 11. Use the same custom bundle ID for every normal dev rebuild

A stable bundle identity matters for more than coexistence.

Keeping the same identity means subsequent Xcode installs update the existing custom app and retain its app container unless the app is explicitly deleted or the data model itself changes incompatibly.

That lets us accumulate:

- local Attention events;
- episode state;
- settings;
- test treatment history;
- model/policy metadata;
- future local personalisation history.

### Avoid

Do not create IDs like:

```text
...lab1
...lab2
...lab3
```

for ordinary iteration.

If we later intentionally create separate **development** and **stable TestFlight** channels, then two bundle IDs may be reasonable, but that should be a conscious choice because they will have separate local data stores.

For now, one custom identity is simpler.

---

## 12. Existing Zukka local data will not automatically appear in the custom app

Because the custom build is a different app identity, it has a separate sandbox.

We should therefore assume:

```text
Zukka local Core Data
        ≠
custom app local Core Data
```

### Useful migration/interoperability paths

1. **Nightscout** for glucose/treatments/history where available.
2. **xDrip backup/restore** if a compatible export can be produced and safely imported.
3. New Attention data starts in the custom app's own local store.

The current V7 `Info.plist` registers the xDrip backup document type, so backup/import remains a viable qualification path.

Do not make our deployment plan depend on directly extracting Zukka's app sandbox.

---

## 13. Watch, widgets and Live Activities need real-device qualification

The V7 repo has separate targets for:

- iPhone app;
- Watch app;
- Watch complication;
- widget/Live Activity extension;
- notification content extension.

A successful iPhone build alone therefore does not prove the deployment is ready.

### Qualification should verify

- Watch companion installs and launches;
- Watch connection/data flow works with the custom app identity;
- complications update;
- widgets can read the intended app-group state;
- Live Activities still function;
- notification content extension signs correctly;
- uninstall/reinstall behaviour is understood;
- the custom display name/icon is distinguishable on both phone and Watch.

This is one reason to keep automatic signing and the existing derived bundle-ID scheme rather than hand-editing each target independently.

---

## 14. Background behaviour must be tested while Zukka is still present

The V7 main app declares Bluetooth central and audio background modes, but entitlement/configuration presence is not proof of real-world reliability.

For a Follower/shadow build, qualification should include:

- fresh Nightscout readings while foregrounded;
- continued expected updates after screen lock;
- app relaunch reconciliation;
- stale-data handling if network/readings stop;
- notification delivery after the app has been backgrounded;
- reboot/relaunch behaviour;
- Watch/widget freshness.

If/when the app later becomes a CGM Master, we need a second qualification pass specifically for direct Bluetooth continuity.

---

## 15. Private TestFlight is useful later, not the first development loop

Apple allows a TestFlight build to be tested for up to **90 days**.

xDrip already includes a browser-build system using GitHub Actions + Fastlane. Its documented pattern includes:

- validation of required secrets;
- App Store Connect API credentials;
- signing certificate/profile generation;
- signed build/archive;
- upload to TestFlight;
- scheduled upstream checks;
- regular rebuilds intended to keep a personal TestFlight install alive.

The current V7 workflow runs on `macos-26` and explicitly selects Xcode 26.2.

### Why this is attractive later

Once set up, it gives a personal recovery channel:

```text
phone lost/deleted/app unavailable
            ↓
       open TestFlight
            ↓
        reinstall build
```

That is much more resilient than needing physical access to the development Mac for every reinstall.

---

## 16. Do not use the upstream browser-build workflow unmodified for our custom branch

There are two important reasons to qualify/adapt it first.

### 16.1 Current workflow assumes an upstream branch with the same name

The current workflow sets roughly:

```text
UPSTREAM_REPO = JohanDegraeve/xdripswift
UPSTREAM_BRANCH = github.ref_name
TARGET_BRANCH = github.ref_name
```

That works naturally for normal upstream branch names.

It does **not** map cleanly to a custom branch such as:

```text
feature/attention-engine
research/v7-readiness
```

because those branches do not exist upstream.

For a future stable custom build, either:

- disable the automatic upstream-sync step for the custom build branch and update upstream deliberately through our normal merge/rebase workflow; or
- explicitly map the custom branch to the upstream branch we track.

Do not let an automated fork-sync job unexpectedly rewrite or conflict with product work.

### 16.2 Current TestFlight documentation and current V7 bundle derivation need reconciliation

The current V7 Xcode project derives the notification-extension ID from the main bundle ID:

```text
$(MAIN_APP_BUNDLE_IDENTIFIER).xDripNotificationContextExtension
```

Older/current browser-build documentation may show a slightly different identifier pattern for that target.

That mismatch is exactly the kind of signing issue that should be caught in a small CI qualification before TestFlight becomes our recovery plan.

### Recommendation

Treat the first TestFlight setup as a deployment-engineering task with an explicit bundle/entitlement audit, not a checkbox.

---

## 17. Local override configuration does not automatically exist in CI

For Xcode development, an ignored `xDripConfigOverride.xcconfig` is ideal.

For GitHub Actions, however, the runner checks out only repository content. An ignored local file is not there.

Therefore a future CI/TestFlight channel needs a deterministic way to provide:

- Apple Team ID;
- stable custom main bundle ID;
- desired display name/channel marker;
- signing configuration.

The bundle ID/display name are not secrets and can safely be represented in repository configuration if we choose. Signing keys/tokens remain secrets.

### Product/deployment consequence

Do not build a TestFlight process that silently falls back to the upstream default identity when our Xcode development build uses a different custom identity.

---

## 18. Keep signing and service secrets out of Git

A private personal build still needs normal secret hygiene.

Never commit:

- App Store Connect private key material;
- GitHub PATs;
- Match passwords;
- Nightscout tokens/API secrets;
- signing certificates/private keys;
- any local health-data export.

Use:

- Xcode/Keychain for local development credentials;
- GitHub Actions encrypted secrets for CI;
- Keychain/local secure storage for Nightscout credentials at runtime.

The custom bundle ID, display name and Apple Team ID are identifiers/configuration rather than authentication secrets, although keeping local signing overrides out of upstream-facing diffs is still convenient.

---

## 19. Proposed real-device qualification gate

Before we treat the custom app as a serious daily companion, verify the following in order.

### Build and identity

- V7 builds cleanly with the current supported Xcode toolchain.
- `xdrip.xcworkspace` is used rather than only the project file.
- automatic signing succeeds for all required targets.
- app installs beside Zukka.
- display name/icon are clearly distinguishable.
- same bundle ID updates the custom install in place.

### Live-data shadow mode

- custom app connects as Nightscout Follower.
- fresh glucose arrives at expected cadence.
- Zukka continues its normal existing role.
- no Bluetooth/sensor ownership conflict occurs.
- stale/lost-data behaviour is clear.

### Side effects

- no accidental duplicate HealthKit glucose writes.
- no accidental duplicate Nightscout uploads.
- treatment upload/import semantics are known before relying on them.
- experimental notifications are distinguishable.
- existing Zukka safety alarms remain enabled during qualification.

### Background + system surfaces

- screen-lock/background reading behaviour works as expected.
- app survives relaunch/reboot cleanly.
- notification permissions/settings are checked.
- widget updates.
- Watch app installs.
- complication updates.
- Live Activity works if enabled.

### Data durability

- local Attention state persists across normal rebuilds.
- a deliberate backup/export path is tested.
- accidental app deletion/reinstall behaviour is understood.
- Nightscout can restore the historical substrate we expect it to provide.

### Only after that

Evaluate whether to make the custom build a direct CGM Master and whether to add a stable TestFlight channel.

---

## 20. Recommended everyday development workflow

Once device qualification is complete, the normal loop should be intentionally boring:

```text
feature branch
    ↓
build + tests
    ↓
install/update same “Lab” app via Xcode
    ↓
shadow test beside Zukka
    ↓
review logs/replay/behaviour
    ↓
draft PR
    ↓
merge when qualified
```

The deployment mechanism should not require creating new app identities or reconfiguring the phone for every feature.

A later stable channel can become:

```text
qualified stable branch/tag
        ↓
GitHub Actions
        ↓
internal TestFlight
        ↓
personal daily/recovery build
```

---

## 21. Reuse / adapt / build / ignore

### Reuse

- V7 `xDrip.xcconfig` / override-file pattern.
- existing automatic-signing structure.
- derived bundle IDs for widget/Watch/notification targets.
- existing app-group entitlements.
- current xDrip backup format/import support.
- existing GitHub Actions + Fastlane TestFlight infrastructure as a starting point.
- Nightscout Follower mode for safe shadow testing.

### Adapt

- create one stable custom root bundle ID for the personal app.
- use a distinct development display name/icon.
- make shadow mode avoid duplicate HealthKit/Nightscout/alert side effects.
- modify/qualify browser-build branch sync for a custom product branch.
- make CI use the same intended bundle identity as the stable personal channel.

### Build

- deployment checklist / app-health screen for notification and data freshness.
- explicit configuration for “shadow/follower qualification” versus later “master” use.
- a repeatable backup/recovery procedure.
- if useful later, a fork-specific internal TestFlight workflow that cannot overwrite product work through automatic upstream sync.

### Ignore for now

- public App Store release.
- external TestFlight distribution.
- enterprise distribution.
- MDM.
- ad hoc IPA distribution unless a real need appears.
- multiple parallel custom beta channels.
- polished final branding.

---

## 22. Open questions for implementation qualification

These do not block the research conclusion:

- Does the user already have an active paid Apple Developer Program membership?
- What Mac/Xcode setup will be used for the first real-device build?
- What exact CGM/sensor connection does Zukka currently own?
- Does the current Zukka build expose a compatible xDrip backup export?
- Which Zukka-side uploads/HealthKit writes are currently enabled?
- Do we eventually want one custom app channel or separate “Lab” and stable TestFlight identities?
- Is Mac-independent recovery valuable enough to justify TestFlight setup early, or only after the Attention Engine is useful?

These should be resolved during the first device-qualification/build step, not by expanding research further.

---

## 23. Product-spec implications

1. **V7 device qualification is the first implementation milestone.**
2. Assume a paid Apple Developer Program membership for the intended real-device capability set.
3. Use one stable, unique custom main bundle ID.
4. Preserve the V7 derived identifier structure for extensions/Watch/widgets.
5. Use an ignored local xcconfig override for Xcode signing/development configuration.
6. Make the development app visually distinguishable from Zukka.
7. Initial live deployment is **shadow mode**, not sensor cutover.
8. Zukka remains the existing CGM/safety path during early qualification.
9. Custom app initially uses **Nightscout Follower** for live glucose.
10. Avoid duplicate HealthKit writes, Nightscout uploads and standard alarms in shadow mode.
11. Experimental Attention notifications must be source-distinguishable and cannot replace the existing safety path yet.
12. Same custom bundle ID should update in place and preserve the custom app's local data.
13. Existing Zukka local data is not assumed to be directly accessible; use Nightscout/backup as interoperability routes.
14. Watch/widget/notification extensions are part of the deployment qualification gate, not optional afterthoughts.
15. Test background behaviour under screen lock/relaunch/reboot before relying on it.
16. Internal TestFlight is a **later stable/recovery channel**, not the primary active-development loop.
17. TestFlight builds expire after 90 days, so a stable channel needs periodic rebuild automation.
18. Adapt the existing Fastlane/GitHub Actions workflow before using it on custom feature/stable branches.
19. Do not let automatic upstream-sync logic modify custom product branches unexpectedly.
20. CI must use a deterministic custom bundle identity; an ignored local xcconfig is insufficient on a runner.
21. Keep Apple/GitHub/Nightscout credentials out of source control.
22. Direct CGM Master cutover is a separate, deliberate qualification step after shadow operation is proven.

---

## Sources

### Apple — official

- Apple Developer, *Developer account overview* — Personal Team limits and 7-day provisioning: https://developer.apple.com/help/account/basics/about-your-developer-account
- Apple Developer, *Apple Developer Program* — membership and TestFlight capability: https://developer.apple.com/programs/
- Apple Developer, *Program enrollment* — current annual fee / local-currency note: https://developer.apple.com/help/account/membership/program-enrollment
- Apple Developer Documentation, *Enabling Developer Mode on a device*: https://developer.apple.com/documentation/Xcode/enabling-developer-mode-on-a-device
- Apple Developer Documentation, *Running your app on simulated or physical devices* — automatic signing/device registration: https://developer.apple.com/documentation/xcode/running-your-app-on-simulated-or-physical-devices
- App Store Connect Help, *TestFlight overview* — 90-day build testing window: https://developer.apple.com/help/app-store-connect/test-a-beta-version/testflight-overview
- Apple Developer, *Create an ad hoc provisioning profile*: https://developer.apple.com/help/account/provisioning-profiles/create-an-ad-hoc-provisioning-profile

### xDrip4iOS / V7 — primary project sources

- PaulPlant/xdripswift `v7-beta`, `README.md` — current build/toolchain/install routes.
- `xDrip/xDrip.xcconfig` — bundle/display/signing defaults and override includes.
- `.gitignore` — local `xDripConfigOverride.xcconfig` excluded from Git.
- `xdrip.xcodeproj/project.pbxproj` — derived bundle IDs and target signing settings.
- `xDrip/xdrip.entitlements` — HealthKit, NFC and app-group entitlements.
- `xDrip Widget Extension.entitlements` — shared app group.
- `xDrip/Supporting Files/Info.plist` — background modes, backup document type, Live Activities and usage descriptions.
- `fastlane/testflight.md` — personal TestFlight setup and refresh model.
- `.github/workflows/build_xdrip.yml` — current browser-build scheduling, upstream-sync assumptions, `macos-26` and Xcode 26.2 build environment.
