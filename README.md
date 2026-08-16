# Cognitive Resilience and Brain Age

Operationalising cognitive resilience as latent cognitive performance conditional on
brain status, where brain status is indexed by a whole-brain multivariate brain age gap
estimated separately from resting-state functional connectivity and from structural
morphometry.

Master's thesis, Inayet Dincer, supervised by Dr. Daniel Kristanto and Prof. Andrea Hildebrandt.

---

## What this does

Two brain-age models are trained from scratch in a single cohort — one functional, one
structural — and each yields a per-participant brain age gap (BAG): predicted age minus
chronological age, after correcting the systematic compression of predictions towards the
sample mean.

Cognitive resilience is then defined as latent cognitive performance conditional on that
gap and on age, sex and education. Lifestyle and psychosocial factors are tested both as
correlates of the gap itself and as correlates of cognition conditional on the gap.

The two modalities are analysed with an identical cross-validation procedure so that any
difference between them reflects the imaging data rather than the pipeline.

---

## Pipeline

```
  RESTING-STATE fMRI                          STRUCTURAL MRI
  seed-wise release files                     released morphometry
          │                                            │
          ▼  01_build_connectivity.py                  ▼  02_build_structural.py
  379 x 379 Fisher-z matrices                 1,500 features
  reconstruct + verify symmetry               360 thickness
          │                                   360 surface area
          ▼  tanh, restore unit diagonal      360 cortical volume
  valid correlation matrices                  360 myelin (T1w/T2w)
  positive definiteness verified               60 subcortical + global
          │                                            │
          ▼                                            │
  ┌───────────────────────────────────────────────────────────────────┐
  │  WITHIN EACH OUTER TRAINING FOLD ONLY                             │
  │                                                                   │
  │  functional:  Karcher reference → tangent projection              │
  │               → 71,631 features → scale → PCA (100 components)    │
  │                                                                   │
  │  structural:  in-fold imputation → ICV adjustment (780 features)  │
  │               → scale       (no manifold step, no PCA)            │
  └───────────────────────────────────────────────────────────────────┘
          │                                            │
          ▼  04_functional_bag.py                      ▼  05_structural_bag.py
  ┌───────────────────────────────────────────────────────────────────┐
  │  NESTED CROSS-VALIDATION, identical for both arms                 │
  │    10 outer folds   generalisation estimate                       │
  │    10 inner folds   hyperparameter selection                      │
  │    10 seeds         repeated partitions, predictions averaged     │
  │    pedigree-grouped, stratified by sex x age decile               │
  │    LightGBM; learning rate and tree depth tuned                   │
  └───────────────────────────────────────────────────────────────────┘
          │                                            │
          ▼  cross-fitted age-bias correction          ▼
  functional BAG                              structural BAG
          └────────────────────┬───────────────────────┘
                               ▼  06_merge_bags.py
                       analysis_table.csv
                               │
                               ▼  R / lavaan
       bifactor-(S-1) cognitive model, psychosocial measurement model,
       structural equation models for H1-H4
```

---

## Repository layout

```
code/brain_age/          the six numbered scripts, run in order
code/brain_age/_dev/     exploratory work: the abandoned covariance route, the
                         eigenvalue-floor sensitivity analysis, representation
                         comparisons, PCA component sweeps. Not needed to reproduce
                         the main result; retained because it documents why several
                         decisions were made.
docs/                    analysis plan, methods notes
```

Data directories are excluded from version control — see *Data access* below.

---

## Running it

```bash
python 01_build_connectivity.py    # ~20 min   -> corr_V1.npy, meta.json
python 02_build_structural.py      # ~1 min    -> Xstruct.npy, icv.npy, volmask.npy
python 03_build_phenotype.py       # seconds   -> data/clean/analysis_phenotype.csv
python 05_structural_bag.py        # ~1 hour   -> bag_structural.csv
python 04_functional_bag.py        # overnight -> bag_functional.csv
python 06_merge_bags.py            # seconds   -> data/clean/analysis_table.csv
```

### Requirements

Python: `numpy`, `scipy`, `pandas`, `scikit-learn`, `lightgbm`
R: `lavaan`, `semTools`, `psych`, `mice`

Exact versions are recorded in the lock file archived with the analysis.

---

## Outputs

| File | Contents |
|---|---|
| `data/clean/analysis_phenotype.csv` | one row per participant: cognitive and psychosocial indicators, demographics, pedigree, site |
| `bag_functional.csv`, `bag_structural.csv` | per-modality gaps, raw and corrected predicted ages, across-seed stability |
| `bag_metrics.json`, `bag_structural_metrics.json` | accuracy per fold and per seed, selection optimism, error by age band, hyperparameters selected, Karcher convergence |
| `data/clean/analysis_table.csv` | everything joined — the single file the R models read |

---

## Methodological notes

A few choices that are not obvious from the code alone. The analysis plan gives the full
reasoning.

**Tangent-space representation.** Raw correlation values are not comparable across
participants: the same edge value means something different in a globally weakly coupled
connectome than in a strongly coupled one. Connectivity matrices are also symmetric
positive-definite objects whose entries are jointly constrained. Each participant is
therefore expressed as a deviation from the Riemannian (Karcher) mean of the training-fold
matrices. In a matched comparison this improved out-of-sample accuracy and roughly halved
fold-to-fold variability relative to directly vectorised correlations.

**Age-bias correction is cross-fitted.** The intercept and slope are estimated from
out-of-fold predictions *within* the training fold, not from the training model's own
in-sample predictions, which would under-correct. Held-out participants contribute to
neither coefficient.

**Pedigree grouping.** Relatives are kept in the same fold at both levels. Participants
without a released pedigree identifier each form their own group rather than being
collapsed together by default missing-value handling.

**Two dispersions are reported separately.** The standard deviation across outer folds
within a partition, and the standard deviation of partition means across repeated seeds,
measure different things and are not combined into a single figure.

**Nesting is complete for the learner, not for the preprocessing.** The tangent reference,
the scaler and the PCA basis are fitted once per outer training fold and the inner folds
cut from the result. The outer-test evaluation is unaffected; hyperparameter selection is
nested with respect to LightGBM but not with respect to these three unsupervised
transforms. This is reported as a limitation rather than described as strict nesting.

---

## Data access

**No data are included in this repository, and none should be added to it.**

The analyses use the Human Connectome Project–Aging / Aging Adult Brain Connectome (AABC),
Release 2, which is distributed under **registered access**. Obtaining the data requires an
approved data use agreement with the NIMH Data Archive. Redistributing any part of it —
including derived per-participant files such as connectivity matrices, feature tables or
brain age gaps — is not permitted.

To reproduce the analyses you will need your own approved access, after which the scripts
expect the released files in the paths set at the top of each script.

---

## Status

Preregistration and pipeline complete. 
