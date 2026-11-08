# Lab 12 Execution Report

## Latest verification

The requested Lab 12 scripts were executed from the project root on
2026-11-07. No Labs 1-11 source files or Kubernetes manifests were changed.
Per-script transcripts, commands, and exit codes are stored in `outputs/lab12`.

Execution order:

```text
12_api_performance_monitoring.py
12_create_evolved_dataset.py
12_create_concept_drift_dataset.py
12_data_drift.py
12_prediction_drift.py
12_concept_drift.py
12_drift_analysis.py
12_performance_monitoring.py
12_performance_trend.py
12_create_updated_dataset.py
12_retrain_model.py
12_model_comparison.py
12_update_mlflow_registry.py
```

## Monitoring

The following stages passed with exit code 0:

- Evolved and concept-drift dataset generation
- Data drift analysis
- Prediction drift analysis
- Concept drift analysis
- Combined drift analysis
- Production performance monitoring
- Performance trend analysis
- API performance monitoring

Observed results included drift in `MonthlyCharges`, probability-distribution
drift, and degraded accuracy/recall in the concept-drift comparison.

Actual combined drift report:

```text
Data drift: 1 feature, MonthlyCharges
Prediction distribution: OK, KS 0.068843
Probability distribution: DRIFT, KS 0.217175
Concept drift: 2 degraded metrics, accuracy and recall
```

## Retraining

The post-drift training batch was validated with zero exact overlap against the
held-out test set. Retraining produced:

- `models/retrained_model_v2.pkl`
- `models/retrained_preprocessor_v2.pkl`
- `outputs/retraining/updated_dataset_summary.csv`

Model comparison completed successfully. Actual metrics on the 1,409-row held-out
set were:

| Metric | V1 | V2 | Change |
|---|---:|---:|---:|
| Accuracy | 0.561391 | 0.567779 | +0.006388 |
| Precision | 0.273234 | 0.271845 | -0.001390 |
| Recall | 0.393048 | 0.374332 | -0.018717 |
| F1 | 0.322368 | 0.314961 | -0.007408 |
| ROC-AUC | 0.495708 | 0.500274 | +0.004566 |

Neither V1 nor V2 passed the configured recall threshold of `0.70`; V2 did not
improve recall over V1.

## Retraining and MLflow result

`12_create_updated_dataset.py` passed and validated zero exact overlap with the
held-out test set. `12_retrain_model.py` trained V2 and wrote the model and
preprocessor artifacts:

- `models/retrained_model_v2.pkl`
- `models/retrained_preprocessor_v2.pkl`

However, it exited with code 1 when MLflow initialized the local tracking
store:

```text
Detected out-of-date database schema
found version b7e2c1a4d9f3, expected 6f8d9c3b2a1e
```

Therefore retraining was **not fully successful as an MLflow-tracked workflow**,
and no successful `outputs/retraining/retraining_report.json` was produced.
`12_update_mlflow_registry.py` was also executed; it exited with code 1 because
that retraining report was absent. No MLflow database migration was performed.

Final command status:

```text
12_api_performance_monitoring.py  PASS (0)
12_create_evolved_dataset.py     PASS (0)
12_create_concept_drift_dataset.py PASS (0)
12_data_drift.py                 PASS (0)
12_prediction_drift.py           PASS (0)
12_concept_drift.py              PASS (0)
12_drift_analysis.py             PASS (0)
12_performance_monitoring.py     PASS (0)
12_performance_trend.py          PASS (0)
12_create_updated_dataset.py     PASS (0)
12_model_comparison.py           PASS (0)
12_retrain_model.py              BLOCKED (1, MLflow schema)
12_update_mlflow_registry.py     BLOCKED (1, missing retraining report)
```

The pre-execution artifact manifest and backups are in
`outputs/lab12/pre_execution_artifact_manifest.csv` and
`outputs/lab12/pre_execution_backup`.

## Isolated MLflow 3.16.0 compatibility test

The original `mlflow.db` was not modified. A separate virtual environment was
created at `C:\Users\User\.copilot\mlflow316-isolated\venv-system`. The local
MLflow 3.16.0 wheel was installed there with `--no-deps` because the package
index was unavailable; the environment used existing system-site packages.
MLflow emitted a warning that `mlflow-skinny` and `mlflow-tracing` were 3.16.1,
not 3.16.0. This warning is retained in the evidence.

The existing verified backup was copied to
`C:\Users\User\.copilot\mlflow316-isolated\mlflow_working_copy.db`.
Before migration, the copy contained 1 experiment, 2 runs, 1 registered model,
and 1 Production model version.

The command:

```text
python -m mlflow db upgrade sqlite:///C:/Users/User/.copilot/mlflow316-isolated/mlflow_working_copy.db
```

completed with exit code `0` under MLflow 3.16.0. The migration output reported
`Updating database tables`. The working copy remained at revision
`b7e2c1a4d9f3`, which is a supported revision in MLflow 3.16.0
(`b7e2c1a4d9f3_add_jobs_creator.py`). This was therefore a compatibility/no-op
upgrade for this database, not a revision change.

Post-test verification of the working copy:

```text
integrity ok
revision b7e2c1a4d9f3
experiments 1
runs 2
registered_models 1
model_versions 1
production_versions 1
```

The MLflow 3.16.0 client successfully queried the working copy and returned the
existing `Default` experiment, both runs, and
`Telco_Churn_Production_Model` version 1 in `Production`. Logical record
comparisons showed the original database, backup, and working copy were equal.
The original database remained at revision `b7e2c1a4d9f3` with integrity `ok`.

Evidence:

- `outputs/lab12/mlflow316_migration_output.txt`
- `outputs/lab12/mlflow316_post_migration_verification.txt`
- `outputs/lab12/mlflow316_record_comparison.txt`
- `outputs/lab12/mlflow316_revision_details.txt`

The safe compatibility conclusion is to use a fully version-aligned MLflow
3.16.x environment for this database, after separately installing matching
`mlflow-skinny` and `mlflow-tracing` packages. Do not migrate the original
database based on this test; the isolated test established compatibility
without requiring a live-database change.

## New-data and trigger status

The trigger checks `data/raw/new_churn.csv`. That path is absent in this
workspace; the existing root-level `new_churn.csv` was not copied into it, so
the trigger did not start ingestion or retraining. The trigger result was:

```text
status=drift_waiting_for_data
drift_detected=true
new_data_present=false
pipeline_called=false
new_data_ingested=false
retraining_executed=false
```

Final data inspection found that the root batch is not new relative to the
current `churn.csv`: all 100 IDs overlap, specifically `NEW000201` through
`NEW000300`. Its schema matches `churn.csv`, it has 100 unique IDs, no nulls,
numeric `TotalCharges`, `tenure` in `[1, 72]`, and `MonthlyCharges` in
`[20.5, 120.0]`. It was not copied or consumed.

## MLflow troubleshooting

Installed MLflow is `3.15.2`, matching `requirements.txt`. The live SQLite
database is structurally healthy (`PRAGMA integrity_check = ok`) and contains
2 runs and 1 model version, but `alembic_version` is `b7e2c1a4d9f3`.
MLflow 3.15.2's migration graph does not contain that revision; its current
head includes `6f8d9c3b2a1e`.

A pre-migration backup was created at
`outputs/lab12/mlflow.db.pre_migration_backup`. The backup was independently
verified for integrity and matching record counts. Migration was tested only
against the backup and failed safely with:

```text
Can't locate revision identified by 'b7e2c1a4d9f3'
```

The safest supported approach is **not** to run `mlflow db upgrade` with the
current MLflow 3.15.2 installation. First identify the MLflow version/build
that created revision `b7e2c1a4d9f3`, use that matching version in an isolated
environment, and test its migration against the backup. Only after a
successful backup migration and record verification should the live database
be migrated. Until then, the deployment bundle and generated reports are the
safe compatibility path. No live database migration, reset, deletion, or
experiment modification was performed.

## Compatibility note

The flattened workspace initially contained a pre-existing mismatch between
`X_train_raw.csv` (5,935 rows) and `y_train.npy` (5,635 labels). The original
root artifacts were not modified. A consistent compatibility input was created
from the existing labeled `updated_training_data.csv` so the Lab 12 retraining
stage could execute successfully.

## Docker MLflow 3.16.0 compatibility test

After approval, two test-only files were added:

- `Dockerfile.lab12-mlflow316`
- `requirements-lab12-mlflow316.txt`

The existing API `Dockerfile`, `requirements-api.txt`, `deployment.yaml`,
`service.yaml`, and image tags were not changed. The test image was built with
the new tag `churn-prediction-lab12-mlflow:3.16.0-test`. Its base environment
is Python `3.11.18` from `python:3.11-slim`.

Container package verification:

```text
mlflow 3.16.0
mlflow-skinny 3.16.0
mlflow-tracing 3.16.0
numpy 2.4.6
pandas 2.3.3
scikit-learn 1.9.0
joblib 1.5.3
PyYAML 6.0.3
pip check: No broken requirements found.
```

The original database was not mounted. A separate staged workspace and copied
database were used under `C:\Users\User\.copilot\mlflow316-docker-test`.
MLflow tracking against that copy passed:

```text
integrity ok
revision b7e2c1a4d9f3
experiments 1 before retraining
runs 2 before retraining
registered model Telco_Churn_Production_Model
Production version 1
```

Lab 12 retraining executed successfully in the test container with exit code
`0`. It produced V2 artifacts and MLflow run
`692eba742d1248c9a547cd8ffe8c7438`. Metrics were accuracy `0.5678`, precision
`0.2718`, recall `0.3743`, F1 `0.3150`, and ROC-AUC `0.5003`.

The first registry-update attempt exposed an MLflow 3.16.0 security
compatibility requirement: the existing script does not pass
`skops_trusted_types` for the trusted RandomForest tree type, so the attempt
failed before registration. A staged-only copy of the script was tested with
`skops_trusted_types=["sklearn.tree._tree.Tree"]`; the retry exited `0` and
created model version `2` in `Staging`. Promotion correctly remained `NO`
because the configured recall quality gate failed (`0.3743 < 0.70`).
The project source script was not changed.

The original database remained integrity-valid with revision `b7e2c1a4d9f3`,
1 experiment, 2 runs, 1 registered model, and 1 model version. The original
API image tags remain on image ID
`sha256:f1d0e7b723ab5a0a0c14491340e201677f52f878b348873d9a1e91f48bf82ff8`,
and Kubernetes remained `3/3` Ready using `churn-prediction-api:1.0`.

This proves Docker compatibility on a copy, but does **not** mark the project
Lab 12 registry workflow complete: the source registry script still needs the
staged trust parameter, and the recall quality gate still prevents promotion.

Evidence:

- `outputs/lab12/docker_mlflow316_package_check.txt`
- `outputs/lab12/docker_mlflow316_retraining.txt`
- `outputs/lab12/docker_mlflow316_registry_initial_failure.txt`
- `outputs/lab12/docker_mlflow316_registry_retry.txt`
- `outputs/lab12/docker_mlflow316_db_and_deployment_verification.txt`
