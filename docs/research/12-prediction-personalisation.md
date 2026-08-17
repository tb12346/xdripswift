# Research 12 — Prediction + Personalisation Approaches

**Status:** Complete  
**Research date:** 2026-08-17

## Executive conclusion

**Do not make a sophisticated glucose forecaster the centre of the first personalised product. Personalisation should arrive in layers, starting with transparent calibration and similar-episode evidence, then adding a small supervised model only when the event ledger contains enough trustworthy labels. Exact glucose forecasting should be a supporting signal to the Attention Engine, not the product objective.**

The research reinforces a distinction that matters for this project:

```text
traditional forecasting question
"What will glucose be in 30/60 minutes?"

our product question
"Does this situation need the user's attention now, or is it likely to resolve / already be adequately handled?"
```

Those questions overlap, but they are not the same. A model can have excellent average RMSE and still create a bad alert product through false alarms, delayed warnings or poor performance in important contexts such as after insulin. Conversely, a simpler model that is slightly worse at point forecasting may be much better at deciding when to interrupt the user.

Recommended sequence:

1. **Deterministic Attention Engine + personalised calibration** — keep the first decision layer explainable and replayable.
2. **Similar-episode retrieval** — compare the current situation with the user's own historical episodes and their outcomes.
3. **Small tabular personalised model** — predict task-specific outcomes such as unresolved rise / likely recovery / useful-interruption probability using glucose dynamics plus meal, recorded insulin/IOB, exercise and episode context.
4. **Short-horizon glucose forecaster** — test a lightweight LSTM or comparable sequence model as an additional signal, not the decision authority.
5. **Foundation/Transformer models later** — benchmark, do not assume they beat a smaller personalised model once substantial personal data exists.

Any learned model should output **confidence / uncertainty**, run behind hard safety and data-quality gates, be versioned, and earn promotion through chronological replay plus shadow evaluation. It must never infer a safe insulin dose from observational data.

---

## Questions for this pass

- What exactly should we try to predict?
- How much value comes from personalisation without deep learning?
- Which existing open-source forecasting frameworks are worth reusing for research?
- What does current research say about personalised versus population/foundation models?
- Should we use similar historical episodes directly?
- When do gradient boosting, LSTM, Transformer or foundation models become justified?
- How should uncertainty enter Attention decisions?
- How should models adapt as the user's behaviour changes?
- Can training/inference remain local/private?
- How should we evaluate models so forecasting accuracy does not hide a poor alert experience?

---

## 1. Prediction is not the product objective

The Attention Engine's job is not to win a glucose-forecasting benchmark.

A standard glucose model usually optimises something like:

```text
input:
- last 1–4 hours glucose
- insulin
- carbs
- sometimes activity/time

output:
- glucose at +30 min
- glucose at +60 min

loss:
- MAE / RMSE
```

That can be useful, but our actual downstream decision is closer to:

```text
current episode
→ quiet
→ watch
→ remind
→ escalate
→ resolve
```

For this product, a useful model should help answer questions such as:

- Is this rise persisting rather than merely noisy?
- Given recorded insulin and its phase, is further attention likely to be needed soon?
- Is this post-low rise behaving like previous recoveries that settled without another intervention?
- After a recent meal and little/no recorded insulin evidence, does this resemble past episodes that became prolonged highs?
- Does recent exercise make the current trajectory unusually likely to reverse?
- Have similar episodes historically generated an alert the user found useful or annoying?

### 2026 task-aware evaluation evidence

A May 2026 preprint, **From Prediction to Practice: A Task-Aware Evaluation Framework for Blood Glucose Forecasting**, is unusually relevant to this product. It evaluates glucose forecasters not only on generic prediction error but also on the downstream task.

For early warning it uses:

- event-level recall;
- false alarms per patient-day;
- clinically important slices such as post-bolus periods.

It reports that models that look strong on aggregate forecasting can perform much worse in specific high-risk slices. It also shows that observational forecasting performance does not establish that a model can predict the effect of changing an insulin action.

That directly supports two project principles:

1. **evaluate the alert/attention task itself;**
2. **do not turn an observational glucose predictor into a dose recommender.**

Source: Namazi A, Shakeri H. *From Prediction to Practice: A Task-Aware Evaluation Framework for Blood Glucose Forecasting*. arXiv:2605.00645 (2026).

---

## 2. Personalisation starts before machine learning

It would be a mistake to define "personalisation" as "train a neural network".

Several high-value forms of personalisation are transparent and data-efficient:

### A. Personal baseline / time-of-day pattern

Examples:

- typical glucose change around breakfast time;
- weekday versus weekend behaviour;
- recurrent dawn rise;
- usual post-lunch excursion duration;
- personal frequency of spontaneous recovery from mild rises.

The `mpereiragu/xdripswift-predict` experiment already implements a useful simple baseline:

```text
prediction = decaying recent-trend component
           + personal time-of-day historical-delta component
```

Its server predictor uses:

- weighted linear regression over recent glucose;
- exponential decay of current trend;
- median glucose change at the same time over the prior 14 days;
- weekday/weekend preference;
- stale-data rejection.

This is not sophisticated ML, but it is transparent and genuinely personalised.

### B. Personal bias calibration

The same fork also contains a walk-forward calibration loop that measures prediction bias by time-of-day and sweeps alert parameters against historical outcomes.

That pattern is more important than its exact hypo rules:

```text
fixed base model
+ walk-forward personal evaluation
+ bounded per-user calibration
```

For our project, the calibration target should not be "best RMSE" or a single sensitivity target. It should use the Attention replay metrics from Pass 06:

- interruptions/day;
- repeats/episode;
- lead time;
- low-value alerts;
- post-treatment nags;
- silent recoveries;
- missed meaningful excursions.

### Recommendation

**Ship/calibrate transparent parameters before training a personal neural network.**

This gives us useful personalisation immediately and provides a baseline that future ML must beat.

---

## 3. Similar historical episodes are especially promising

For a single-user application, retrieval of the user's own analogous episodes is attractive because it combines:

- personalisation;
- interpretability;
- low training-data requirements;
- graceful improvement as history accumulates.

Example current state:

```text
8.4 mmol/L
rising persistently for 20 min
meal ~35 min ago
recorded rapid insulin ~18 min ago
IOB active and near peak
aerobic workout ended 2 h ago
```

A retrieval layer could find the most similar prior situations using features such as:

```text
- starting glucose
- 15/30/45 min glucose deltas
- acceleration / shape
- time of day / weekday
- meal recency and confirmed/estimated carbs if known
- recorded insulin recency
- IOB + activity/phase
- recent exercise type/recency
- low-recovery state
- data quality / staleness
```

Then summarize what happened next:

```text
12 similar episodes
- 7 recovered without another logged action
- 3 continued above threshold for >60 min
- 2 had additional insulin recorded
median +30 min glucose change: +0.6 mmol/L
```

This evidence can feed the Attention Engine without claiming causal certainty.

### Why retrieval deserves an early experiment

The emerging **GlyRAG** work uses retrieval of similar historical CGM windows before forecasting and reports improvements over its non-retrieval baseline. Its architecture is far more complex than we need, but it provides current research support for the underlying idea that historical analogues are useful.

For this project we should start much simpler:

```text
normalized episode feature vector
→ nearest neighbours from this user's history
→ outcome summary + distance/confidence
```

No LLM is required in the decision path.

### Important limitation

Similar outcome is not the same as counterfactual outcome.

If a prior episode ended well after the user took insulin, it does not prove the current episode will end well without action. Retrieval should describe prior observations, not recommend a dose or imply causality.

Source: *GlyRAG: Context-Aware Retrieval-Augmented Framework for Blood Glucose Forecasting*. arXiv:2601.05353 (2026 preprint).

---

## 4. The first learned model should probably be tabular and task-specific

Once we have enough Attention events and reviewed episodes, the first supervised model should be deliberately boring.

Candidate model families:

- logistic regression;
- regularized linear models;
- random forest;
- gradient-boosted trees (e.g. LightGBM/XGBoost-style approaches).

### Why tabular first

Our highest-value contextual signals are naturally structured:

```text
current glucose
recent delta / slope / acceleration
persistence
recent insulin age
modeled IOB
insulin activity/phase
Ate age
confirmed/estimated carb context
exercise recency/type
recovery-state flag
hour/day
alert history
prior acknowledgement
staleness/noise flags
```

A tree model can model nonlinear interactions such as:

```text
persistent rise
AND meal recent
AND low recorded IOB
AND no recovery context
```

without requiring months of high-quality dense labels.

Tree/linear models also give us easier feature inspection and ablation than a large sequence model.

### Relevant evidence

A 2024 T1D postprandial-response study used **LightGBM** with CGM, insulin dose, dietary nutrients and patient measurements to predict personalised postprandial glycaemic response. The dataset was small (13 participants), so it is not proof that LightGBM is universally best, but it confirms that structured glucose/insulin/meal features can support a useful personalised tabular model.

Source: *Prediction of personalised postprandial glycaemic response in type 1 diabetes mellitus* (2024).

### Better target than exact future glucose

Potential learned targets for us include:

```text
P(unresolved meaningful rise within next 30–45 min)
P(recovery without new recorded action within next 30 min)
P(cross high threshold within next 30 min)
P(low within next 30 min)
expected glucose delta at +30 min
```

And later, when enough explicit user feedback exists:

```text
P(user would want an interruption now)
```

The final Attention decision remains policy logic, not a direct model output.

---

## 5. Labels: do not train on our own bad habits as if they were truth

The event ledger from Pass 11 is valuable because it gives us several kinds of outcomes, but they have different reliability.

### Useful objective outcomes

Examples:

- glucose eventually crosses a threshold;
- excursion magnitude / area above range;
- glucose starts falling within a time window;
- episode persists for N readings;
- stale-data gap occurs;
- low-recovery episode resolves;
- additional treatment is recorded.

### Stronger subjective labels

The best label for the product question is eventually explicit feedback such as:

- wanted this alert sooner;
- useful alert;
- unnecessary / annoying;
- already handled;
- intentionally waiting;
- no insulin needed.

### Weak labels that must not become truth

Do **not** interpret:

```text
insulin recorded after an alert
```

as automatically meaning:

```text
the alert was correct
```

And do not interpret:

```text
no additional insulin recorded
```

as:

```text
no action was needed / no insulin was taken
```

Those mistakes would train the model to reproduce the exact logging/attention imperfections this product is trying to improve.

### Recommendation

Use a hierarchy of labels:

```text
explicit reviewed/user feedback      → strongest
clear physiological outcome          → strong proxy
structured user action               → contextual proxy
absence of a log                      → unknown, not negative label
```

---

## 6. Short-horizon forecasting remains useful — but as one feature

A forecast can still help the Attention Engine.

Examples:

- predicted crossing of a high threshold;
- predicted recovery toward range;
- expected +30 minute delta;
- widening prediction uncertainty;
- disagreement between physiology-aware policy and statistical forecast.

But we should not let a forecaster replace the contextual engine.

### Personalised LSTM evidence

A 2025 IEEE study, **Personalized Blood Glucose Forecasting from Limited CGM Data Using Incrementally Retrained LSTM**, used incremental personal retraining on T1D data.

Key findings:

- a small stacked LSTM was incrementally updated as each person's data accumulated;
- it used 2 hours of history;
- multivariate variants included CGM, insulin and carbs;
- at a 30-minute horizon, incremental personal retraining improved RMSE relative to the stacked-LSTM comparator on both OpenAPS and Replace-BG datasets;
- the work specifically addresses cold-start and adaptation as personal history grows.

This supports a future architecture where a small forecasting model can gradually adapt rather than needing a giant bespoke model.

Source: Shen Y, Kleinberg S. IEEE Transactions on Biomedical Engineering, 2025;72(4):1266–1277. PMCID: PMC11999170.

### MDI caveat

A 2025 study using T1DEXI data compared modern forecasting approaches for people using pumps and MDI. Personalised approaches improved short-horizon forecasting, but reported forecast precision was lower in MDI users than pump users.

That is relevant to this project: MDI data contains more uncertainty because injections may be omitted or logged late and basal delivery is not observed as continuously as pump telemetry.

This is another reason to keep model certainty explicit and avoid treating a numerical forecast as physiological truth.

Source: *Personalized glucose forecasting for people with type 1 diabetes using large language models*, Computer Methods and Programs in Biomedicine 265 (2025), 108737.

---

## 7. Fancy models do not automatically win

There is now substantial work on:

- Transformers;
- time-series LLMs;
- meta-learning;
- foundation models;
- state-space models;
- retrieval-augmented forecasting.

These are worth tracking, but they do not justify starting there.

### GlucoFM-Bench — June 2026

The very recent **GlucoFM-Bench** preprint compares time-series foundation models with supervised deep-learning models across 15 diabetes-relevant datasets and 1,117 individuals.

Its headline result is useful for our decision:

- pretrained foundation models were strong in zero-shot/few-shot settings;
- the best zero-shot model came within roughly 5% of the best full-shot supervised model;
- **when task-specific training data were abundant, a lightweight LSTM was strongest**, outperforming the foundation models by 4–21% in the reported full-shot comparisons;
- T1D and hypo/hyperglycaemic regions remained harder than aggregate metrics suggest.

This is preprint evidence, not a final clinical validation, but it argues strongly against assuming "bigger/newer model = better personal model".

Source: Lu B et al. *GlucoFM-Bench: Benchmarking Time-Series Foundation Models for Blood Glucose Forecasting*. arXiv:2606.06881 (2026).

### GluFormer — interesting research asset, not our first production model

The 2026 Nature **GluFormer** model is an impressive self-supervised CGM foundation model trained on more than 10 million glucose measurements from 10,812 adults and transferred across diverse cohorts including T1D.

Its open-source repository is Apache-2.0 and supports:

- autoregressive CGM modelling;
- embedding extraction;
- missing-data imputation;
- multimodal dietary tokens;
- downstream metabolic prediction tasks.

However:

- the pretraining cohort was mainly people without diabetes;
- much of the paper's strongest evidence concerns representation learning / longer-term metabolic outcomes rather than our MDI Attention task;
- the reference training stack used large NVIDIA GPUs;
- our personal event context (insulin, Ate, acknowledgements, exercise, recovery state) is more important than a generic CGM representation alone.

So GluFormer should be treated as a **later benchmark / representation experiment**, not a dependency for the first personalised system.

Sources:
- Lutsker G et al. *A foundation model for continuous glucose monitoring data*. Nature (2026).
- `Guylu/GluFormer`, Apache License 2.0.

---

## 8. Uncertainty is mandatory if prediction influences alerts

A point forecast such as:

```text
"9.1 mmol/L in 30 min"
```

looks far more certain than the underlying data warrants.

Uncertainty comes from several sources:

- CGM noise;
- missing/delayed insulin logs;
- unknown meal quantity;
- variable absorption;
- unlogged exercise;
- illness/stress;
- changing insulin sensitivity;
- model extrapolation into rare situations.

Research in personalised T1D forecasting has specifically explored evidential deep learning so the model returns predictive uncertainty alongside glucose estimates. More recent 2026 work continues to study uncertainty-aware LSTM/GRU/Transformer approaches.

We do not need evidential deep learning on day one to respect the principle.

### First implementation could use simpler confidence signals

For example:

```text
PredictionConfidence
- data freshness
- number of similar historical episodes
- nearest-neighbour distance
- whether meal/insulin context is known
- model calibration bucket
- recent out-of-distribution score
- forecast ensemble disagreement (later)
```

### Policy rule

**Low-confidence prediction should not suppress a hard safety/attention rule.**

A learned signal can increase confidence that an episode is resolving or worsening, but uncertainty should push the policy toward conservative interpretation rather than false precision.

Sources:
- Zhu T et al. *Personalized Blood Glucose Prediction for Type 1 Diabetes Using Evidential Deep Learning and Meta-Learning*. IEEE TBME (2023).
- *Uncertainty-aware Blood Glucose Prediction from Continuous Glucose Monitoring Data*. arXiv:2603.04955 (2026 preprint).

---

## 9. Personalisation should adapt slowly and be reversible

Diabetes behaviour is non-stationary.

Patterns can change with:

- exercise routine;
- illness;
- stress;
- travel/time zone;
- seasons;
- insulin type/settings;
- food habits;
- weight;
- sleep;
- changes in logging behaviour.

A model trained once on six months of history can become stale.

But unconstrained online learning after every event is also risky because a few unusual days could shift the model dramatically.

### Recommended lifecycle

```text
new events accumulate
      ↓
periodic candidate retraining / recalibration
      ↓
chronological validation on recent held-out data
      ↓
Attention replay + safety slices
      ↓
shadow candidate vs current model
      ↓
only then promote model version
```

Keep:

- model version;
- training-data time range;
- feature-schema version;
- calibration version;
- evaluation report;
- previous model for rollback.

### Recency

Rather than deleting old data, experiment with:

- rolling training windows;
- recency weighting;
- separate stable baseline + recent residual correction.

That can retain long-term knowledge while adapting to changing routines.

---

## 10. Shadow mode is essential

Before a learned model changes alerts, run it invisibly.

Example:

```text
live Attention policy makes real decision
personal model also scores episode silently
```

Store:

```text
model would have said: escalate
actual policy said: watch
outcome: recovered in 20 min
user feedback: no alert wanted
```

This allows prospective evaluation without exposing the user to an unvalidated personalised policy.

### Promotion criteria

A new model must beat the current policy on the metrics that matter:

- useful attention captured;
- false/low-value interruptions;
- repeated alerts;
- lead time;
- post-treatment nagging;
- recovery-state errors;
- stale/missing-data slices;
- exercise slices;
- low-glucose slices;
- high-glucose slices.

Do not promote based solely on MAE/RMSE.

---

## 11. Use chronological evaluation only

Time-series leakage is particularly dangerous for a personalised model.

Bad split:

```text
randomly shuffle episodes from 6 months
80% train / 20% test
```

This lets nearly identical adjacent days leak routines into both sets.

Prefer:

```text
months 1–3 → initial training
month 4    → validation
month 5    → test
month 6    → prospective shadow period
```

or walk-forward evaluation.

Also ensure that `recordedAt` versus `occurredAt` semantics from Pass 11 are respected: a late-entered meal/workout/insulin event must not appear in historical model inputs before the system actually knew about it.

---

## 12. Evaluate slices, not only global performance

The 2026 task-aware benchmark's post-bolus findings reinforce the need for slice evaluation.

For our product, every candidate model/policy should report at least:

```text
all episodes

meal / no meal
recorded insulin / no recorded insulin
high IOB / low IOB
insulin rising-to-peak / insulin tail
recent exercise / none observed
low recovery / ordinary rise
morning / daytime / overnight
fresh CGM / borderline stale
high variability / stable periods
known carbs / unknown carbs
manual exercise / HealthKit exercise
```

A model that improves average performance while getting **post-insulin rising episodes** worse may be unacceptable because that is exactly where we are trying to avoid unnecessary nagging without missing a genuinely worsening situation.

---

## 13. Open-source research tooling we can reuse

### GluPredKit — strongest practical research dependency found

`replicahealth/GluPredKit` is a mature Python framework specifically for blood-glucose prediction research.

Useful features already present:

- MIT license;
- Nightscout parser;
- parsers for OpenAPS, T1DEXI, OhioT1DM, Tidepool and Apple Health;
- models including zero-order, ridge, weighted ridge, random forest, SVR, LSTM, double LSTM and TCN;
- Loop/pyloopkit-derived model adapters;
- glycaemia-detection metrics;
- Clarke/Parkes error grids;
- temporal-gain metrics;
- trajectory/event plots;
- reproducible train/test/report pipeline.

Its Nightscout parser already fetches:

- CGM entries;
- treatments;
- profiles;

and normalizes glucose, carbs, bolus/basal insulin and time features onto a 5-minute grid.

### How to use it here

**Do not put GluPredKit in the iOS runtime.**

Use it as an offline research harness alongside our Swift Attention replay system:

```text
Nightscout/private export
        ↓
GluPredKit
→ benchmark forecast models
→ compare horizons / clinical errors
→ export candidate predictions

same history
        ↓
Swift Attention replay
→ measure actual attention/alert value
```

This separates:

- forecasting research;
- production Attention policy;
- iPhone runtime dependencies.

It may save us substantial effort when we later ask whether a personal LSTM materially beats the simple predictor.

Sources:
- `replicahealth/GluPredKit`
- MIT License
- JOSS: *GluPredKit: A Python Package for Blood Glucose Prediction and Evaluation* (2024).

### GlucoBench / newer benchmark suites

GlucoBench and 2026-era GlucoFM/GlucoTune work are useful references for reproducible preprocessing and fair model comparison. They reinforce a broad lesson: preprocessing choices, prediction horizon and population/slice selection can materially change which model looks best.

We should borrow benchmark discipline more than any single architecture.

---

## 14. The existing xDrip prediction fork: reuse ideas, not architecture

The `mpereiragu/xdripswift-predict` fork remains useful research.

### Reuse/adapt

- recent-trend weighted regression;
- stale-data refusal;
- time-of-day personal median deltas;
- weekday/weekend pattern preference;
- walk-forward historical calibration;
- bias measurement by time slot;
- outcome tracking;
- bounded auto-tuning concept.

### Do not copy directly

- server-first architecture as a requirement;
- exact predicted-hypo alert policy;
- separate Python and Swift copies of decision logic;
- model parameters optimised only for predictor accuracy/sensitivity;
- generic AI/LLM prose in the real-time decision path.

The most valuable part of the fork is its philosophy:

> start with a transparent baseline, replay it on personal data, measure systematic error, and calibrate slowly.

That aligns well with this project.

---

## 15. On-device inference and later on-device learning are feasible

For privacy and reliability, production inference should ideally run locally when the model is small enough.

Apple Core ML supports:

- local model inference;
- traditional regression/tree/neural models;
- user-specific **on-device model updates** for models marked as updatable via `MLUpdateTask`.

This means a future personalised model does not inherently require uploading raw health history to a remote training service.

### But do not start with on-device training complexity

Recommended first research workflow:

```text
private Nightscout/local data
→ offline training/evaluation tool
→ frozen candidate model/parameters
→ local iPhone inference
```

Only consider `MLUpdateTask` after the model/target and update cadence are proven. Continuous on-device adaptation adds validation, rollback and debugging complexity even if Apple's APIs make it technically possible.

Source: Apple Core ML documentation — *Personalizing a Model with On-Device Updates* / `MLUpdateTask`.

---

## 16. Proposed personalised-signal architecture

Keep learned components behind a narrow interface.

Illustrative domain boundary:

```swift
struct PersonalisedAttentionSignals {
    let riskOfMeaningfulRise30m: ProbabilityEstimate?
    let riskOfLow30m: ProbabilityEstimate?
    let probabilityOfRecovery: ProbabilityEstimate?
    let expectedDelta30m: DistributionEstimate?
    let similarEpisodeSummary: SimilarEpisodeSummary?
    let confidence: ModelConfidence
    let modelVersion: String
}

protocol PersonalisedSignalProviding {
    func signals(for context: AttentionContext) async -> PersonalisedAttentionSignals
}
```

Then:

```text
AttentionContext
      │
      ├──── deterministic rules / hard gates
      │
      └──── PersonalisedSignalProvider
                    │
                    ├─ retrieval
                    ├─ calibrated risk model
                    └─ optional glucose forecaster
      │
      ↓
AttentionEngine
→ quiet / watch / remind / escalate / resolve
```

### Critical property

The `AttentionEngine` remains capable of making a conservative decision when:

- the model is unavailable;
- the model is stale;
- there is not enough history;
- confidence is low;
- a model version fails to load;
- external/private training has not run.

Personalisation is enhancement, not a correctness dependency.

---

## 17. A realistic staged experiment plan

### Stage A — no learned model

Compare:

1. current xDrip/Zukka baseline;
2. contextual deterministic Attention policy;
3. contextual policy + personal time-of-day calibration.

### Stage B — retrieval

Add similar-episode outcomes.

Ablate:

```text
+ glucose shape only
+ insulin/IOB
+ meal state
+ exercise
+ time of day
```

Question:

> Do similar personal episodes improve quiet/remind/escalate decisions without generating extra interruptions?

### Stage C — tabular task model

Train a simple model on objective proxy outcomes and later reviewed labels.

Compare:

- logistic/ridge;
- random forest;
- gradient boosting.

Measure calibration, not just discrimination.

### Stage D — forecasting model

Use GluPredKit to compare:

- zero-order;
- linear/ridge;
- SVR/random forest;
- small LSTM/TCN;
- simple xDrip prediction baseline.

Candidate horizon:

- start with **30 minutes**;
- evaluate 60 minutes only if it adds Attention value.

Do not assume a longer horizon is better merely because it provides earlier warning; uncertainty rises with horizon.

### Stage E — advanced models

Only after the simpler stages plateau:

- incrementally retrained LSTM;
- Transformer;
- foundation-model embeddings;
- retrieval-augmented sequence model;
- meta-learning / transfer learning.

Every stage must beat the simpler model on the **Attention objective**, not only forecasting metrics.

---

## 18. What not to do

### Do not build a personalised insulin-dose model

Observational data contains actions selected by the user under historical conditions. It does not tell us safely what would have happened under a different insulin dose.

The 2026 task-aware forecasting work is a strong reminder that good observational forecast accuracy does not imply correct counterfactual action-response prediction.

### Do not train directly on "alert → insulin" as success

That would reward interrupting the user until they act, even if the alert was unnecessary.

### Do not let ML hide missing data

If insulin, carbs, exercise or CGM are unknown, the model should receive explicit missingness/provenance rather than imputed false facts.

### Do not retrain after every bad day

Personalisation should adapt, but candidate updates need replay/validation before promotion.

### Do not expose false precision

A forecast or risk score can stay internal. The user-facing product can still say:

```text
"Glucose is continuing to rise and I can't see recent insulin logged."
```

rather than:

```text
"There is an 82.7% probability you'll be 12.4 mmol/L in 37 minutes."
```

---

## 19. Reuse / adapt / build / ignore

### Reuse

- GluPredKit as an **offline** MIT-licensed model/forecast evaluation toolkit.
- Its Nightscout parser concepts and standard forecasting metrics.
- Existing xDrip-predict fork's transparent recent-trend/time-of-day baseline concepts.
- Core ML for eventual local inference.

### Adapt

- xDrip-predict walk-forward auto-calibration to our Attention metrics.
- similar-episode retrieval into a simple personal nearest-neighbour system.
- modern glucose-forecast benchmarks into task/slice-aware evaluation.
- LSTM incremental-personalisation pattern if a sequence model later proves useful.

### Build

- task-specific Attention labels and outcomes;
- `PersonalisedSignalProviding` boundary;
- personal similar-episode index;
- model-confidence/data-sufficiency representation;
- candidate/shadow/promotion model lifecycle;
- Attention-specific metrics and safety slices;
- versioned local model metadata.

### Ignore/defer

- direct insulin-dose recommendation;
- reinforcement-learning treatment selection;
- giant foundation model in iOS MVP;
- LLM in real-time physiological decision path;
- unbounded online learning;
- generic "AI insight" prose as a substitute for deterministic analysis;
- optimizing solely for RMSE/MAE.

---

## 20. Product-spec implications

1. **The Attention Engine remains the decision authority.** Learned models provide contextual signals, never direct actions.
2. **First personalisation is deterministic/calibrated.** Time-of-day patterns, personal bias and episode outcomes can add value before ML.
3. **Similar-episode retrieval should be an early experiment.** It is personal, interpretable and data-efficient.
4. **The first supervised model should be small and task-specific.** Start with tabular models before deep sequence models.
5. **Forecasting is secondary.** If added, start around 30 minutes and prove incremental Attention value.
6. **Uncertainty is first-class.** Missing context and out-of-distribution situations reduce confidence rather than disappearing through imputation.
7. **No causal insulin claims.** Observational prediction cannot safely answer "what if I take X units?"
8. **Models adapt through candidate → validation → shadow → promotion.** No uncontrolled live learning.
9. **Model provenance is stored.** Version, training window, feature schema and calibration version must be available for replay/debugging.
10. **Evaluation is chronological and slice-aware.** Include IOB, post-meal, exercise, recovery, stale-data and time-of-day slices.
11. **Attention metrics outrank forecasting metrics.** Interruption burden, repeated nags and useful lead time are primary.
12. **Offline research can use GluPredKit.** Keep Python model experimentation outside the iOS runtime and the Swift Attention decision logic.
13. **Local inference is preferred.** Core ML can support future on-device inference and, later, controlled personal updates.
14. **The app must work with no personal model.** Personalisation can fail or be unavailable without breaking safe base behaviour.

---

## Open questions to carry into implementation/spec

- What exact outcome definition should the first task-specific model use: prolonged excursion, threshold crossing, recovery, or a composite episode score?
- How many reviewed episodes are needed before subjective "wanted attention" labels are trustworthy enough to train on?
- Which distance metric and feature normalization work best for similar-episode retrieval?
- Should retrieval use a rolling time window so very old episodes have lower influence?
- Does a 30-minute forecaster add anything once current slope, IOB, meal state and retrieval evidence are present?
- Can a simple gradient-boosted model match/beat LSTM for the Attention target?
- What uncertainty/calibration method is sufficient for a first learned risk score?
- How often should candidate models retrain, and how much recent data should be held out?
- Should model training remain entirely local/offline for the personal prototype, or is a private optional service worth the operational simplicity later?

---

## Research recommendation

The first personalised system should look less like a generic AI glucose forecaster and more like a **layered personal evidence system**:

```text
current physiological/context facts
          ↓
transparent Attention rules
          +
personal historical calibration
          +
similar past episodes
          +
small calibrated learned risk model (later)
          +
short-horizon glucose forecast (only if additive)
          ↓
quiet / watch / remind / escalate / resolve
```

The core principle is:

> **Personalise the decision about attention before trying to perfectly predict glucose.**

That best matches the user's real problem and gives every more sophisticated model a clear bar it must beat.
