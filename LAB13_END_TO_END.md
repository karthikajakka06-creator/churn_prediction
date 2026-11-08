# Lab 13 — End-to-End MLOps Integration Report

## Objective

Integrate data versioning and validation, preprocessing, training, MLflow tracking/model registry, evaluation and quality gates, lifecycle decisions, deployment, monitoring, retraining triggers, reproducibility, and engineering documentation into one traceable system.

## Integrated execution

GitHub Actions runs on a self-hosted Windows runner and executes `src/mlops_pipeline/12_monitor_trigger.py` from the project virtual environment. The trigger runs drift analysis, checks for `data/raw/new_churn.csv`, and invokes `src/mlops_pipeline/churn_prediction_pipeline.py` when drift and a batch are present.

The main pipeline runs these stages:

1. Data Loading
2. Raw Data Validation
3. Preprocessing
4. Processed Output Validation
5. Training + MLflow + Model Registry
6. Model Evaluation
7. Model Quality Gate
8. Automated Model Lifecycle
9. Deployment Update (promotion-gated)
10. Production Inference

If the batch was already ingested, the pipeline stops safely rather than training again on the same data.

## Validation evidence

### Duplicate-batch behavior

A monitoring run detected drift and found `new_churn.csv`, but the loader reported `already_ingested`. The monitoring report showed `pipeline_status: passed`, `new_data_ingested: false`, and `retraining_executed: false`. This demonstrates idempotent handling of a previously consumed batch.

### New-batch ingestion and retraining

A simulated 100-row batch with distinct IDs `NEW000201`–`NEW000300` passed schema and identity validation. The loader archived the prior dataset as `churn_v03.csv`, appended the new rows, and increased the current raw dataset from 7,244 to 7,344 rows.

### Drift observations from the tested run

- Data drift was reported for `MonthlyCharges` and `TotalCharges`.
- Prediction-distribution KS statistic: 0.101490 (status `DRIFT`).
- Probability-distribution KS statistic: 0.229950 (status `DRIFT`).
- Four monitored concept/performance metrics were reported as degraded: accuracy, recall, F1, and ROC-AUC.

These are recorded demonstration results, not current live measurements.

### Candidate training and quality gate

Candidate model version 17 was trained and registered in MLflow:

| Metric | Candidate v17 |
|---|---:|
| Accuracy | 0.7672 |
| Recall | 0.7219 |
| F1 | 0.6221 |
| ROC-AUC | 0.8452 |

The recall quality gate required at least 0.70, so the candidate passed the gate.

### Lifecycle and deployment decision

Production version 1 had recall approximately 0.78075, higher than candidate v17's 0.7219. The lifecycle manager retained Production v1 and archived v17. The promotion-gated deployment step saw `promoted: false` and skipped deployment. Production inference then ran using the registered Production model.

This is an intentionally correct lifecycle outcome: passing the minimum quality gate does not automatically promote a candidate that fails to improve on Production.

### Deployment and Kubernetes

The standalone production-deployment automation was tested with Production model v1. It reused `churn-prediction-api:model-v1`, updated the Kubernetes Deployment, and reported a successful rollout. Kubernetes was observed with 3/3 replicas ready and three pods running during the recorded verification.

The candidate-v17 `promoted=true` path has not been demonstrated because the real candidate was correctly rejected. The non-promotion branch was verified in the integrated run, and the deployment mechanism was independently tested with Production v1.

## Important artifacts

| Artifact | Purpose |
|---|---|
| `artifacts/data_load_report.json` | Ingestion outcome, hash, and row counts. |
| `data/raw/archive/dataset_lineage.json` | Batch hashes and dataset lineage. |
| `outputs/drift/drift_analysis_report.csv` | Combined drift summary. |
| `outputs/monitoring/monitor_trigger_report.json` | Monitoring/retraining trigger decision. |
| `reports/evaluation_report.json` | Candidate model evaluation metrics. |
| `reports/lifecycle_report.json` | Candidate-versus-Production decision. |
| `reports/deployment_automation_report.json` | Deployment image, model version, Kubernetes target, and status. |
| `deployment/model_metadata.json` | Identity and lineage of the packaged Production model. |
| `logs/churn_prediction_pipeline_run_log.json` | Stage-by-stage pipeline status. |

Some runtime reports live in ignored local output directories. Their existence on the runner does not mean every report is committed to Git.

## GitHub Actions validation

The self-hosted runner used labels `self-hosted`, `Windows`, and `X64`. A diagnostic workflow successfully checked Python, Git, Docker, `kubectl`, and the project path. The end-to-end workflow then successfully ran monitoring and retraining through GitHub Actions.

The monitoring workflow is merged into `main` and is configured with `workflow_dispatch` and an hourly schedule at minute 17 in `Asia/Kolkata`. A successful scheduled run has not yet been confirmed in the recorded evidence; manual and push-triggered runs were used for integration testing.

## Limitations and follow-up

- The incoming batch was simulated data, not a feed from an external production source.
- The runner currently runs interactively via `C:\actions-runner\run.cmd`; the terminal must remain open and the computer must remain awake and online.
- Scheduled execution therefore depends on the runner being online at the scheduled time.
- The promotion-positive deployment path remains unverified with a naturally better candidate.
- Deployment/Kubernetes checks require Docker Desktop and Kubernetes to be running.

## Local execution evidence

## Latest verification

The existing Lab 13 scripts were executed locally after the Lab 12 monitoring
chain. Evidence transcripts and per-script exit codes are stored in
`outputs/lab13`.

The final requested execution passed these scripts with exit code 0:

- `12_request_history.py`
- `12_prediction_monitoring.py`
- `12_api_performance_monitoring.py`
- `12_monitor_trigger.py`

The legacy `09_inference.py` entry point was executed and exited with code 1:

```text
Could not load Production model from MLflow:
Detected out-of-date database schema
found version b7e2c1a4d9f3, expected 6f8d9c3b2a1e
```

No database migration was performed. To validate the actual deployed model
without claiming the failed MLflow path passed, the existing
`deployment/model.pkl`, `deployment/preprocessor.pkl`, and
`deployment/model_metadata.json` bundle was loaded through `api.model_loader`.
Ten real held-out rows were scored and written to
`logs/inference/inference_log.jsonl`.

The following existing Lab 13 scripts then passed with exit code 0:

- `12_prediction_monitoring.py`
- `12_request_history.py`
- `12_api_performance_monitoring.py`
- `12_monitor_trigger.py`

Final Lab 13 command status:

```text
12_request_history.py          PASS (0)
12_prediction_monitoring.py    PASS (0)
12_api_performance_monitoring.py PASS (0)
12_monitor_trigger.py          PASS (0)
```

The trigger checked `data/raw/new_churn.csv`, which is absent. The root-level
`new_churn.csv` was inspected but not copied because its customer IDs are
already present in `churn.csv`; it is duplicate data, not a new incoming batch.

Production-bundle inference generated ten successful records, and prediction
distribution, request-history, and API-performance monitoring all completed.
The monitoring trigger completed with:

```text
status: drift_waiting_for_data
drift_detected: true
new_data_present: false
pipeline_called: false
retraining_executed: false
```

This is the expected safe branch when drift is present but no new incoming
batch is available. Kubernetes remained healthy with the Lab 11 deployment at
3/3 ready replicas.

## End-to-end status

Passed:

- Inference history generation from the existing deployment bundle
- Prediction logging to `logs/inference/inference_log.jsonl`
- Prediction-distribution monitoring
- Request-history inspection
- API latency/success monitoring
- Drift analysis and monitoring-trigger validation
- Kubernetes health check remained 3/3 replicas Ready
- Kubernetes Service API validation: `/health` and `/metadata` returned HTTP 200

Failed or blocked:

- MLflow-backed `09_inference.py`, blocked by the existing SQLite schema
- Full retraining/registry path, blocked by the same MLflow schema and the
  missing retraining report after that failure

No fake predictions, dummy datasets, MLflow migrations, or previous-lab changes
were introduced during this verification.

## Final troubleshooting status

The root `new_churn.csv` was inspected but not copied into `data/raw/`. The
expected trigger path is absent, and all 100 root-batch customer IDs already
exist in the current `churn.csv`, so consuming it would be duplicate ingestion.

The Docker image remains available with tags `churn-prediction-api:1.0` and
`churn-prediction-api:lab11-final`. The historical standalone container named
`churn-prediction-api` is currently exited, but the Lab 11 Kubernetes deployment
is healthy at 3/3 Ready replicas and serves the API successfully through the
Kubernetes Service.

The MLflow-backed paths remain blocked by the incompatible database revision
described in the Lab 12 report. The live database was not changed.
