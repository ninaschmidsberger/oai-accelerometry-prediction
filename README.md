# Accelerometry-derived activity patterns as predictors of outcomes in knee osteoarthritis

Analysis code for a study examining whether accelerometry-derived physical activity
features predict clinical, functional and structural outcomes in knee osteoarthritis.
The pipeline derives activity features from raw minute-level accelerometer counts,
reduces the predictor and outcome sets, fits and compares five model families, tests
temporal transportability from the 48-month to the 72-month visit, and produces the
publication figures.

**This repository contains code only. It contains no participant data.** The data are
held in a controlled-access repository and must be obtained separately, as described
below.

---

## DATA SOURCE

Data used in the preparation of this work were obtained from the Osteoarthritis
Initiative (OAI) database. The OAI is a multicentre, longitudinal, prospective
observational study of knee osteoarthritis. It enrolled 4,796 participants aged 45 to 79
years at recruitment, with annual clinical assessment and imaging. Data collection ran
from February 2004 to October 2015.

This analysis draws on the accelerometry ancillary study together with core clinical and
radiographic datasets. Two visits are analysed. Visit 06 is the 48-month follow-up and
serves as the development sample. Visit 08 is the 72-month follow-up and serves as the
temporal validation sample.

### Files required in `data/input/06_input`

| OAI file | Use in this analysis |
| --- | --- |
| `Accelerometry06` | Participant-level accelerometry summary |
| `AccelDataByDay06` | Day-level accelerometry |
| `Acceldatabymin06` | Minute-level activity counts |
| `AllClinical06` | Covariates and outcomes at Visit 06 |
| `AllClinical00`, `AllClinical01`, `AllClinical03`, `AllClinical05` | Prior knee surgery history |
| `KXR_SQ_BU06` | Kellgren and Lawrence grade |
| `Enrollees` | Sex |

### Files required in `data/input/08_input`

| OAI file | Use in this analysis |
| --- | --- |
| `Accelerometry08` | Participant-level accelerometry summary |
| `AccelDataByDay08` | Day-level accelerometry |
| `Acceldatabymin08` | Minute-level activity counts |
| `AllClinical08` | Outcomes at Visit 08 |
| `AllClinical07` | Interval knee surgery history |
| `KXR_SQ_BU08` | Kellgren and Lawrence grade |
| `Enrollees` | Sex |

OAI text exports are pipe-delimited. Section 1.3 of notebooks 01 and 02 rewrites every
`.txt` file in the input directory to `.csv`, preserving the pipe separator. All
downstream reads use `sep="|"`.

---

## HOW TO OBTAIN THE DATA

OAI data are distributed through the NIMH Data Archive (NDA). A user account and
registration for access are required. Access is free, but a Data Use Agreement must be
signed before any data can be downloaded.

### 1. Create an account and sign in

Go to <https://nda.nih.gov/oai>. Sign in through the Research Auth Service using a
Login.gov account, an eRA Commons account, or a PIV/CAC credential.

### 2. Request access to the OAI permission group

Open "My Account" in the upper right, then "Data Permissions". Scroll to "Request Access
to a Permission Group", find "Osteoarthritis Initiative", and click "Request Access",
then "Start Request". Follow the remaining prompts. The Data Use Agreement is part of
this step and must be completed before data become available.

Data within the OAI permission group are open access, so approval is administrative
rather than a scientific review. Allow time for processing all the same.

### 3. Build a data package and download it

Once access is granted, select the datasets listed above in the NDA query tool and
create a data package. Note the numeric package identifier.

Packages can be downloaded through the browser or with the official command line client.

### 4. Place the files

Sort the downloaded files into `data/input/06_input` and `data/input/08_input` as listed
above. `Enrollees` is needed in both. Paths in the notebooks are relative to the
repository root, so no path editing is required if the layout is followed.

---

## REQUIREMENTS

Python 3.13 or later. Dependencies are managed with [uv](https://docs.astral.sh/uv/).

```bash
uv sync
```

This creates a virtual environment and installs the exact versions recorded in
`uv.lock`. Select that environment as the Jupyter kernel before running the notebooks.

| Package | Used in |
| --- | --- |
| pandas, numpy, scipy | All notebooks |
| statsmodels | Harmonic regression, notebook 02 |
| matplotlib, seaborn | Exploratory plots, notebooks 01 and 02 |
| scikit-posthocs | Group comparisons, notebooks 01 and 02 |
| scikit-learn, xgboost | Modelling, notebook 03 |

## REPOSITORY STRUCTURE

```
├── 01_build_06_metrics.ipynb
├── 02_build_08_metrics.ipynb
├── 03_predictive_models.ipynb
├── data/
│   ├── input/
│   │   ├── 06_input/                 # OAI files, Visit 06 (not tracked)
│   │   └── 08_input/                 # OAI files, Visit 08 (not tracked)
│   └── output/
│       ├── 06_output/                # Derived Visit 06 dataframes
│       ├── 08_output/                # Derived Visit 08 dataframes
│       └── prediction_output/        # Model results and figures
├── pyproject.toml
├── uv.lock
├── LICENSE
├── .gitignore
└── README.md
```

`data/` is excluded from version control. Create the input directories and place the OAI
files there before running anything.

---

## RUNNING THE ANALYSIS

Run the notebooks in numerical order. Each stage writes files that the next stage reads,
so the order is not optional.

### 01. Build Visit 06 metrics

Defines and reduces the outcome set, builds the participant, day and minute level
dataframes, applies wear-time and validity cleaning, and engineers the activity
features. Feature engineering covers bout structure, WHO guideline compliance, activity
onset and offset, two-harmonic regression yielding MESOR, amplitude and acrophase,
intradaily variability and interdaily stability, per-day harmonic decomposition, and
day-type contrasts between weekdays and weekends. Section 6 applies the two-stage
predictor reduction, first removing arithmetic identities and pre-specified intensity
collapses, then removing empirically collinear pairs, to arrive at
`FINAL_PREDICTOR_COLUMNS`.

Writes to `data/output/06_output`:
`all_clinical_06_merged.csv`, `summary_metrics_06.csv`, `daily_metrics_06.csv`,
`minute_metrics_06.csv`.

### 02. Build Visit 08 metrics

Repeats the feature derivation at Visit 08 using the outcome set fixed at Visit 06. The
Visit 08 cohort is restricted to the Visit 06 analytic sample, so
`data/output/06_output/summary_metrics_06.csv` must already exist.

Writes to `data/output/08_output`:
`all_clinical_08_merged.csv`, `summary_metrics_08.csv`, `daily_metrics_08.csv`,
`minute_metrics_08.csv`.

### 03. Predictive models

Reads the derived summary metrics from both visits. Fits LASSO, ridge, elastic net,
random forest and XGBoost across all outcomes, plus a class-balanced XGBoost classifier
for Kellgren and Lawrence grade. Refits the winning model per outcome on the full Visit
06 sample, applies it at Visit 08, and runs the confirmatory incremental R² permutation
test comparing a covariate-only model against a model that adds the activity block.

Writes to `data/output/prediction_output`:

```
stage_1_lasso_summary.csv
stage_1_feature_selection_frequency.csv
stage_2_ridge_summary.csv
stage_3_elastic_net_summary.csv
stage_4_random_forest_summary.csv
stage_5_xgboost_summary.csv
stage_6_kl_grade_classification.csv
stage_7_model_comparison.csv
stage_8_longitudinal_validation.csv
stage_8_longitudinal_combined_vs_activity_only.csv
stage_8_kl_grade_longitudinal.csv
stage_9_confirmatory_inference.csv
predictions_all_outcomes.csv
figures/                              # Vector and raster versions of Figures 1 to 3
```

### Reproducibility

`RANDOM_STATE = 42` and `CROSS_VALIDATION_FOLDS = 10` throughout notebook 03. Given the
same input data and package versions, results are reproducible. Note that model families
and hyperparameters for the longitudinal refit were selected with covariates present and
are reused unchanged for the activity-only refit. This is documented in the methods.

---

## CITATION

If you use this code, please cite the accompanying article.

> [Author list]. [Title]. [Journal]. [Year]. doi:[DOI]

The analysis is also registered as an NDA Study with its own persistent identifier:
doi:[NDA STUDY DOI].

---

## ACKNOWLEDGEMENT OF THE OAI

Anyone publishing work based on OAI data is required to reproduce the following
acknowledgement.

> The OAI is a public-private partnership comprised of five contracts
> (N01-AR-2-2258; N01-AR-2-2259; N01-AR-2-2260; N01-AR-2-2261; N01-AR-2-2262) funded by
> the National Institutes of Health, a branch of the Department of Health and Human
> Services, and conducted by the OAI Study Investigators. Private funding partners
> include Merck Research Laboratories; Novartis Pharmaceuticals Corporation,
> GlaxoSmithKline; and Pfizer, Inc. Private sector funding for the OAI is managed by the
> Foundation for the National Institutes of Health.

This work was conducted retrospectively using data made available by the OAI. Ethical
approval and participant informed consent were obtained by the OAI. The views expressed
here are those of the authors and do not necessarily reflect the opinions of the OAI
investigators, the National Institutes of Health, or the private funding partners.

---

## LICENSE

Code in this repository is released under the terms in [LICENSE](LICENSE). The license
applies to the code only and confers no rights over the OAI data, which remain governed
by the NDA Data Use Agreement.

---

## CONTACT

Nina Schmidsberger, FH Campus Wien, University of Applied Sciences, Favoritensraße 226, 1100 Vienna, Austria,
nina.schmidsberger@fh-campuswien.ac.at
