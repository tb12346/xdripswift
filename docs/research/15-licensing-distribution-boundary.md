# Research 15 — Licensing + Future Distribution Boundary

**Status:** Complete  
**Research date:** 2026-08-17

> Research note, not legal advice. The goal is to identify the boundaries that matter before product specification and implementation. Any real public distribution should get current legal/regulatory review for the intended markets and feature set.

## Executive conclusion

**Private personal development on the xDrip4iOS GPLv3 codebase is straightforward. Sharing the app with other people is a qualitatively different project.** The moment another party receives a build, open-source distribution duties apply; if the product is presented as diabetes-management software that can influence when a user acts, medical-device regulation may also apply even if it never recommends an insulin dose.

For this project, the sensible boundary is:

```text
NOW: personal experimental fork
- modify/run privately
- Xcode shadow deployment
- no public product claims
- no direct insulin-dose recommendations
                |
                v
IF SHARING BECOMES REAL
run a deliberate distribution gate:
- GPL/source compliance
- third-party licence + API-term audit
- Apple distribution/EULA review
- intended-purpose + MHRA classification assessment
- privacy / health-data compliance
- clinical safety / evidence / post-market obligations if applicable
                |
                v
choose distribution architecture intentionally
```

The strongest strategic recommendation is **not to optimize the current prototype for public App Store distribution yet**. Build the personal product and validate the Attention Engine first, while keeping genuinely original domain logic as portable and xDrip-independent as practical.

---

## 1. xDrip4iOS is GPLv3

The repository `LICENSE` is the GNU General Public License version 3.

Important consequences:

- the licence permits use, study and modification;
- private modifications do not need to be published merely because they exist;
- modified source distributed to others must remain under GPLv3 as a whole where it is a covered/derivative work;
- distributed binaries require access to the corresponding source in the GPL-prescribed manner;
- recipients receive the right to run, modify and propagate the covered work;
- additional restrictions cannot be imposed that take away the GPL rights;
- modified versions should be clearly marked as changed.

GNU's own GPL FAQ explicitly states that a modified GPL program can be used privately without releasing the modifications. GPLv3 section 2 similarly permits making, running and propagating covered works that are not conveyed.

### What this means now

For one-person use on the user's own devices, there is **no GPL requirement to publish every experimental commit or make the custom app available to others**.

Keeping the fork public is fine, but that is a project choice rather than a requirement triggered merely by private use.

---

## 2. The key GPL boundary is conveying a copy to another party

GPLv3 defines conveying broadly as propagation that enables **other parties** to make or receive copies.

Practical examples:

| Situation | Distribution boundary |
|---|---|
| Build in Xcode and install on own iPhone | Personal use; no downstream recipient |
| Install own build on own replacement iPhone | Still personal use |
| Run own private/internal build through own developer account for own use | Primarily personal deployment; do not treat this as permission for broader sharing |
| Give an IPA/build to a friend | Another party receives a copy: GPL distribution duties apply |
| Add another person as a TestFlight tester | Binary is being supplied to another person: treat as GPL distribution |
| Public TestFlight | Distribution |
| App Store release | Distribution |
| Publish only source on GitHub | Source distribution under GPL; simpler than binary distribution but still a public release of software |

Once we distribute a binary, the **corresponding source needs to correspond to that build**, not merely to a roughly similar branch. GNU's GPL FAQ specifically warns that the source must match the distributed binary version.

### Build reproducibility becomes a compliance feature

If we ever distribute binaries, record at least:

- exact Git commit/tag;
- build/version number;
- dependency versions;
- relevant build scripts/configuration needed to produce the work;
- licence/notice bundle;
- a stable source URL for that exact release.

This aligns well with our existing recommendation to version policies, models and schemas.

---

## 3. Our xDrip modifications cannot simply become closed-source later

If we modify GPL-covered xDrip code and distribute the resulting covered work, the whole covered combination must be licensed under GPLv3 to its recipients.

So a future plan like:

> "We will build everything inside xDrip now, then later copy the finished code into a proprietary App Store app"

is not a safe assumption.

There are two separate concepts:

1. **Ideas, product requirements, algorithms and independently reimplemented concepts** are not automatically the same thing as copying GPL-covered implementation code.
2. **Code derived from or combined into the GPL work** cannot simply be relicensed by us if we do not own all relevant copyright or the upstream licence does not permit it.

### Product implication

If future non-GPL reuse matters, preserve architectural separation now rather than trying to unscramble it later.

---

## 4. The pure Attention Engine is also a useful licensing boundary

The architecture we already chose is helpful:

```text
xDrip UI / glucose / treatment / notification adapters
                     |
                     v
              AttentionEngine
         pure rules + domain types
                     |
                     v
          decisions / reason codes
```

A genuinely independent domain module can avoid importing xDrip UI, Core Data entities or proprietary app plumbing. That has technical and licensing benefits.

### Possible future strategy

If public-product optionality becomes important, consider keeping **new, original, separable core components** under an explicit permissive or separately controlled licence, while the xDrip-specific application/adapters remain GPLv3.

For example, original components might include:

- pure `AttentionEngine` rules;
- domain event/decision types;
- replay engine primitives;
- generic IOB abstractions where independently implemented;
- similar-episode feature calculation;
- generic policy-evaluation tooling.

The xDrip-facing layer would contain:

- `BgReading` adapters;
- `TreatmentEntry` adapters;
- xDrip Core Data integration;
- existing notification plumbing integration;
- xDrip SwiftUI screens and settings;
- Watch/widget integration tied to xDrip data stores.

### Important caveat

This is **not a licence loophole**. Whether a component is genuinely independent/derivative is fact-specific. If we want to rely on separate licensing for future commercial or App Store reuse, structure and ownership need proper legal review.

The technical recommendation stands regardless: keep core decision logic pure and loosely coupled.

---

## 5. Current Swift package dependencies do not show an obvious licence blocker

The project currently references these Swift packages:

- `SwiftCharts` — Apache License 2.0;
- `ActionClosurable` — MIT;
- `CryptoSwift` — permissive zlib-style licence with attribution requirements;
- `PieCharts` — Apache License 2.0.

GNU lists Apache 2.0 as compatible with GPLv3. MIT-style permissive licences are also generally GPL-compatible.

This means the currently visible Swift package set does **not** present an obvious reason we cannot continue private development or GPL distribution.

### But this is not a complete distribution audit

Before any release to others, generate a full software bill of materials / third-party notice inventory covering:

- Swift packages;
- copied/embedded source files;
- assets/icons/fonts;
- Ruby/Fastlane build dependencies where relevant to supplied build scripts;
- ML models/datasets if embedded;
- nutrition databases;
- API SDKs;
- any code adopted from Loop, Trio, AAPS, OpenAPS or other research projects.

Do not assume that something being visible on GitHub makes it reusable under our chosen terms.

---

## 6. External APIs are a separate contractual boundary

Meal-photo analysis, nutrition lookup, AI providers and similar remote services have their own terms independent of xDrip's GPL licence.

For personal prototyping this can stay simple, but before public distribution re-check:

- whether health-related use is permitted;
- whether user images/data are retained or used for provider training;
- data-processing geography;
- caching/storage restrictions;
- attribution requirements;
- per-user/request restrictions;
- whether one developer API key may serve a public app;
- whether the API key can be safely kept server-side;
- pricing at real usage levels;
- terms around automated decision support.

### Architecture implication

Keep external services behind provider protocols. Do not make the Attention Engine depend on a particular vendor's proprietary response format or licensing terms.

Also do not embed long-lived secret API keys in a publicly distributed client.

---

## 7. Apple App Store distribution has a genuine GPL tension

This is a meaningful future blocker, not a reason to stop the personal prototype.

GPLv3 section 10 says downstream recipients automatically receive GPL rights and that the distributor may not impose further restrictions on those rights.

Apple's current Developer Program agreement requires App Store EULAs to include minimum terms. The required scope includes a **non-transferable licence** to use the application on Apple-branded products under Apple's Usage Rules.

Those concepts are in tension with GPL rights that allow recipients to propagate the covered work.

### Consequence

Do **not** assume that putting this GPLv3 xDrip fork directly on the public App Store is a routine administrative step.

A custom EULA is not an obvious fix because Apple says a custom EULA may not conflict with its required minimum terms.

Before a central public App Store distribution of the GPL-derived app, obtain specialist open-source/Apple-platform legal advice.

### What this does *not* mean

It does not prevent:

- private local Xcode use;
- publishing GPL source;
- continuing the existing DIY/self-build model;
- experimenting privately with TestFlight for the developer's own use.

It means **central distribution to other people through Apple is a separate licence gate**.

---

## 8. Public TestFlight is not a loophole around distribution obligations

Apple describes TestFlight as beta distribution. Internal and external testers receive copies of the application; external beta distribution is reviewed and TestFlight builds expire after 90 days.

From the GPL side, if another person receives the GPL-covered binary, treat that as conveying a copy and satisfy GPL obligations.

From Apple's side, TestFlight is governed by Apple terms and App Review rules; App Review Guidelines say TestFlight betas should be intended for public distribution and comply with the Guidelines.

### Recommendation

For future community sharing, the lowest-friction path to investigate first is **source + DIY personal builds**, similar to xDrip's existing model, rather than one central public TestFlight run by us.

That does not eliminate medical-device questions, but it avoids assuming a clean central GPL/App Store binary distribution path.

---

## 9. GPL compliance and medical-device regulation are independent

Open source does not exempt medical software from medical-device regulation.

Likewise, being fully compliant with medical-device regulation would not remove GPL obligations.

A future shared product potentially has to satisfy **both**.

This is important because it is easy to think:

> "It is free and open source, therefore we are not really a manufacturer."

MHRA's Software and AI as a Medical Device programme explicitly recognises open-source software as an area where manufacturer responsibility can arise when someone modifies/deploys code.

---

## 10. Direct dose recommendations are clearly across the regulatory line

Current MHRA guidance says software is most likely to be a medical device if it is intended to influence actual treatment — including **dose, time or type of treatment**.

MHRA public guidance also gives a very direct example: software is likely to be a medical device when a user enters information and the app uses it to calculate a medicine dose to take/inject.

Apple is similarly strict. Current App Review Guideline 1.4.2 says drug dosage calculators must come from specified approved healthcare entities or have regulatory approval from the FDA or an international counterpart.

### Product consequence

Our existing decision to keep **direct insulin-dose recommendations out of scope** is strongly supported.

A meal-photo estimate should remain:

- food recognition;
- carb range;
- user confirmation/edit;
- context for attention/personalisation;

not:

- "take X units".

---

## 11. But an Attention Engine may still be medical-device software if distributed

This is the more subtle conclusion.

A shared/public version could plausibly have an intended purpose such as:

> monitor diabetes data and determine when a person should be alerted to take action.

That is not merely a neutral diary.

Current UK guidance states that software is most likely a device when it is intended to influence the time/type/dose of treatment. Other MHRA material also identifies diabetes-management, patient monitoring and clinical decision-support software as categories that can fall within medical-device regulation.

Our app would potentially influence **when a user attends to and acts on glucose**, even without specifying a dose.

### Therefore

Do not rely on wording such as:

- "experimental";
- "not medical advice";
- "for informational purposes only";

as a mechanism for escaping medical-device status if the actual intended purpose/function is medical.

MHRA places heavy emphasis on the manufacturer's **intended purpose**, including what the software does, the population, users, medical condition, inputs/outputs and how output influences decision-making.

### Personal prototype versus public product

For the private prototype we can appropriately describe it as experimental personal software and avoid making public product claims.

If sharing becomes real, we should draft a formal intended-purpose statement and obtain classification advice **before** deciding the release channel.

---

## 12. Free distribution can still count as placing a medical device on the GB market

MHRA's current registration guidance says placing on the GB market is the first making available of a device for use/distribution, and specifically says devices can be **sold, leased, lent or gifted**.

So:

> "We are giving it away for free"

is not a regulatory exemption if the software qualifies as a medical device and is being placed on the Great Britain market.

If a medical device is placed on the GB market, the manufacturer generally has obligations around conformity, registration and ongoing compliance according to device classification and route.

MHRA guidance updated in July 2026 states that devices must be registered before being placed on the GB market.

---

## 13. A regulated public version creates ongoing obligations, not just a one-time certificate

If a future version is classified and distributed as a medical device, expect a substantially different engineering/operating model.

Depending on classification and route, this can include:

- clear intended-purpose documentation;
- risk management;
- software lifecycle/change controls;
- verification and validation;
- clinical/performance evidence;
- cybersecurity controls;
- usability/human-factors evidence;
- conformity assessment and appropriate marking;
- MHRA registration for GB;
- incident/vigilance reporting;
- post-market surveillance;
- controlled release/change processes.

MHRA's current post-market surveillance rules require manufacturers to define a PMS system/plan proportionate to device risk and link real-world information to corrective/preventive action.

This matters particularly for personalised/learned behaviour. A model that continuously changes how alerts are produced is a much bigger public regulatory proposition than a versioned deterministic policy tested and released deliberately.

### Product implication

Our plan to keep learned models in **shadow mode** before promotion is not only good engineering; it is aligned with the discipline a future regulated product would need.

---

## 14. Public distribution also changes privacy obligations

The personal prototype is deliberately local-first and largely processes the user's own data for the user's own purposes.

A service offered to other people changes the privacy posture.

The ICO identifies health data, including medical-device/fitness-tracker data and health inferences, as **special category personal data** under UK GDPR.

If we operate services that receive other users' glucose, insulin, meal, exercise, sleep or prediction data, we would need to determine roles and obligations such as:

- controller/processor roles;
- lawful basis for personal data;
- an Article 9 condition for special-category processing;
- clear privacy notices;
- data minimisation;
- retention/deletion policies;
- security controls;
- processor/vendor agreements;
- international transfer considerations;
- breach/incident handling;
- user rights.

### Apple adds additional HealthKit constraints

Apple's current rules prohibit use of HealthKit-derived health data for advertising/marketing/data-mining purposes and impose restrictions on disclosure to third parties. Apple also requires HealthKit apps to disclose how the health data is used and provide a privacy policy.

### Architecture consequence

The current local-first approach remains the best default:

- raw HealthKit history stays on device where possible;
- derive compact context rather than copy all data to a cloud;
- photos stay local unless the user initiates remote analysis;
- Nightscout remains user-controlled where practical;
- no advertising/data-broker business model around health data.

---

## 15. Rebranding modified public builds is prudent

GPL grants copyright permissions; it does not give us a blanket right to present a modified app as though it were an official upstream release.

GPLv3 also expects modified versions to carry prominent notices that they were changed.

Even before a formal trademark analysis, a future shared build should:

- have a distinct product name/icon;
- state that it is based on xDrip4iOS;
- link to upstream;
- retain required copyright/licence notices;
- clearly identify our modifications/version;
- avoid implying endorsement by upstream maintainers, Zukka, CGM manufacturers or Apple.

The temporary distinct identity recommended in Pass 14 therefore helps both safety and future distribution hygiene.

---

## 16. Recommended future distribution ladder

Do not jump directly from personal prototype to public App Store product.

### Level 0 — current personal use

```text
source fork
  ↓
user's own Xcode/developer account
  ↓
user's own devices
```

**Recommended now.**

Primary obligations are safe engineering and compliance with the upstream licence for the source we use. No need to manufacture a public release process.

### Level 1 — publish research/source

```text
public GPL source
+ documentation
+ no centrally supplied public binary
```

Open-source obligations are straightforward. Regulatory/intended-purpose review is still needed if the project is actively offered as a medical product rather than merely published research/source.

### Level 2 — DIY community builds

```text
public source
+ self-build instructions
+ each user signs/builds their own copy
```

This resembles current xDrip practice and avoids us centrally shipping a binary through the App Store.

Still do not assume it eliminates manufacturer/regulatory responsibility if the product is promoted for diabetes management.

### Level 3 — small beta supplied by us

```text
we supply binaries to other testers
```

Before doing this:

- GPL exact-source compliance;
- Apple/TestFlight terms review;
- privacy policy/data-flow review;
- formal intended-purpose/regulatory assessment;
- stronger safety/evidence process.

### Level 4 — public App Store / broad product

This is a separate product programme.

Likely needs:

- specialist GPL/Apple legal review or different software licensing architecture;
- medical-device regulatory strategy if intended purpose qualifies;
- company/legal manufacturer identity;
- quality/risk/clinical evidence processes;
- privacy/commercial infrastructure;
- ongoing support/vigilance/update capability.

Do not design the personal MVP around reaching Level 4 quickly.

---

## 17. If a broad App Store product becomes the goal, there are two architecture paths

### Path A — remain xDrip-derived and open-source

Pros:

- reuse mature CGM infrastructure;
- community transparency;
- existing architecture/features;
- lower implementation cost.

Cons:

- GPLv3 obligations continue;
- central Apple binary distribution needs legal review because of EULA/usage-rule tension;
- upstream dependency/licence/brand provenance stays relevant;
- medical regulation still applies independently.

### Path B — eventually create a clean standalone application

Pros:

- choose licensing deliberately;
- simplify target/product surface;
- potentially easier central App Store licensing/distribution;
- no need to carry all xDrip legacy architecture.

Cons:

- large engineering effort;
- CGM integrations are complex and often vendor/protocol sensitive;
- cannot simply copy GPL-derived implementation into the new app;
- medical regulation is **not** avoided by a rewrite.

### Likely sensible strategy

Use xDrip to discover whether the product is valuable. Do not pay the cost of a standalone rewrite before that question is answered.

Preserve optionality by keeping original decision logic modular and well-specified.

---

## 18. Reuse / adapt / build / ignore

### Reuse

- GPLv3 xDrip codebase for personal prototype;
- existing licence file/notices;
- current permissively licensed Swift packages;
- xDrip's DIY/self-build deployment precedent;
- local-first health-data architecture;
- deterministic/versioned safety and Attention policies.

### Adapt

- release/build metadata so an exact distributed build can map to exact source;
- About/legal screen if/when another user receives a build;
- third-party notice generation;
- distinct branding for modified builds;
- privacy and provider abstraction before any multi-user release.

### Build if distribution becomes real

- formal distribution checklist/gate;
- dependency/SBOM licence inventory;
- release tags and exact-source archive;
- privacy policy/data-flow inventory;
- intended-purpose statement;
- regulatory classification assessment;
- risk-management/clinical-safety evidence appropriate to intended purpose;
- post-market/incident process if regulated;
- public support/update ownership;
- separate provider credential/backend architecture where remote APIs are used.

### Ignore for the personal MVP

- App Store submission;
- public TestFlight programme;
- company formation solely for this prototype;
- UKCA/CE submission work before intended purpose/public distribution is real;
- full QMS implementation;
- public marketing claims;
- direct insulin-dose recommendation;
- rewriting xDrip solely to escape GPL.

---

## 19. Open questions for a future distribution decision

Do not answer these now unless the project actually moves beyond personal use:

1. Will the distributed product merely display/log information, or actively decide when the user should act?
2. What exact intended-purpose wording would we make publicly?
3. Which markets: Great Britain only, EU/NI, US, Australia, others?
4. Source-only/DIY distribution or centrally supplied binary?
5. Can a GPLv3 app be distributed through the intended Apple channel under then-current terms without additional restrictions?
6. Do we want the future product itself to remain GPL/open source?
7. Which original components should be explicitly separable/permissively licensed now?
8. Who is the legal manufacturer/operator if another person uses the app?
9. Which cloud/API providers receive health data?
10. What evidence would support claims about reduced missed action, improved TIR or reduced cognitive burden?
11. How would a personalised model be frozen/versioned/validated for release?
12. What support and incident-response commitment is realistic?

---

## 20. Product-spec implications

1. The product spec we write next can focus on **personal experimental use**; do not burden MVP requirements with public-distribution machinery.
2. Keep direct insulin-dose recommendation explicitly out of scope.
3. Do not assume attention-only functionality is outside medical-device regulation if later distributed.
4. Preserve a clearly versioned intended behaviour: deterministic policy first, learned signals bounded/shadowed.
5. Keep the Attention Engine pure and xDrip-independent behind protocols.
6. Keep original domain logic sufficiently separable that future licensing architecture remains a choice.
7. Keep xDrip-specific adapters thin.
8. Preserve exact policy/model/schema/build versions in event records.
9. Local-first storage remains the default for privacy as well as engineering reasons.
10. Keep remote AI/nutrition providers replaceable and do not hardcode secrets.
11. Add an explicit source/build identifier to future experimental builds so observed behaviour can be traced to code.
12. A modified build should have a distinct identity from upstream/Zukka before any sharing.
13. If another person is ever invited to a build, stop and run the distribution gate first.
14. Do not use App Store publication as the validation mechanism for the product idea.
15. If broad public distribution becomes a strategic goal, assess GPL/Apple compatibility and MHRA intended-purpose classification before committing to an architecture/channel.

---

## Bottom line

For the foreseeable next phase:

> **Build the best personal experimental diabetes Attention app we can, safely, on xDrip.**

There is no need to solve public distribution yet.

But we should preserve three boundaries now:

```text
1. Core safety vs personalised attention
2. Original portable domain logic vs xDrip-specific integration
3. Personal experimental use vs any release to another person
```

The third boundary is the trigger for a new workstream. If crossed, the question stops being only "can we ship this?" and becomes:

> **Can we legally license it, distribute it on the chosen platform, protect other people's health data, substantiate its medical purpose, and operate it safely over time?**

That should be a deliberate decision, not an accidental consequence of adding someone to TestFlight.

---

## Primary sources

### GPL / open source

- GNU GPLv3: https://www.gnu.org/licenses/gpl-3.0.html
- GNU GPL FAQ: https://www.gnu.org/licenses/gpl-faq.html
- GNU licence compatibility list: https://www.gnu.org/licenses/license-list.html
- Repository GPLv3 licence: `LICENSE`
- Swift package references: `xdrip.xcodeproj/project.pbxproj`
- SwiftCharts licence: https://github.com/ivanschuetz/SwiftCharts
- ActionClosurable licence: https://github.com/takasek/ActionClosurable
- CryptoSwift licence: https://github.com/krzyzanowskim/CryptoSwift
- PieCharts licence: https://github.com/paulplant/PieCharts

### Apple

- Apple App Review Guidelines: https://developer.apple.com/app-store/review/guidelines/
- Apple Developer Program License Agreement: https://developer.apple.com/support/terms/apple-developer-program-license-agreement/
- Apple agreements/guidelines: https://developer.apple.com/support/terms/
- TestFlight overview: https://developer.apple.com/help/app-store-connect/test-a-beta-version/testflight-overview
- Apple Health/fitness privacy guidance: https://developer.apple.com/health-fitness/
- HealthKit privacy: https://developer.apple.com/documentation/healthkit/protecting-user-privacy

### UK medical-device regulation

- MHRA software/AI as a medical device: https://www.gov.uk/government/publications/software-and-artificial-intelligence-ai-as-a-medical-device
- MHRA intended-purpose guidance: https://www.gov.uk/government/publications/crafting-an-intended-purpose-in-the-context-of-software-as-a-medical-device-samd
- MHRA current clinical-investigation/software qualification guidance: https://www.gov.uk/government/publications/medical-devices-that-need-a-clinical-investigation/determining-if-a-clinical-investigations-is-required
- MHRA medical-device registration: https://www.gov.uk/guidance/register-medical-devices-to-place-on-the-market
- Regulating medical devices in the UK: https://www.gov.uk/guidance/regulating-medical-devices-in-the-uk
- SaMD vigilance guidance: https://www.gov.uk/government/publications/reporting-adverse-incidents-involving-software-as-a-medical-device-under-the-vigilance-system
- Manufacturer post-market surveillance system: https://www.gov.uk/government/publications/medical-devices-post-market-surveillance-requirements/requirements-of-the-manufacturers-pms-system

### UK privacy

- ICO special-category/health data guidance: https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/lawful-basis/special-category-data/what-is-special-category-data/
