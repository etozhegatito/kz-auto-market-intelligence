# Verified findings and failed experiments

This document records negative results as first-class project output. A failed
experiment narrows the search space, protects future work from repetition, and
is more useful than an unexplained metric improvement.

Metrics below refer to the historical snapshot on which each experiment was
run. The current production numbers live in [MODEL_CARD.md](MODEL_CARD.md).
Withdrawn CV figures are retained only to explain why they must not be cited.

## Executive summary

| Question | Finding | Decision |
|---|---|---|
| Do more listing-table features lower MAPE? | Four feature groups added noise. | Keep the 13-feature contract. |
| Does seller text help? | Sparse full-data text hurt; fully enriched cheap rows showed a local gain. | Improve unbiased coverage and close train/serve skew first. |
| Do generic image embeddings improve price? | ResNet50 and full-frame CLIP did not beat tabular baselines. | Do not add them to production price inference. |
| Does more same-source listing data solve the error? | Repeated growth moved MAPE little and the baseline caught up. | Treat listing-card data as plateaued. |
| Where is the error? | Old vehicles below 5M tenge dominate. | Prioritize physical-condition evidence. |
| Is the anomaly detector a fraud classifier? | One correlated two-listing fraud case is confirmed; this is far too little for stable recall. | Keep human review and random controls. |
| Is supervised photo damage ready? | No; definitions are repaired, but only 18 independent positive listings remain. | Expand to roughly 200 positives before evaluation. |

## 1. Additional tabular feature groups did not help

Four groups were tested beyond the stable 13-feature schema: richer technical
attributes, publication metadata, interaction-style transformations, and
enrichment-derived fields. They either left grouped MAPE unchanged or made it
worse. Sparse categories increased variance and coverage patterns encoded the
pipeline's own selection policy.

Decision: keep a compact feature contract unless a new field has adequate
coverage, exists at inference, and improves grouped plus temporal validation.

## 2. Image embeddings failed for different reasons

### 2.1 ResNet50 embeddings focused on the wrong signal

Generic ImageNet features mainly represented scene, angle, colour, and vehicle
type. They added dimensionality without useful price-condition information and
did not lower duplicate-safe validation error.

### 2.2 Cover-photo CLIP asked the wrong frame

The first image is usually selected to sell the car, not reveal its defect.
Encoding only the cover therefore tested marketing-photo composition rather
than vehicle condition.

### 2.3 All-frame CLIP found signal but not incremental value

Aggregating the gallery made age and broad condition more visible, but the
features did not improve price estimates beyond age and price-related tabular
information. A model can detect a visual pattern without adding product value.

Decision: full-frame embeddings are not production price features. Localized
damage detection remains a separate, narrower research question.

## 3. Target-derived features are forbidden

Kolesa's reference average price and price category make validation look better
because they derive from the marketplace target. They are useful for anomaly
cross-checks and stratification, never for training the price estimator.

## 4. Repeated-digit mileage was not a reliable fraud rule

Rounded or repeated mileage values looked suspicious by intuition, but review
did not show sufficient association with deception. Treat them as ordinary data
quality, not fraud evidence.

## 5. The current ceiling is a signal problem

At 5M tenge and above, MAPE is about 16%. Below 5M it is about 29%. The largest
intersection is age 21+ and price below 5M. Listing-table data describe vehicle
identity well but not corrosion, crash repair, mechanical state, or restoration
quality. Hyperparameter tuning cannot recover a signal that is absent.

## 6. Zero confirmed fraud was still a measurement

At this historical stage, the manual sample contained no confirmed fraud. That
did not prove zero prevalence. Sixty-five random controls with zero positives
implied a rough one-sided 95% upper bound of `3 / 65 ≈ 4.6%` by the rule of
three. The later complete audit is reported in Finding 32.

Decision: describe candidates as anomalies, retain `unknown`, and continue
random-control review before claiming recall or prevalence.

## 7. More data of the same type reached a plateau

Several collection rounds substantially increased rows but moved grouped MAPE
by tenths or hundredths of a percentage point. One larger growth stage improved
MAPE by about 1.06 points, but later rounds produced almost no stable gain.

The simple make/model/year baseline improved faster as the market grid became
denser. By median APE it approached the CatBoost model. The ML model's remaining
advantage is mainly in the tail.

Decision: fresh collection remains useful for drift and market coverage, but
deep pagination of the same fields is not the path to 18% MAPE.

## 8. Changing the loss was not a free improvement

Training closer to absolute error improved the median case but worsened average
percentage error. That is a trade-off, not a universal improvement. The chosen
objective must match the product metric and segment priorities.

## 9. A global multiplier improved MAPE by adding bias

Multiplying all predictions by approximately 0.95 reduced MAPE by about 0.52
percentage points on one evaluation, but systematically underpriced vehicles.
This exploits asymmetry in percentage error rather than learning a better fair
price.

Decision: reject metric gaming that worsens product calibration.

## 10. Interval tail imbalance was a conditioning artifact

When grouped by actual price, cheap vehicles appeared more often below the
lower bound than above the upper bound. This is expected: selecting rows with a
low actual target preferentially selects cases the model overpredicted.

Grouping by predicted price—the quantity available at inference—produced
balanced tails. The interval implementation was not defective.

## 11. Data growth helps, but the baseline also benefits

The model improved as the dataset grew, yet the simple group baseline closed the
gap faster for typical rows. “More data helps” and “the model's relative value
shrinks” can both be true.

Population Stability Index checks showed no major covariate shift during the
relevant window. The plateau was not explained by a dramatic population change.

## 12. Enrichment is valuable for anomaly exculpation

Detail-page context removed roughly 40% of some rule-positive suspicions by
revealing disclosed damage, non-running badges, instalment terms, or seller
explanations. This is a real product improvement even when price MAPE does not
move.

A later bug fix ensured that exculpation applies to residual-model candidates as
well as rule candidates.

## 13. The first photo conclusion was too broad

“Photos do not help” was incorrect. The actual result was narrower: full-frame
image features did not add price value beyond age and price-related features.
Separate zero-shot axes detected rust and dirt well in historical evaluation.

A proposed claim that better photos cause more views failed because the
observational data could not separate photo quality from vehicle and seller
confounders. No seller advice should promise more views from that test.

## 14. Geographic expansion was rejected

Adding Astana, Shymkent, or Karaganda would mix different demand, income,
logistics, and price regimes. With only a few thousand rows per city, a city
feature would mostly learn offsets while sparse make/model/year/city cells
remain weak.

The Almaty listing pool was also not exhausted. Expansion would change the
product from an Almaty estimator to an under-specified national estimator.

Decision: keep one city until a separate multi-market design is justified.

## 15. The objective was reframed after the plateau

The old intuition that roughly 17,000 similar rows would automatically reach
18% MAPE was unsupported. The measured target became segment-specific:

```text
vehicles at 5M+ KZT: keep MAPE below 17%    achieved at about 16%
vehicles below 5M:   reduce ~29% toward 20.5%
```

If the stronger segment remains unchanged, the latter improvement is roughly
what the weighted arithmetic requires for 18% overall.

## 16. Tiling showed that impact damage is local

Historical zero-shot damage AUC increased from 0.776 on whole frames to 0.827
when the maximum tile score was used. Rust moved in the opposite direction,
from 0.881 to 0.809. Dents are local; rust often covers a larger body area.

Decision: collect localized impact boxes and keep rust as a separate signal.

## 17. Enrichment can help price in the right subset

An early measurement incorrectly concluded that enriched fields had no value
because it compared mismatched samples. On a fully enriched cheap subset,
seller text and options improved the local result by about 3.4 percentage
points. On the full dataset with only about 12% useful enrichment coverage, the
gain shrank to about 0.05 points.

Coverage was also non-random: selected suspicious rows had shorter and
different text. Presence of enrichment could therefore encode pipeline policy.

Decision: improve balanced coverage and expose the same inputs at serving time
before adding these features.

The detail-page audit also showed that structured `page_condition` is not always
present and public pages do not reveal the VIN. The parser now stores only
positive evidence for a vehicle-history/VIN-backed flag and never stores VIN.

## 18. A fifth measurement confirmed the plateau

An approximately 18% increase in training data moved overall MAPE by about 0.19
percentage points while median error worsened. The baseline's median approached
the model's median, reinforcing that CatBoost's value was concentrated in hard
tails rather than typical rows.

## 19. A supervised damage score looked strong for the wrong reason

An early classifier produced an attractive overall AUC, but age plus price alone
explained much of it: damaged vehicles in the labelled sample were simply older
and cheaper. Inside the inexpensive segment, image performance did not clearly
beat the tabular baseline.

Lesson: compare every CV model with a strong non-image baseline and evaluate by
independent listings, not frames.

## 20. Bounding-box crops helped but did not clear the gate

On the historical labels, whole-frame CLIP in the cheap segment had AUC around
0.607, tile/crop aggregation around 0.633, and age plus price around 0.704.
The no-body axis performed strongly on manually identified interiors, supporting
queue ordering rather than exclusion.

An annotator also found that boxes drawn around rust were silently discarded
when the frame label was `intact`. The journal now preserves boxes with every
label and requires them only for `damaged`.

All figures in this section are historical and not current claims because the
positive labels were later quarantined for definition drift.

## 21. Part of cheap-segment MAPE is metric arithmetic and target noise

The same absolute tenge miss creates a larger percentage error on a cheap car.
Advertised prices also contain negotiation margins, missing condition, and
occasional non-comparable terms. This creates an irreducible floor unless the
target or evidence improves.

A separate cheap-segment model improved overall MAPE by roughly 0.25 percentage
points in its first honest experiment—useful but far from the required gain.

## 22. The cheap specialist became a valid production route

The first experiment routed by actual price, which is unavailable in production
and therefore invalid. The corrected route uses the general model's prediction
below 5M and trains the specialist on a wider actual-price band below 8M.

The current snapshot shows only -0.03 percentage points grouped improvement,
with a confidence interval crossing zero. The route is leakage-safe, but its
overall benefit is not statistically established on the latest data.

## 23. Reproducible full-frame CV remained negative

Photo evaluation was rebuilt with grouped folds, one prediction per listing,
paired bootstrap, and both ROC-AUC and PR-AUC. CLIP did not establish an
incremental benefit over age plus price. PCA was also moved inside each fold to
prevent distribution leakage from the test fold.

## 24. Active learning cannot define the final test set

Model-ranked frames are intentionally enriched for likely positives. Treating
them as a test set would overstate real-world prevalence and entangle evaluation
with the current model. New listings now receive a deterministic random audit
split before ranking. Legacy labels remain training-only because they already
influenced experiments.

Exact-photo pHash components are grouped in addition to `ad_id` so copied images
cannot cross train/test boundaries under different listings.

## 25. Another 461 rows moved MAPE by only 0.13 points

A later collection round added 461 training rows. Overall MAPE improved from
about 21.52% to 21.39%, while the under-5M segment barely changed. This is within
the broader plateau pattern and does not justify more deep pagination as the
main strategy.

## 26. The hard segment is not “all vehicles older than five”

Measured age bands showed approximately 16% MAPE for 6–10 years, 18% for 11–20,
and about 30% for 21+. The age-21+-and-below-5M intersection represented about
28% of rows but approximately 41% of all percentage error.

Decision: target that intersection rather than describing every vehicle older
than five years as equally difficult.

## 27. Another 149 rows left overall MAPE unchanged

A fresh block increased training rows from 11,991 to 12,140. Routed grouped MAPE
moved from 21.3927% to 21.3951%, effectively zero; median APE worsened by about
0.11 points. The cheap segment improved slightly, while out-of-time MAPE worsened.

One positive signal appeared: routed inference beat the general model on that
temporal holdout with a paired confidence interval below zero. Later snapshots
did not preserve a conclusive overall advantage, so both grouped and temporal
evidence remain necessary.

## 28. Definition drift invalidated supervised photo claims

The old interface used a broad term that annotators reasonably interpreted as
including rust, scuffs, dirt, and paint defects. Comments on all 47 legacy
`damaged` frames showed that 38 required visual re-review; only a small subset
clearly described impact, dents, deformation, or missing parts.

Every legacy `damaged` row was marked `needs_review` non-destructively. The
original CSV was backed up, pending rows are excluded from training and COCO
export, and no automatic relabeling was attempted. Only three independent
positive listings are currently verified for CV.

Decision: withdraw all supervised CV figures until the labels are reviewed
under the exact English protocol.

## 29. Targeted enrichment changed anomaly flags, not MAPE

A 2 September enrichment batch added 20 detail pages and 20 average-price/badge
records without HTTP 429. Six rule alerts were exculpated. After the full ML
chain, grouped MAPE moved by +0.0439 percentage points—far below the roughly
0.25-point bootstrap standard deviation.

This was not a meaningful degradation. Enrichment improved anomaly evidence;
the batch was simply too small and too sparsely covered to move global price
accuracy.

## 30. The listing number needs an explicit price-basis policy

One enriched listing advertised 7.0M KZT without customs clearance, 10.9M KZT
with customs clearance, and 11.4M KZT on credit. The saved listing target was
7.0M. Treating a generic credit or customs keyword as a row-level flag would be
wrong because all three meanings occur in the same description.

A contextual classifier now parses amounts and associates the saved price with
the nearest supported cue. It also handles customs negation, spelling variants,
clause boundaries, and disagreement between structured fields and prose.
Ordinary dealer finance boilerplate did produce false positives in the first
draft; corpus review caught them, and credit/down-payment labels now require the
advertised amount to be explicitly tied to the cue.

On the corpus audit, 26 of 12,799 rows were classified as `cash_uncleared`; 24
had previously been eligible for training. No current row met the
high-confidence credit-price or down-payment rule. Ambiguous rows remain
eligible.

A controlled grouped-CV A/B on the same snapshot measured **21.7327% MAPE
without** this filter and **21.6333% with** it, an improvement of 0.0993
percentage points. That small change is below the model's overall bootstrap
variation, but the target definition is more correct independently of the
headline metric. The previous artifact's 21.3044% is not a valid A/B baseline:
manual verdicts changed the training cohort between those runs.

The same eligibility rule now governs model training, floor calibration,
residual review, generated reports, CLI examples, and local comparable listings.
This closed a train/report skew found when the first updated dashboard still
reported 12,666 rows instead of the artifact's 12,642.

## 31. Cheap-segment labels must diagnose causes before becoming features

A damaged vehicle can have a perfectly honest comparable cash price, while an
intact vehicle can advertise only a down payment. One broad “cheap because of
condition” label would mix target corruption with physical condition and make
both text and CV experiments uninterpretable.

The first `/price-review` pilot therefore fixes 50 listings before any labels
are removed: 30 old vehicles with large grouped-OOF errors, 10 random cheap
controls, and 10 random audit listings selected before error ranking. The audit
is random within the already-downloaded-photo pool; it cannot estimate
segment-wide prevalence until photo acquisition is randomized. Model
predictions and errors are not sent to the browser. The reviewer independently
records vehicle state, price validity, evidence source, and an optional
data-quality issue while seeing at least three locally stored viewpoints and
existing text. A single seller-selected cover is insufficient for a confident
listing-level condition decision.

These labels are not connected to price training. They first answer whether
the missing scalable signal is mostly parseable seller text, visible condition,
non-comparable prices, or wrong structured fields. A photo can be opened in the
existing bounding-box tool, but frame geometry remains in the CV journal and
listing-level price diagnosis remains in its own journal.

Decision: analyse the fixed pilot before expanding annotation. Build an
automated text or photo feature only for a cause that is common and associated
with OOF error, then require improvement on grouped CV and the temporal holdout
before changing the deployed model.

## 32. The complete local review produced two fraud rows and many honest unknowns

On 5 September all 498 durable anomaly-journal rows received an explicit
review state: 378 `legit`, 2 `fraud`, and 118 `unknown`; no row is untouched.
The audit used stored descriptions, structured condition badges, duplicate
relations, and already-downloaded photos. It made no new source-site requests.

The two fraud rows are one correlated case: an exact-photo match (pHash distance
zero) with identical price, mileage, and description posted hours apart as the
incompatible UAZ Pickup and UAZ Patriot models. This is strong identity
deception evidence, but it is not two independent fraud mechanisms.

Most unknown rows are residual candidates with no locally stored seller text,
condition badge, or photo. Calling those listings legitimate would train
absence of enrichment as a negative label; calling them fraud would confuse a
model error with deception. `unknown` is therefore the only defensible label
until evidence is acquired. Raw binary-sample metrics are precision 1.6%, recall
100%, and F1 3.2%, but recall has only the two correlated UAZ positives and is
not stable enough for a product claim.

## 33. The first cheap-price and photo-definition audits are complete

The fixed 50-listing below-₸5M pilot contains 20 normal, 16 cosmetic,
9 repair-needed, 3 parts, 1 non-running, and 1 unclear vehicle. It remains a
diagnostic dataset rather than a training input. Finding 34 records its joined
OOF analysis and the first target-policy change supported by that evidence.

The complete 784-frame visual journal was also reviewed under the narrow impact
definition. Final counts are 18 boxed `damaged` frames from 16 listings,
6 `wreck` frames from 2 listings, 9 `parts`, 615 `intact`, and 136 `unclear`.
No `needs_review` row remains. The key limitation is now sample size rather
than definition drift: only 18 independent damaged/wreck listings are verified,
roughly 182 short of the planned stable local evaluation target.

## 34. The cheap-price pilot found target contamination before a useful CV feature

The 50 completed reviews were joined to the OOF predictions hidden during
annotation. The 30 old listings deliberately selected for high error had
104.87% mean APE. The two random-source subsets contained 20 listings and had
14.32% combined MAPE. Neither figure is a full-market estimate: the former is
selected on the outcome, while both sources require already-downloaded photos.

The strongest actionable pattern was not a new image feature. Three reviewed
Delicas were sold without both engine and gearbox. Their mean APE was 457%
because the regression target represented an incomplete shell while the model
was trained to estimate complete vehicles. The corpus-wide rule was therefore
kept deliberately narrow: `parts_price` requires explicit grammatical evidence
that both major assemblies are absent. Generic “for parts or restoration” text
does not qualify. Corpus review found five matches; four had previously been
training-eligible.

The first broad pattern falsely matched a poorly punctuated claim that an intact
Nissan's engine and gearbox were ideal. That experiment was stopped before any
artifact was written. Restricting `without/no` patterns to the expected
genitive forms removed the false positive and left exactly the five auditable
shell listings.

On the same 12,638 valid rows, retraining after the rule changed MAPE from
21.5072% to 21.4453%, a paired delta of -0.0619 percentage points with a 95%
grouped-bootstrap interval of [-0.2602, +0.1377]. This is not statistically
supported model lift. The production rebuild reports 21.4842% on 12,639 rows,
but its difference from the prior 21.6333% headline also includes the changed
evaluation cohort and must not be sold as pure model improvement.

The fixed pilot is now stored in an immutable local manifest before retraining.
This closes a lifecycle bug: recomputing the high-error queue after target
cleaning would otherwise replace already-reviewed cases and make the diagnostic
analysis irreproducible.

## 35. The public image silently used a fallback price range

The service prefers calibrated lower and upper quantile models, but the public
Dockerfile originally copied only the point model, cheap specialist, and point
metadata. The local development service therefore returned a conformal range
while the live image always fell back to a fixed 0.88–1.15 multiplier. That was
a deployment-contract bug: the README's calibrated-range claim was true for
local evaluation but false for the packaged product.

The public artifact bundle now contains all six files, and CI creates a small
schema-compatible synthetic interval pair for its container smoke test. A
local build of the real public image returned `range_method=conformal` with
stored grouped-OOF coverage 0.8010. No database or private row-level data are
included in the image.

## Practical rules derived from these findings

1. Measure on grouped OOF and out-of-time predictions, never training rows.
2. Report uncertainty before interpreting a change of a few hundredths.
3. Compare images with age-plus-price, not with a coin flip.
4. Keep active learning separate from the random audit.
5. Never mix rust, cosmetic wear, and impact under one damage class.
6. Do not add a feature until it exists at both train and serve time.
7. Do not call an anomaly fraud before an evidence-backed review verdict.
8. Do not expand geography without redefining and validating the product.
9. Prefer new condition evidence over more repetitions of plateaued fields.
10. Preserve failed experiments so future work starts from evidence.
11. Classify what a displayed price means before treating it as a regression target.
12. Freeze a human-review cohort before any rule that can change its membership.
13. Test the packaged product contract; local artifact availability is not deployment proof.

---

## 36. Fresh listings do not help, and the cheap segment is a tail problem

Two questions were asked together: whether collecting fresher listings
improves MAPE, and how the below-5M segment could actually be reduced.

### The market did not move, so recency buys nothing

Median basket price fell from 7.8M to 6.3M across the collection window,
which looks like a falling market until composition is held fixed. Within
make + model + year groups of at least eight cars, relative price stayed at
1.000 every week from mid-July to early September. The fitted trend is
-0.1% per month.

| week | comparable rows | relative price |
|---|---:|---:|
| 13 Jul | 1187 | 1.000 |
| 20 Jul | 252 | 1.000 |
| 27 Jul | 95 | 1.013 |
| 3 Aug | 156 | 1.000 |
| 24 Aug | 2480 | 1.000 |
| 31 Aug | 229 | 1.000 |

A model trained on July data therefore does not misprice September
listings, and "collect fresher cards" is not a MAPE lever. It remains
useful for anomaly review and for photo supply. The window is only seven
weeks, so this says nothing about a year.

### The apparent bias in the cheap segment is mostly how the segment is cut

Segmenting by actual price selects, by construction, the cars the model
priced above their asking price: a low actual price *is* the definition of
over-prediction. The same mechanism already explained the interval tail
skew in section 10.

| segment definition | rows | MAPE | mean bias | median bias |
|---|---:|---:|---:|---:|
| actual price below 5M | 5043 | 29.83% | +12.45% | +3.23% |
| predicted price below 5M | 5119 | 28.78% | +7.33% | -1.45% |

Age carries no such trap, because it is known before the prediction:

| age | rows | MAPE | mean bias | median bias |
|---|---:|---:|---:|---:|
| 0-10 | 4982 | 16.47% | +2.23% | -0.45% |
| 10-20 | 3408 | 18.99% | +3.60% | -1.44% |
| 20-30 | 2503 | 24.78% | +5.64% | -1.27% |
| 30+ | 1746 | 37.12% | +11.13% | -0.52% |

**Median bias is approximately zero in every age band.** The model is not
systematically optimistic about old cars. The rising mean comes entirely
from a right tail of listings priced far below what their specification
implies.

### The error is a tail, and part of the tail is already identifiable

Within the cheap segment the worst 1% of rows carry 9.2% of segment error,
the worst 10% carry 38.9%, and the worst 25% carry 62.8%. Tabular fields do
not separate the bad quartile from the rest: missing mileage, photo count,
VIP status, and description length are all indistinguishable.

What does separate them is evidence that the car is not an ordinary car:

| subgroup | rows | MAPE | mean bias |
|---|---:|---:|---:|
| accident / non-running badge | 32 | **168.3%** | +166.6% |
| explicit damage keywords | 80 | 36.7% | +27.6% |
| "negotiable" or "urgent" in text | 525 | 34.0% | +22.1% |
| weld / rot / rust wording | 35 | 35.1% | +15.8% |
| seller says painted or hit | 37 | 23.0% | **-0.8%** |

The last row is the interesting one. When a seller states the car was hit
or repainted, the model is unbiased — those sellers price honestly and the
listing is otherwise ordinary. The damage is disclosed and already in the
price.

### What follows

Chasing "cheap-segment MAPE" as a single number is the wrong framing. Half
of it is metric arithmetic (section 21), the apparent bias is largely how
the segment is cut, and the median listing is priced correctly. The
remaining, real problem is a minority of listings whose price reflects
something other than a working car of that specification.

Two of those causes are already handled or identifiable without any vision
work: shells missing engine and gearbox (`parts_price`, section 34) and
listings the marketplace itself flags as damaged or non-running. The badge
comes from enrichment and covers 12.8% of rows, so it is training-data
hygiene rather than a feature — the same coverage trap as section 17.

The honest target is therefore not "reduce 29% to 20.5%" but "identify the
tail". A metric that improves because a shell listing left the training set
is a real improvement; a metric that improves because the segment boundary
moved is not.

---

## 37. Excluding wrecks moves the headline, not the model

The below-5M tail contains listings whose price is not the price of a working
vehicle. Section 34 already removed shells missing both engine and gearbox.
This adds vehicles the listing or the marketplace states do not run.

Measured out-of-fold before any change:

| evidence | rows | MAPE | mean bias |
|---|---:|---:|---:|
| marketplace badge "Аварийная/Не на ходу" | 33 | **163.2%** | +161.6% |
| text: does not run / will not start / does not drive | 15 | 90.8% | +82.9% |
| text: аварийн | 11 | 52.0% | +48.6% |
| text: после ДТП | 88 | **18.7%** | +5.6% |
| whole corpus | 12639 | 21.6% | +4.5% |

### What is excluded and what is not

"После ДТП" is **not** excluded. Those listings score better than the corpus
average with almost no bias, because a repaired car is an ordinary car and
its seller has already priced the history in. Removing them would delete easy
rows and flatter the metric while hiding nothing — the difference between
cleaning a target and gaming a number.

Two patterns were dropped after adversarial checks rather than shipped:

* `на запчасти` fires on "денег на запчасти не жалели", which describes a
  well-maintained car, and on "есть комплект на запчасти в подарок". It
  added six rows against a wide false-positive surface, and genuine shells
  are already caught by the powertrain rule.
* `аварийная` needs its noun. Unqualified, it matches "аварийная
  сигнализация" — a hazard light fitted to every car.

Both errors point the same way as section 34's false positive: prefer a
missed wreck to a healthy car thrown out of training.

### The effect, split so it cannot be misread

Removing rows from **evaluation** is a cohort change: the metric falls
because harder cases stopped being measured. Removing them from **training**
is a model change. Reporting the sum as an improvement would be dishonest,
so both are measured separately on identical rows.

| what is measured | MAPE |
|---|---:|
| A. current: all rows in training and evaluation | 21.648% |
| A. same predictions, non-running rows dropped from evaluation only | 21.207% |
| B. non-running rows also dropped from training, same evaluation rows | 21.184% |

```
cohort change   -0.441 points   ← not a model improvement
model effect    -0.024 points   95% CI [-0.140, +0.091]
```

The model barely moved. Forty-nine rows out of 12,639 cannot change what a
gradient-boosted model has learned, and the confidence interval crosses zero.

**The honest statement is therefore: the exclusion is correct target hygiene
and the headline MAPE falls by roughly 0.44 points because the question
changed, not because the answer improved.**

### Why the cleaned number is still not "the real MAPE"

The badge exists only for enriched rows, currently 12.8% of the corpus. Forty
-three badged wrecks at that coverage imply a few hundred in total, of which
this rule identifies well under a quarter. What remains is a partially
cleaned set, not a clean one.

### Scope has to hold at serving time too

Narrowing training scope obliges the service to admit the boundary, so
`/estimate` now warns when a description states the vehicle does not run and
says the estimate does not apply. Without it the form would quietly price a
wreck as a working car — the same train/serve mismatch that let the public
image substitute a fixed price range for a calibrated one (section 35).
