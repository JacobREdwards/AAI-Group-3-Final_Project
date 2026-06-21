# PackageGuard AI: Supply Chain Risk Detector

**AAI-540 Group 3 Final Project**  
**Business Name:** PackageGuard AI  
**Project Type:** End-to-End MLOps System  
**Domain:** Software Supply Chain Security  
**Primary Ecosystem:** PyPI / Python packages  

---

## 1. Project Overview

PackageGuard AI is a machine learning system designed to detect potentially malicious or risky Python packages before they are used in software projects. Modern applications rely heavily on open-source dependencies, and attackers often abuse this trust by publishing malicious packages, typosquatting popular libraries, or introducing risky packages into dependency chains.

The system analyzes PyPI package metadata and engineered package-risk features to produce a risk score between `0` and `1`. Packages with higher scores are flagged for human review. The system is not designed to automatically block packages or accuse maintainers. It is designed as an early warning and decision-support tool for developers, security teams, and CI/CD workflows.

---

## 2. Business Use Case

Open-source software supply chain attacks can create serious business risk. A malicious package introduced into a build pipeline can steal secrets, exfiltrate environment variables, install backdoors, or compromise downstream systems.

PackageGuard AI helps reduce this risk by:

- Identifying suspicious PyPI packages before adoption
- Ranking package risk using machine learning
- Supporting human review with explainable features
- Providing a repeatable MLOps pipeline for retraining and deployment
- Supporting integration into future CI/CD or security review workflows

---

## 3. Machine Learning Problem

This project solves a **supervised binary classification problem**.

### Model Objective

Predict whether a PyPI package is:

- `0` = benign / low risk
- `1` = suspicious / potentially malicious

### Model Output

The deployed model returns:

```json
{
  "package_name": "example-package",
  "malicious_probability": 0.94,
  "predicted_label": 1
}
```

The model output is interpreted as a risk signal, not final proof of malicious behavior.

---

## 4. Data Sources

The project uses public package metadata and malicious package labels.

### Primary Data Source

| Source | Purpose |
|---|---|
| `bigquery-public-data.pypi.distribution_metadata` | PyPI package metadata used for feature engineering |

### Malicious Label Sources

| Source | Purpose |
|---|---|
| OpenSSF malicious package reports | Known malicious package labels |
| DataDog malicious software packages dataset | Curated malicious package labels |

### Data Storage

Raw, processed, and feature data are stored in Amazon S3.

Example project S3 layout:

```text
s3://sagemaker-us-east-1-016905035504/supply-chain-risk/
  raw/
  processed/
  feature-store/
  training/
  validation/
  test/
  model-artifacts/
  batch-output/
  monitoring/
```

The SageMaker Feature Store offline store writes feature records to:

```text
s3://sagemaker-us-east-1-016905035504/supply-chain-risk/feature-store
```

---

## 5. Dataset Summary

The final training dataset was built using a balanced malicious and benign package design.

| Dataset Detail | Value |
|---|---|
| Malicious packages | 7,003 |
| Benign packages | 7,003 |
| Total rows | 14,006 |
| Class balance | 1:1 |
| Missing values after leakage fix | 0 NaN cells |
| Final feature count | 23 clean model features |
| Split strategy | Stratified train / validation / test |

The dataset was engineered to reduce leakage risk. Earlier versions of the dataset contained fields that could indirectly reveal whether packages were already removed or flagged. These fields were removed so the model learned real package patterns instead of shortcuts.

---

## 6. Feature Engineering

The model uses package metadata and package-name features.

### Metadata Features

Examples:

- Package age
- Days since latest release
- Release count
- Summary length
- Description length
- License presence
- Author / maintainer metadata presence
- Python version requirement indicator

### Name-Shape Features

Examples:

- Package name length
- Character diversity
- Name entropy
- Digit count
- Special character count
- Hyphen / underscore count
- Suspicious naming patterns

### Final Feature Store Output

The SageMaker Feature Store implementation prepared:

- 1,000 sample rows for feature ingestion
- 40 feature definitions
- 23 model-ready features used for training
- Online store for real-time lookup
- Offline store for training and batch use cases

Example Feature Group created during the final notebook run:

```text
supply-chain-risk-features-20260621144143
```

Example record verified from the online store:

```text
package_name: esqurlget
summary_length: 48
description_length: 4422
requires_python_present: 1
author_present: 1
license_present: 0
```

---

## 7. Model Training and Selection

Three models were trained and compared:

1. Logistic Regression
2. Random Forest
3. XGBoost

### Model Comparison

| Model | ROC-AUC | F1 Score | Role |
|---|---:|---:|---|
| Logistic Regression | 0.900 | 0.842 | Baseline |
| Random Forest | 0.976–0.977 | 0.924 | Candidate model |
| XGBoost | 0.981 test ROC-AUC | 0.929 | Selected model |

XGBoost was selected because it provided the best overall balance of ROC-AUC, F1 score, and practical training time.

---

## 8. Generalization Validation

The model was validated using multiple checks to reduce the risk of overfitting.

### Validation Methods

- Stratified train / validation / test split
- Train/test AUC gap check
- Five-fold cross-validation
- Permutation test
- Feature importance review

### Validation Results

| Validation Check | Result |
|---|---|
| Train/Test AUC gap | Approximately 0.015 |
| Cross-validation | Approximately 0.984 ± 0.001 |
| Permutation test | p < 0.01 |
| Conclusion | Model generalizes well and is not simply memorizing labels |

---

## 9. AWS Architecture

The project was implemented on AWS using SageMaker and supporting services.

```text
BigQuery PyPI Metadata + Malicious Labels
                 ↓
             Amazon S3
                 ↓
      SageMaker Processing / Notebook
                 ↓
         Feature Engineering
                 ↓
      SageMaker Feature Store
        /                    \
Online Store              Offline Store
   ↓                          ↓
Endpoint Inference       Training Dataset
                              ↓
                    SageMaker Training Job
                              ↓
                    Model Evaluation
                              ↓
                    SageMaker Model Registry
                              ↓
                    SageMaker Pipeline / CI-CD
```

---

## 10. SageMaker Feature Store

The project uses SageMaker Feature Store to manage package-level features.

### Feature Store Purpose

- Maintain a reusable feature repository
- Support consistent training and inference features
- Provide online lookup for real-time prediction
- Provide offline storage for batch training and retraining

### Feature Group Details

| Field | Value |
|---|---|
| Feature Group | `supply-chain-risk-features-20260621144143` |
| Feature definitions prepared | 40 |
| Sample rows ingested | 1,000 |
| Failed ingestion rows | 0 |
| Offline Store | S3-backed Parquet |
| Online Store | Enabled |

---

## 11. Model Registry

The selected model is registered in SageMaker Model Registry.

### Model Registry Purpose

- Store model versions
- Track model metrics
- Track model artifacts
- Support approval workflow
- Support deployment governance

### Model Package Group

```text
supply-chain-risk-models
```

Each registered model version includes:

- Algorithm name
- Model artifact reference
- Evaluation metrics
- Dataset reference
- Model metadata
- Approval status

---

## 12. CI/CD Pipeline

The project includes a SageMaker Pipeline to automate the ML lifecycle.

### Pipeline Stages

```text
DataPrep → FeatureEng → Train → Evaluate → Register
```

### CI/CD Quality Gate

The evaluation step acts as a checkpoint. If the model performance does not meet the required threshold, the pipeline stops and the model is not registered for deployment.

Example quality gate:

```text
Reject model if ROC-AUC < 0.85
```

This makes the pipeline repeatable, controlled, and safer for future retraining.

---

## 13. Deployment and Inference

The project supports real-time endpoint inference.

### Endpoint Use Case

A package or package feature vector is submitted to the deployed endpoint, and the model returns a malicious probability and predicted label.

Example:

```json
{
  "malicious_probability": 0.94,
  "predicted_label": 1
}
```

### Human Review Policy

Predictions above the configured threshold are flagged for human review. The system does not automatically block packages in the current MVP.

---

## 14. Monitoring

Monitoring is included as part of the ML system design.

### Infrastructure Monitoring

CloudWatch can track:

- Endpoint latency
- Invocation count
- Error rate
- 4XX / 5XX errors
- Request volume

### Data and Model Monitoring

The project design includes:

- PSI drift reports
- KS test reports
- Feature distribution checks
- Model metric checks
- Prediction distribution monitoring

Suggested drift interpretation:

| PSI Value | Meaning |
|---:|---|
| < 0.10 | Stable |
| 0.10–0.20 | Monitor |
| > 0.20 | Consider retraining |

---

## 15. Repository Structure

The GitHub repository should be organized as follows:

```text
.
├── README.md
├── requirements.txt
├── notebooks/
│   └── AAI540_final_run_aws_notebook_clean_run_v7.ipynb
├── src/
│   └── packageguard/
│       ├── data/
│       ├── features/
│       ├── models/
│       ├── monitoring/
│       └── utils/
├── scripts/
│   ├── 00_download_malicious_labels.py
│   ├── 01_ingest_data.py
│   ├── 02_feature_engineering.py
│   ├── 03_train_model.py
│   ├── 04_evaluate_model.py
│   ├── 05_register_model.py
│   └── 06_run_pipeline.py
├── docs/
│   ├── architecture.md
│   ├── data_sources.md
│   └── model_card.md
├── reports/
│   ├── model_results_v7.png
│   ├── generalization_tests.png
│   └── drift_report.json
└── tests/
    ├── test_features.py
    ├── test_schema.py
    └── test_model_smoke.py
```

---

## 16. Notebook Requirements

All notebooks must be stored in `.ipynb` format.

The final notebook should include:

- Data loading
- Data cleaning
- Feature engineering
- Leakage audit
- EDA charts
- Model training
- Model comparison chart
- Feature importance / SHAP chart
- Generalization validation
- SageMaker Feature Store creation
- Model Registry registration
- SageMaker Pipeline creation
- Endpoint invocation or batch inference
- Monitoring or drift report generation

Any graphs or charts used in the presentation should be generated or referenced in the notebook.

---

## 17. Graphics and Reports

The repository should include important graphics that help explain the system.

Recommended files:

```text
reports/model_results_v7.png
reports/generalization_tests.png
reports/feature_importance.png
reports/shap_summary.png
reports/pipeline_dag.png
reports/feature_store_screenshot.png
reports/model_registry_screenshot.png
```

These graphics help connect the GitHub repository to the presentation and ML Design Document.

---

## 18. How to Run the Code

### 1. Clone repository

```bash
git clone <TEAM_GITHUB_REPO_URL>
cd <repo-name>
```

### 2. Install dependencies

```bash
pip install -r requirements.txt
```

### 3. Run notebook

Open:

```text
notebooks/AAI540_final_run_aws_notebook_clean_run_v7.ipynb
```

Run the notebook in SageMaker Studio.

### 4. Verify Feature Store

In SageMaker Studio:

```text
Data → Feature Store → Feature Groups
```

Look for:

```text
supply-chain-risk-features-20260621144143
```

### 5. Verify Model Registry

In SageMaker Studio:

```text
Governance → Model Registry → Model Package Groups
```

Look for:

```text
supply-chain-risk-models
```

### 6. Verify Pipeline

In SageMaker Studio:

```text
Pipelines
```

Look for the project pipeline with steps:

```text
DataPrep → FeatureEng → Train → Evaluate → Register
```

---

## 19. Deliverable #3 Requirement Mapping

This repository is designed to satisfy Deliverable #3: Codebase GitHub Repository.

### Method Requirements

| Requirement | How This Repository Satisfies It |
|---|---|
| Code stored cleanly in GitHub | Repository uses organized folders for notebooks, source code, scripts, reports, and docs |
| Notebooks stored in `.ipynb` format | Final SageMaker notebook is stored under `notebooks/` |
| Code is clean and project-focused | Code supports data ingestion, feature engineering, training, evaluation, deployment, and monitoring |
| Useful comments included | Notebook and scripts should include comments explaining major ML and MLOps steps |
| Data stored in S3 and documented | S3 paths are documented in this README and notebook |
| Graphics included in notebooks | Model comparison, generalization, feature importance, and monitoring charts are included or referenced |

### ML Design Requirements

| Requirement | How This Repository Satisfies It |
|---|---|
| Comprehensive ML system codebase | Includes data, features, training, registry, deployment, monitoring, and CI/CD workflow |
| Supports ML Design Document | Code implements the same architecture described in the design document |
| Reflects project scope | Focuses on PyPI package risk detection and avoids out-of-scope ecosystems |
| Reflects production-style MLOps | Uses SageMaker Feature Store, Model Registry, Pipeline, endpoint, and monitoring design |
| Supports final presentation | Notebook outputs and reports align with presentation slides |

---

## 20. Teamwork Evidence

The repository should reflect teamwork through:

- Clear commit history
- Separate task ownership
- Issues or project board references
- Pull requests or reviewed changes
- Team tracker updates
- Consistent file organization

Suggested team ownership:

| Team Member | Area |
|---|---|
| Member 1 | Data sources, S3 data lake, Athena / data documentation |
| Member 2 | Feature engineering, EDA, model training, evaluation |
| Member 3 | Feature Store, Model Registry, CI/CD pipeline, monitoring, demo |

---

## 21. Known Limitations

- MVP focuses only on PyPI
- Benign labels mean “not known malicious,” not guaranteed safe
- No deep static code analysis in current version
- No dynamic sandbox execution
- No automatic blocking
- False positives may affect package reputation
- Model may not detect all novel or zero-day threats

---

## 22. Future Enhancements

Future improvements include:

- Add npm, Maven, and RubyGems support
- Add static code analysis of package source code
- Add dependency graph traversal beyond metadata features
- Add vulnerability enrichment from NVD and GitHub Advisory Database
- Add SHAP explanations for every prediction
- Add active learning from analyst feedback
- Add daily refresh of malicious package labels
- Add auto-scaling endpoint deployment
- Add full GitHub Actions or AWS CodePipeline automation

---

## 23. Responsible Use

PackageGuard AI should be used as a risk scoring and review-prioritization system. It should not be used as final proof that a package or maintainer is malicious.

High-risk predictions should be reviewed by a human security analyst before enforcement action.

---

## 24. Project Status

Current status:

- Dataset prepared
- Leakage-audited features created
- Three models trained
- XGBoost selected
- Generalization validation completed
- SageMaker Feature Store created
- Feature records ingested
- SageMaker Model Registry configured
- SageMaker Pipeline designed
- Endpoint inference demonstrated
- Monitoring strategy documented

---

## 25. References

Public data and security sources used or referenced:

- PyPI package metadata
- BigQuery public PyPI distribution metadata
- OpenSSF malicious package reports
- DataDog malicious software package dataset
- AWS SageMaker Feature Store
- AWS SageMaker Model Registry
- AWS SageMaker Pipelines
- AWS CloudWatch
