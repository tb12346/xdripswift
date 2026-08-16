# Research 09 — Meal Photo Recognition + Carb Estimation

**Status:** Complete  
**Research date:** 2026-08-16

## Executive conclusion

**Do not train a custom food-vision model for the first version. Use a pluggable multimodal/specialist vision provider, ground the result in nutrition data, and make the user confirm an editable carb range before it becomes treatment data.**

Current vision systems are already good enough to make a useful **second opinion**, but image-only carbohydrate estimation is not reliable enough to be treated as dosing-grade truth. The hard problem is usually not recognizing that a plate contains pasta, rice or bread; it is estimating **portion size**, hidden ingredients and mixed-dish composition.

The first product should therefore optimize for:

1. instant capture of the fact that the user **ate**;
2. useful food/component recognition;
3. a carb **range** with explicit uncertainty and assumptions;
4. one-tap confirmation or correction;
5. storing confirmed data for later personalization;
6. never converting the image estimate directly into an insulin recommendation.

Recommended MVP flow:

```text
photo captured
   ↓
record Ate immediately
   ↓
MealVisionProvider analyzes image
   ↓
food components + portion assumptions
   ↓
nutrition grounding
   ↓
35–50 g estimated carbs
"biggest uncertainty: rice portion"
   ↓
Looks right / Edit
   ↓
confirmed carb TreatmentEntry (optional)
```

The provider should be replaceable. For an initial personal benchmark, compare a general multimodal model with a specialist nutrition API rather than committing to one vendor up front.

---

## Questions for this pass

- Is there useful open-source meal-recognition code or training data we can reuse?
- Are there APIs that already combine food recognition, portion estimation and nutrition?
- Can a general multimodal model do enough for a personal prototype?
- How large is the portion-estimation problem in practice?
- What nutrition database should ground recognized foods?
- What should we store so confirmed photo meals become useful later?
- What level of confidence is reasonable for a diabetes context?
- What belongs in the MVP versus later on-device/personalized models?

---

## 1. The problem is really three separate problems

A photo-to-carbs feature looks like one AI task, but it is actually a chain:

```text
1. recognition
   "rice, chicken curry, naan"

2. quantity / portion estimation
   "~180–240 g cooked rice"

3. nutrition mapping
   "that portion contributes roughly X–Y g carbohydrate"
```

Recognition can be strong while the final carb result is poor if the quantity is wrong.

This is the central design implication from both the research datasets and current clinical evaluations: **portion uncertainty must be represented explicitly rather than hidden behind a single number.**

---

## 2. Clinical evidence says “second opinion”, not “automatic truth”

Recent 2026 studies are encouraging but also show why a conservative UX is necessary.

### Image-only estimation can work well for simple foods, but mixed meals are much harder

A 2026 study evaluating ChatGPT-4o for carbohydrate estimation in adolescents with type 1 diabetes used 120 meals/food portions and images with and without size references.

For fruits and vegetables, estimates were frequently within ±10 g of reference carbohydrate. For composite meals, performance was substantially worse: fewer than half were within ±10 g in the reported evaluation.

That is exactly the failure mode relevant to real life: a banana is relatively constrained; a bowl of pasta with sauce, cheese and an uncertain portion is not.

### Richer context can dramatically improve estimates

Another 2026 study found much lower carbohydrate error when multimodal models were given **detailed meal descriptions alongside photos** compared with image-only estimation.

That suggests our best UX is not necessarily “photo and magic”. A tiny amount of confirmation can be high leverage:

```text
AI: "Looks like chicken curry, basmati rice and naan. Is that right?"
User: one tap / quick correction
```

or:

```text
AI: "Rice portion looks medium–large."
User: Small / Medium / Large
```

One extra interaction may improve usefulness far more than building an elaborate computer-vision pipeline.

### Tail errors matter more than average error

A separate 2026 comparison of dietitians and several multimodal models found meaningful differences between models and a non-trivial rate of large carbohydrate overestimates.

For this product, a 20–30 g miss is more important than a good average MAE. Evaluation should therefore include **catastrophic error rates**, not merely mean error.

---

## 3. Open-source route: useful research assets, not a turnkey MVP

### Nutrition5k — the most relevant open dataset

Nutrition5k is the strongest directly relevant open dataset found.

It contains roughly 5,000 real plated meals with:

- food images/video;
- ingredient lists;
- per-ingredient mass;
- total meal mass;
- calories;
- fat;
- protein;
- carbohydrate;
- RGB-D imagery for a subset/setup.

The dataset is licensed CC BY 4.0, which is comparatively friendly for research and later model development with attribution.

Its most important lesson is architectural: models with depth/portion information perform materially better than image-only direct nutrition prediction, and models given true portion mass perform better still.

**Implication:** Nutrition5k is valuable for benchmarking or training a later portion/nutrition model, but it does not eliminate the portion-estimation problem.

Limitations for our immediate use:

- very large dataset;
- cafeteria-biased collection rather than comprehensive world cuisine coverage;
- archived official repository as of 2026;
- no polished iOS/Core ML model that we can simply drop into xDrip.

**Call: keep as a future training/evaluation asset; do not build MVP around it.**

### Food-101 — recognition only

Food-101 contains 101 food categories and 101,000 images.

It is useful as a classic food-image classification benchmark, but it does not provide the quantity/nutrition ground truth needed for carb estimation. A classifier that tells us “pizza” does not tell us whether the user ate one slice or four.

Its image licensing/provenance is also less attractive for a future distributed commercial product than a deliberately permissive nutrition dataset.

**Call: useful baseline/reference, not a carb-estimation solution.**

### FoodSAM — useful segmentation research

FoodSAM is an Apache-2.0 open-source research implementation for food segmentation, combining segmentation and detection components.

Potential future value:

```text
photo
→ identify separate plate regions/items
→ estimate relative area/geometry
→ improve per-component portion estimate
```

But it is a Python research stack with substantial model dependencies, not an iOS-ready component. Segmentation alone also does not solve real-world volume/mass estimation.

**Call: watch/reuse ideas later; not MVP.**

### Recent portion-estimation research

Recent 2026 work is explicitly targeting the weakness of general multimodal models by adding geometry/portion estimation heads. One July 2026 preprint reports substantial portion-error reductions by adding a DINOv2-based portion head to a frozen multimodal model.

Another 2026 project, NutriMLLM, is training multimodal models specifically for nutrient estimation using a large synthetic image-description-nutrient dataset.

These are promising signals that specialized open models may become more attractive soon. Neither currently provides the stable, packaged, iPhone-ready dependency we need for the first personal prototype.

**Call: watchlist. Reassess before any later on-device model investment.**

---

## 4. On-device Core ML is technically viable later

Apple supports:

- training image classifiers with Create ML;
- converting PyTorch models to Core ML with `coremltools`;
- running Core ML models locally using Apple hardware accelerators.

So an eventual privacy-first architecture is technically plausible:

```text
photo
→ on-device recognition / segmentation / portion model
→ local nutrition lookup
→ estimate
```

But Core ML solves **deployment**, not the hard research problem. We would still need a good model, suitable training data, validation and ongoing maintenance.

Therefore “let's make it on-device” should not accidentally become “let's spend months training a food model before validating whether users value the feature.”

---

## 5. Specialist API option: LogMeal

LogMeal is the most complete specialist food-computer-vision API found.

Its current APIs cover:

- food recognition;
- multi-dish segmentation;
- dish/ingredient identification;
- nutrition information;
- quantity estimation on eligible plans.

That makes it attractive as a benchmark because it tackles the whole recognition → quantity → nutrition chain rather than requiring us to assemble each stage.

### Strengths

- purpose-built for food;
- segmentation for multiple foods on one image;
- structured nutrition outputs;
- explicit quantity-estimation capability;
- trial makes it possible to test before committing.

### Weaknesses

- proprietary/network dependency;
- quantity estimation sits behind a higher product tier;
- ongoing price/terms risk;
- vendor output should not define our data model.

**Recommendation:** use LogMeal as an optional specialist benchmark during prototype evaluation, particularly if general multimodal models struggle with multi-item plates or quantity.

---

## 6. Specialist API option: Edamam Vision

Edamam's Food Database API now includes image-based Vision functionality that combines image recognition with its nutrition-analysis database.

It can return recognized dishes/items, ingredient information, quantities and nutrient data.

At current published pricing, the entry-level paid tier is inexpensive enough for a personal prototype and includes a limited number of Vision requests each month. Higher tiers offer larger quotas/pay-as-you-go options.

### Why it is attractive for us

- recognition and nutrition grounding are already integrated;
- very low engineering burden for an experiment;
- relatively affordable for personal testing;
- produces structured outputs rather than requiring us to parse free-form prose.

### Important constraint: data-use/caching terms

Edamam's plans place restrictions on what API data can be cached/stored, with broader caching rights tied to plan/add-on terms. Attribution requirements also apply.

That means Edamam should **not become the canonical schema for our meal history**.

A safer architecture is:

```text
Edamam suggestion
→ user confirms/edits
→ our app stores the USER-CONFIRMED food/carb fact
```

rather than blindly persisting the complete vendor response indefinitely.

Before any public distribution, current API terms would need to be rechecked.

**Recommendation:** excellent first specialist provider to benchmark because the personal-prototype cost/complexity is low.

---

## 7. Nutritionix — useful fallback, not the photo engine

Nutritionix/Syndigo offers strong natural-language food parsing, a large branded-food database, barcode/UPC data and exercise parsing. It also has region support relevant to UK branded foods.

We did not find a current general food-photo recognition endpoint comparable to Edamam Vision or LogMeal.

Potential later role:

```text
AI recognizes "Tesco chicken katsu curry"
→ Nutritionix branded search / text parsing
→ retrieve a more precise labelled product match
```

That could be useful for packaged/branded meals, but it is not the first photo-analysis provider.

---

## 8. USDA FoodData Central — excellent open nutrition grounding

FoodData Central provides official food-search and food-detail APIs and publishes its data under CC0/public-domain terms.

This makes it attractive as a provider-independent nutrition reference.

Potential pipeline:

```text
vision model: "cooked basmati rice, ~200 g"
        ↓
FoodData Central search
        ↓
known carbohydrate per 100 g
        ↓
portion-based carb range
```

Advantages:

- open/public-domain data;
- stable structured API;
- no proprietary model lock-in;
- good fit for generic ingredients.

Limitations:

- primarily US-oriented datasets/brands;
- food-name matching still requires judgement;
- it does not recognize photographs or estimate portions.

**Recommendation:** strong open grounding source, especially for generic foods/ingredients. It can coexist with specialist APIs rather than requiring an either/or choice.

---

## 9. General multimodal models are the fastest flexible prototype

Modern multimodal APIs can accept images and return structured outputs. That means we can ask a model for a machine-readable result such as:

```json
{
  "components": [
    {
      "food": "basmati rice",
      "portion_g_low": 160,
      "portion_g_high": 230,
      "confidence": "medium"
    },
    {
      "food": "chicken curry",
      "portion_g_low": 180,
      "portion_g_high": 260,
      "confidence": "medium"
    }
  ],
  "uncertainties": [
    "rice depth is difficult to judge",
    "sauce may contain sugar"
  ]
}
```

That flexibility is extremely useful during research because we can change the schema/prompt without retraining a model.

### General-model strengths

- excellent zero-shot recognition;
- understands mixed dishes and natural descriptions;
- can state assumptions/uncertainty;
- structured JSON output;
- very little implementation work;
- easy to swap model/provider behind a protocol.

### Weaknesses

- portion estimation remains uncertain;
- direct nutrition numbers can be hallucinated or inconsistent;
- network/privacy dependency;
- provider/model behaviour can change;
- confidence generated by a language model is not automatically calibrated probability.

**Recommendation:** use a general image-capable multimodal model as one of the first benchmark providers, but do not rely on its internal nutrition knowledge alone. Ground recognized foods against an external nutrition source where practical.

---

## 10. Proposed provider-neutral architecture

The app should not know whether the image is being processed by OpenAI, Edamam, LogMeal or a future local Core ML model.

Conceptually:

```swift
protocol MealVisionProvider {
    func analyze(_ image: Data) async throws -> MealVisionEstimate
}
```

Normalized output:

```text
MealVisionEstimate
- provider
- providerModelVersion
- components[]
    - label
    - portionEstimate / range
    - carbEstimate / range
    - confidence
    - assumptions
- totalCarbEstimate
- totalCarbRange
- overallConfidence
- uncertainties[]
```

Provider implementations can evolve independently:

```text
OpenAI / another MLLM
Edamam Vision
LogMeal
future Core ML model
```

This is important because vendor performance and pricing are moving quickly. Provider lock-in would be needless architecture debt.

---

## 11. Recommended MVP interaction

### Step 1 — camera action immediately records `Ate`

Do not wait for AI.

```text
user photographs meal at 13:05
→ Ate event is persisted at 13:05
→ image analysis begins
```

If the request fails or the user closes the app, we still know a meal happened.

That protects the core Attention Engine use case from an optional AI dependency.

### Step 2 — AI recognizes components and estimates portions

Example internal result:

```text
- pasta with tomato/meat sauce
- garlic bread
- parmesan
```

with explicit portion assumptions.

### Step 3 — ground nutrition

Where possible, map recognized components to:

- specialist provider nutrition data; or
- an open source such as FoodData Central.

### Step 4 — display uncertainty honestly

Preferred UX:

> **Looks like pasta + garlic bread**  
> Estimated carbs: **55–75 g**  
> Biggest uncertainty: pasta portion

Not:

> **67 g carbs**

The point estimate can exist as a secondary convenience, but the UI should not imply precision the image cannot support.

### Step 5 — one-tap confirm/edit

Primary actions:

- `Looks right`
- `Edit`

Editing should favor quick high-impact corrections rather than a nutrition-app form.

Examples:

- change recognized food;
- adjust carbs;
- Small / Medium / Large portion;
- remove an incorrectly recognized component.

### Step 6 — create treatment only after confirmation

The unconfirmed model output is **not** a carbohydrate treatment.

After confirmation:

```text
confirmed carbs
→ create normal xDrip Carbs TreatmentEntry
→ optionally sync Nightscout
→ preserve provenance linking it to MealPhotoAnalysis
```

If user never confirms, `Ate` remains useful context while carb amount stays unknown.

---

## 12. Store provenance and corrections from day one

The confirmed/corrected examples are more valuable long term than the original model predictions.

Conceptual local model:

```text
MealPhotoAnalysis
- id
- mealEventID
- capturedAt
- localPhotoReference? 
- provider
- providerModelVersion
- components[]
- estimatedCarbsPoint?
- estimatedCarbsLow?
- estimatedCarbsHigh?
- confidence
- assumptions[]
- status
    - pending
    - confirmed
    - edited
    - rejected
- userConfirmedCarbs?
- confirmedAt?
- linkedTreatmentID?
```

Why store provider/model/version?

Because later we can answer:

```text
Did model version B actually outperform A for my meals?
Does Edamam require fewer edits than the general MLLM?
Which model is systematically wrong for takeaway curry?
```

Do not automatically persist complete raw provider payloads unless provider terms and privacy policy clearly permit it. Normalize only what the app needs.

Photos remain private local health-context data and must never enter Git.

---

## 13. Personal benchmark before provider commitment

The most useful next step when this feature reaches prototype stage is not another vendor comparison spreadsheet. It is a **small personal dataset**.

Test roughly 50–100 representative meals across at least two providers.

Where convenient — not every meal — obtain stronger reference information using:

- nutrition label;
- known recipe;
- weighed portion;
- manually verified carb count.

Then score:

```text
food/component recognition correctness
absolute carb error
% within ±5 g
% within ±10 g
large overestimate rate (>20 g)
large underestimate rate (>20 g)
number of user edits required
time from photo → confirmed
confidence calibration
```

Also categorize meals:

```text
simple single food
packaged/branded food
home-cooked mixed dish
restaurant/takeaway
high-carb meal
low-carb meal
sauce-heavy/hidden-carb meal
```

This will tell us much more than vendor marketing benchmarks about what is good enough for this specific user and workflow.

### Suggested first benchmark pair

**Provider A: general multimodal model**  
Fast, flexible, rich explanation/uncertainty.

**Provider B: Edamam Vision**  
Purpose-built recognition + nutrition, currently inexpensive enough for personal testing.

Then trial **LogMeal** if specialist segmentation/quantity performance looks likely to improve mixed meals materially.

Do not hard-code this choice into the app architecture.

---

## 14. Why not train our own model now?

A custom model sounds attractive because it promises:

- privacy;
- no API cost;
- control;
- personalization.

But it creates a large research project before we know whether the feature actually improves diabetes management.

We would need to solve:

```text
training dataset selection/licensing
food taxonomy
mixed-dish segmentation
portion/volume estimation
nutrition mapping
Core ML optimization
validation across real meals
model updates
```

And we would still need a nutrition database.

The better sequence is:

```text
prove photo workflow is useful
→ collect user-confirmed examples
→ benchmark failure modes
→ only then decide whether on-device/custom modeling is justified
```

At that point the user's own confirmed meal history may also become a useful retrieval/personalization signal.

---

## 15. A likely stronger long-term approach: retrieval + recognition

For one person, repeated meals matter.

After enough confirmed photo meals, a future system can ask:

```text
Does this look similar to a meal the user has confirmed before?
```

Example:

```text
photo today
≈ very similar to user's previous breakfast
previous confirmed estimate: 42 g
```

That may be more useful than continually asking a general model to estimate the same familiar breakfast from scratch.

Future stack could become:

```text
photo
↓
visual embedding / meal recognition
↓
known personal meal match?
  yes → prior confirmed meal + adjustment
  no  → general vision provider
↓
user confirmation
```

This is one reason storing corrections/provenance now matters even if prediction personalization is much later.

---

## 16. Safety / product boundaries

### The photo model does NOT dose insulin

Early product language should be explicit:

> Estimated meal carbohydrates: 55–75 g

not:

> Take X units

### Do not let unconfirmed AI data silently change Attention logic too strongly

A model saying “70 g carbs” should not automatically create a carbohydrate treatment or suppress/escalate alerts as if the amount were known.

Safe hierarchy:

```text
photo taken = meal evidence
recognized food = contextual evidence
AI carb range = uncertain estimate
user-confirmed carbs = treatment-quality user input
```

### Confidence is not a guarantee

Provider-supplied/model-generated confidence may itself be poorly calibrated. We should eventually calibrate confidence against the user's confirmed meals rather than trusting a label like “90% confident”.

### Errors should be cheap to correct

The feature succeeds if it reduces thought and typing. If correction takes longer than rough manual estimation, it has failed the product goal even if its model is technically sophisticated.

---

## 17. Reuse / adapt / build / defer

| Area | Call |
|---|---|
| General multimodal image API | **Prototype / benchmark** |
| Edamam Vision | **Prototype / benchmark** |
| LogMeal | **Benchmark if needed** |
| USDA FoodData Central | **Reuse as open nutrition grounding** |
| Nutritionix | **Optional text/branded-food fallback** |
| Nutrition5k | **Future training/evaluation asset** |
| FoodSAM | **Future segmentation research** |
| Food-101 | **Reference only** |
| Custom Core ML food model | **Defer** |
| Direct AI-to-insulin dose | **Avoid** |
| Single exact carb number with no uncertainty | **Avoid** |
| Provider-specific app data model | **Avoid** |
| Provider-neutral `MealVisionProvider` | **Build** |
| Local `MealPhotoAnalysis` + provenance | **Build** |
| Immediate `Ate` event before inference | **Build** |
| Confirmation/edit before Carb TreatmentEntry | **Build** |

---

## 18. Product-spec implications

1. Meal-photo capture must **immediately create an `Ate` event**; AI success is optional enrichment.
2. Image analysis sits behind a **provider-neutral interface**.
3. Initial provider should be selected by a small personal benchmark rather than vendor claims.
4. Output should show recognized meal/components, an **editable carb range**, assumptions and meaningful uncertainty.
5. A carb `TreatmentEntry` is created only after user confirmation/edit.
6. Preserve the original meal event even if carbs remain unknown.
7. Store provider/model provenance and the user's correction for later evaluation/personalization.
8. Do not let an unconfirmed AI carb estimate masquerade as reliable COB.
9. Do not generate insulin-dose recommendations from the photo result.
10. Use open nutrition grounding where useful; keep vendor payload formats outside the core data model.
11. Photos and raw health data remain private and outside Git.
12. Training an on-device model is a later optimization only if API/privacy/cost/performance evidence justifies it.

---

## 19. Open questions handed forward

### Pass 10 — Apple Health + Watch context

Meal-photo research reinforces that the engine will accumulate several signals with very different confidence levels. Exercise should be modeled the same way: distinguish observed HealthKit facts from inferred insulin-sensitivity effects.

### Pass 11 — future personalized data model

Need to formalize:

- model/provider provenance;
- estimated vs user-confirmed values;
- uncertainty ranges;
- corrections;
- personal repeated-meal identity/embedding;
- links between `Ate`, photo analysis, Carb TreatmentEntry and later glucose response.

### Pass 12 — prediction/personalization

Investigate whether repeated personal meals / image embeddings improve estimates more safely than generalized nutrient prediction.

### Pass 13 — safety

Define minimum confirmation language and which estimated fields are allowed to influence Attention decisions before confirmation.

---

## 20. Key sources reviewed

Primary/official resources used for this pass include:

- Google Research **Nutrition5k** dataset and the CVPR 2021 paper.
- ETH Zurich **Food-101** dataset page.
- `jamesjg/FoodSAM` official open-source repository.
- Apple **Core ML / Create ML / coremltools** documentation.
- **USDA FoodData Central** API/data documentation.
- **LogMeal** API and pricing/usage documentation.
- **Edamam Food Database API / Vision** documentation and current plan terms.
- **Nutritionix / Syndigo** API documentation.
- OpenAI official API image-input/model documentation for a general multimodal-provider reference.
- 2026 peer-reviewed/medical literature evaluating multimodal carbohydrate estimation in diabetes/meal settings.
- Recent 2026 preprints on nutrition-focused multimodal models and geometry-enhanced portion estimation.

## Bottom line

**The product does not need to solve food vision from scratch to learn whether meal photos are valuable.**

The best first version is a low-friction capture-and-confirm loop:

> **Photo → Ate immediately → AI suggestion → carb range + uncertainty → one-tap confirmation/edit → confirmed treatment**

That gives the user a useful second opinion today, protects against false precision, and quietly creates the high-quality personal meal dataset that could make a later personalized/on-device system substantially better.