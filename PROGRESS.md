# Project progress

## 2026-10-28 — Memory initialized

Read `AGENTS.md` and inspected the current 47-cell `data.ipynb`. Created
`PROJECT.md` and this log for continuity between chats. Results below were checked
against saved code and outputs; experiments were not rerun during this task.
Existing notebook and datasets were not modified.

## Completed work carried forward

- Loaded both CSV files, removed one fully empty training row, checked missing
  values and within-file duplicates, and compared descriptive statistics.
- Created a stratified 80/20 split of the provided training file.
- Trained two logistic regression pipelines, with and without `Contract Length`.
- Evaluated train/validation/testing ROC-AUC and validation/testing classification
  reports and confusion matrices. ROC plots are embedded notebook attachments.
- Added a baseline results table and a note about differing dataset relationships.
- Built grouped count/churn-rate tables for `Support Calls`, `Payment Delay`, and
  `Tenure` in each file.

### Recorded ROC-AUC

| Variant | Train | Validation | Testing |
|---|---:|---:|---:|
| With Contract Length (`model`) | 0.959724 | 0.959026 | 0.689440 |
| Without Contract Length (`model_noLength`) | 0.928481 | 0.928107 | 0.774983 |

### Recorded confusion matrices at threshold 0.5

Entries are TN, FP, FN, TP; class 1 means churn.

| Dataset | Variant | TN | FP | FN | TP |
|---|---|---:|---:|---:|---:|
| Validation | With Contract Length | 34,492 | 3,675 | 5,726 | 44,274 |
| Validation | Without Contract Length | 32,255 | 5,912 | 7,252 | 42,748 |
| Testing | With Contract Length | 6,564 | 27,317 | 287 | 30,206 |
| Testing | Without Contract Length | 7,813 | 26,068 | 528 | 29,965 |

Removing Contract Length improved testing AUC and reduced FP by 1,249, while
increasing FN by 241. It reduced validation AUC. Both variants produce many false
alarms on testing. Do not repeat the earlier mistaken claim that validation FN
decreased after removing the feature: they increased from 5,726 to 7,252.

## Data findings and interpretation

- Churn prevalence: approximately 56.7% in training versus 47.4% in testing.
- All training customers with `Contract Length = Monthly` churned; testing churn
  for Monthly is approximately 51.6%.
- Training churn is 100% for `Support Calls >= 6`, with about 24,000 customers per
  value. Testing churn for the same values is about 61–62%.
- Training churn is 100% for `Payment Delay >= 21` (not 22). Testing churn for
  values 21–30 is about 76–78%; at 0–15 it is about 10%, and at 16–20 about 59%.
- `Tenure` relationships also differ. Median tenure for stayed/churned customers:
  training 33/30, testing 27/37.
- These are observed associations, not evidence that support calls or late
  payments cause churn. Marginal group tables do not isolate causal effects.
- Train and validation AUC are very close. A large external performance drop is
  consistent with differences between datasets; classic train/validation
  overfitting is not the main demonstrated issue.
- Source of the sharp rules is unknown. Synthetic generation, selection effects,
  and leakage have not been established as facts.

## Decisions and unresolved issues

- Continue the educational project with these data while documenting limitations.
- Keep both existing models for comparison; the variant without Contract Length
  is the current reference for further experiments, not a proven production model.
- Support Calls and Payment Delay remain in both implemented variants. Their
  removal has only been discussed, not tested.
- Testing has influenced feature decisions and cannot be described as independent.
- Prediction horizon, feature timing, data provenance, and business error costs
  remain unresolved. New unseen data would be needed for an independent final check.
- Dependencies are not yet captured in a requirements file. A complete clean
  kernel run has not been verified in this task.

## Next steps discussed, not yet performed

1. Most recent proposed learning task: controlled feature ablation using logistic
   regression without Contract Length as reference. Compare additional exclusion
   of Support Calls, Payment Delay, and both. Keep rows, settings, and evaluation
   consistent; rebuild feature lists and use independent preprocessing objects.
   Results on the already examined testing file remain exploratory.
2. Then compare one CatBoost classifier with logistic regression using identical
   rows and features. Explain parameters and categorical handling before coding.
   CatBoost has been explained but is not present in the notebook.
3. Later: validation-based threshold selection under an explicit error-cost or
   outreach-budget assumption, interpretation, and reproducibility/documentation.

Do not automatically implement these experiments: resume mentorship with a small
concrete task and syntax support as needed. The student understands ML concepts
but has requested help remembering library syntax.

## 2026-11-14 — Full feature-by-feature comparison of train vs testing

Student wrote `feature_comprassion(feature, train_df, test_df, target="Churn")`
(cell 47): per-value `count` and `churn_rate` in each file, joined with
`how="outer"` so values present in only one file are kept. With authorization,
Claude fixed the call arguments in cell 47 and added cell 48 (loop over all
features except `CustomerID`, `Churn`, `Total Spend`). Cells were not executed in
the student's kernel; results below come from a standalone script using the same
loading steps. `Support Calls` output matched cell 43 exactly.

### Findings (churn rate, train vs testing)

- `Age`: train has decade steps (18–19 ≈0.65, 20–29 ≈0.54, 30–39 ≈0.47,
  40–49 ≈0.40, 50 ≈0.36) and **51–65 → 100% churn** (~82k rows). Testing is flat
  ≈0.44 below 51 and ≈0.52–0.55 for 51–65. New hard rule, not seen before.
- `Last Interaction`: train step at 16 (≈0.49 → ≈0.66); testing flat ≈0.47.
- `Tenure`: train steps at 6, 12, 24 (≈0.65 / 0.54 / 0.63 / 0.54); testing one
  step at 24 (≈0.30 → ≈0.56).
- `Usage Frequency`: train step at 10 (≈0.61 → ≈0.55); testing steps at 3 and 6
  (≈0.69 / 0.61 / 0.43).
- `Payment Delay`: steps at 16 and 21 in both files, but levels differ
  (0–15: 0.47 train vs 0.10 testing; 21–30: 1.00 vs ≈0.77).
- `Support Calls`, `Contract Length`: as recorded earlier.
- `Gender`: Female churn higher in both files (0.67/0.49 train, 0.55/0.39
  testing) — the only feature with a consistent direction.
- `Subscription Type`: nearly flat in both files — likely weak signal.
- `Total Spend`: 68,363 unique values in train vs 901 in testing — different
  precision/rounding suspected, not yet checked; needs binning for comparison.
- `CustomerID`: 62,995 IDs appear in both files, but testing IDs are a
  contiguous 1..64,374 range, so this is likely row numbering, not shared
  customers. Not verified.

### Interpretation

Nearly all numeric features show piecewise-constant churn rates with steps at
round values, and the step locations/levels differ between files. This strongly
suggests synthetic data generated with different rules per file (widely
discussed for this Kaggle dataset), but it is an inference, not a verified fact.
Implication: one-at-a-time ablation of `Support Calls`/`Payment Delay` is less
informative than planned, because deterministic rules exist in most features
(`Age`, `Contract Length`, `Support Calls`, `Payment Delay`).

### Open questions / next steps

- Decide how to proceed given likely synthetic data: continue as a methodology
  exercise with documented limitations, or switch to a more realistic churn
  dataset. Not decided.
- Optional checks: `Total Spend` precision/binning (bin edges from train only);
  share of train rows covered by any 100%-churn rule; adversarial validation.
- Function name typo (`comprassion` → `comparison`) not yet fixed.

## 2026-11-14 — Decision: close Kaggle dataset, move to IBM Telco; Git set up

- Student wrote a markdown conclusion (cell 49 of `data.ipynb`): high validation
  AUC (0.96) vs testing (0.69–0.77) explained by differing deterministic rules
  between files (e.g. `Payment Delay > 20` → 100% train churn; in train no
  retained customer has `Monthly` contract). Claude suggested removing a leftover
  copied phrase ("вот доказательства»"), stating the Monthly rule holds only for
  train, and fixing typos; student considered the conclusion sufficient.
  Whether these fixes were applied was not verified.
- **Decision (option B):** stop modeling on the Kaggle dataset — it is very
  likely synthetic, so CatBoost/threshold/interpretation results would be
  meaningless. `data.ipynb` stays as a completed investigation. Next modeling
  stages move to the IBM Telco Customer Churn dataset (Kaggle, ~7k rows) in a
  **new notebook**.
- Created `README.md` (title + dataset link; CSVs not in repo). Student is
  expected to extend it with project description/conclusions.
- Git: student initialized the repo, added `.gitignore` (`*.csv`, `.DS_Store`,
  `.ipynb_checkpoints/`, `CLAUDE.md`), committed, and pushed to GitHub
  (`origin/main`, 2 commits as of end of session). Student also replaced the
  `AGENTS.md` reference in `PROJECT.md` with `CLAUDE.md`.
- A personal Git cheat sheet was written to `~/git_guide.md` (outside the repo).
- This PROGRESS/PROJECT update is uncommitted — commit and push it next session.

### Where to resume

1. Commit/push the updated `PROGRESS.md` and `PROJECT.md`.
2. Start the Telco project: download the dataset, decide folder layout (new
   notebook in this repo vs a separate repo — not decided), first look at data.
   Known Telco pitfalls to let the student discover: `TotalCharges` stored as
   string with blank values, class imbalance (~26% churn), `customerID` to drop,
   correlated features (`tenure`, `MonthlyCharges`, `TotalCharges`).
3. Follow the full workflow: split → baseline (LogReg pipeline) → CatBoost →
   threshold selection on validation → interpretation. Keep a true held-out test
   untouched this time.

## 2026-11-19 — Telco dataset chosen

- Student downloaded `Telco_customer_churn.xlsx` from Kaggle: the **extended IBM
  Telco version** (7,043 rows, 33 columns, sheet `Telco_Churn`), not the classic
  21-column `WA_Fn-UseC_-Telco-Customer-Churn.csv`. Student chose to use it.
- Extra columns: `Count`, `Country`, `State`, `City`, `Zip Code`, `Lat Long`,
  `Latitude`, `Longitude`, `Churn Label`, `Churn Value`, `Churn Score`, `CLTV`,
  `Churn Reason`. Several are post-outcome/derived (leakage) — student's task is
  to identify them; answers deliberately not given.
- Reading requires `openpyxl` (not installed as of this session; student installs).
- Proposed layout (not yet confirmed): `data/` folder for raw files, new
  `telco_churn.ipynb`, optional `git mv data.ipynb kaggle_synthetic_eda.ipynb`.
- Current task (stage 1): load, classify columns (target / drop with reason /
  features), check dtypes vs meaning, hidden missing values, duplicates, class
  balance; write findings in markdown without fixing. No split yet.
- Previous PROGRESS/PROJECT updates still uncommitted at session start.
