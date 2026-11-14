# Customer Churn Prediction

## Purpose and collaboration

Educational ML engineering project covering data analysis, preprocessing,
classification, evaluation, interpretation, and reproducibility.
Follow `AGENTS.md`: communicate in Russian, explain decisions, provide progressive
syntax help, and let the student implement learning tasks. Editing code or
installing dependencies requires explicit authorization. Maintain these memory
files as instructed in `AGENTS.md`.

At session start, read `AGENTS.md`, this file, and `PROGRESS.md`, then inspect
relevant code. The notebook is the source of truth; memory may become outdated.

## Files and architecture

- `data.ipynb`: data loading, EDA, splitting, preprocessing, model training,
  evaluation, and dataset diagnostics. Currently the only implementation file.
- `customer_churn_dataset-training-master.csv`: provided training dataset.
- `customer_churn_dataset-testing-master.csv`: provided testing dataset.
- `PROGRESS.md`: completed experiments, results, limitations, and next steps.

Keep the architecture simple. No deployment, APIs, Docker, cloud infrastructure,
or complex MLOps is in scope unless explicitly agreed.

## Data and prediction task

- Selected source: https://www.kaggle.com/datasets/muhammadshahidazeem/customer-churn-dataset
- Correspondence of local files to the original Kaggle files has not been verified.
- Both files contain 12 columns, including the target `Churn`.
- `Churn = 1`: customer left; `Churn = 0`: customer stayed.
- Training: 440,832 rows after `dropna(how="all")` removes one empty row.
- Testing: 64,374 rows. No remaining missing values were found.
- Within-file duplicate checks excluding `CustomerID` returned zero; this does
  not establish absence of overlapping customers between files.
- Prediction time, feature observation windows, and churn horizon are undefined.
  Availability of features before the outcome remains an unverified assumption.

## Current technical approach

- Exclude `CustomerID` and `Churn` from predictors.
- Split the provided training file with `train_test_split(test_size=0.2,
  random_state=42, stratify=y)`: 352,665 training and 88,167 validation rows.
- Numeric features: `Age`, `Tenure`, `Usage Frequency`, `Support Calls`,
  `Payment Delay`, `Total Spend`, `Last Interaction`.
- Categorical features: `Gender`, `Subscription Type`, `Contract Length`.
- `ColumnTransformer`: `StandardScaler()` for numeric features and
  `OneHotEncoder(handle_unknown="ignore")` for categorical features.
- `Pipeline` combines preprocessing with `LogisticRegression(max_iter=1000)`.
  Preprocessing is fitted only on training rows. Classification threshold: 0.5.
- Two variants exist: `model` with all predictors, `model_noLength` without
  `Contract Length`. Both use the same rows and target vectors.
- Evaluate ROC-AUC using probabilities for class 1; classification reports and
  confusion matrices use predicted classes. Matrix layout: `[[TN, FP], [FN, TP]]`.

## Evaluation limitations

The provided testing file has already informed diagnostics and feature decisions.
It is an external diagnostic dataset, not an untouched final holdout. Do not tune
thresholds or hyperparameters on it or claim an independent final assessment.

Several feature/target relationships differ sharply between files. Similar train
and validation AUC with substantially worse testing AUC supports a transfer
problem; it does not prove a particular data-generation mechanism or target
leakage. Suspicious features are not automatically useless, and removing them
does not establish reliability on future customers.

CatBoost, further feature ablations, threshold selection, hyperparameter search,
and a fresh independent holdout have not been implemented.
