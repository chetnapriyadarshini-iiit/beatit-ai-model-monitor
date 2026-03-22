# Beatit-AI — Model Monitor (AWS SageMaker Model Monitor)

> **Part of the [Beatit-AI Churn Prediction System](https://github.com/chetnapriyadarshini-iiit)** — a production MLOps system for predicting music streaming subscriber churn on AWS.

This repository implements **automated production monitoring** for the Beatit churn model endpoints using AWS SageMaker Model Monitor — detecting data quality drift, model quality degradation, prediction bias, and feature attribution shifts on both staging and production endpoints.

---

## Table of Contents

- [Overview](#overview)
- [Monitor Types](#monitor-types)
- [Architecture](#architecture)
- [Repository Structure](#repository-structure)
- [Enabling and Disabling Monitors](#enabling-and-disabling-monitors)
- [Technologies Used](#technologies-used)
- [Related Repositories](#related-repositories)
- [Contact](#contact)

---

## Overview

Once a new model version is deployed to a SageMaker endpoint, this monitoring pipeline is automatically triggered when the endpoint status changes to `InService`. It deploys four types of continuous monitors that run on a scheduled basis, comparing live inference data against established baselines to detect degradation before it impacts business outcomes.

Any update to the production endpoint — such as a new model version deployment — re-triggers the monitor pipeline to refresh baselines and configurations automatically.

---

## Monitor Types

| Monitor | What It Detects | Business Value |
|---|---|---|
| **Data Quality Monitor** | Drift in input feature distributions relative to training baseline | Catches upstream data pipeline changes that could silently degrade predictions |
| **Model Quality Monitor** | Drift in model performance metrics (e.g. AUC, accuracy) vs. ground truth | Early warning that the model is becoming less accurate in production |
| **Bias Drift Monitor** (Clarify) | Changes in prediction bias across demographic or feature groups | Ensures fair and consistent predictions across subscriber segments |
| **Feature Attribution Drift Monitor** (Clarify) | Shifts in which features drive predictions over time | Signals concept drift — the world has changed, the model needs retraining |

---

## Architecture

```
SageMaker Production Endpoint
(Status: InService)
         │
         ▼ (Auto-triggered)
  AWS CodePipeline
         │
         ▼
┌────────────────────────────────┐
│  Build Stage                   │  buildspec.yml
│  · get_baselines_and_configs.py│  Fetches baselines from
│  · Retrieve baselines from     │  SageMaker Model Registry
│    Model Registry              │
│  · Merge with config files     │
└───────────────┬────────────────┘
                │
                ▼
┌────────────────────────────────┐
│  Deploy Monitor Stack          │  model-monitor-template.yml
│  (CloudFormation)              │  CloudFormation deployment
│                                │
│  ┌─────────────────────────┐   │
│  │ Staging Monitor Schedule│   │  staging-monitoring-schedule-config.json
│  └─────────────────────────┘   │
│  ┌─────────────────────────┐   │
│  │  Prod Monitor Schedule  │   │  prod-monitoring-schedule-config.json
│  └─────────────────────────┘   │
└────────────────────────────────┘
         │  Scheduled runs
         ▼
  CloudWatch Alerts on violations
```

---

## Repository Structure

```
beatit-ai-model-monitor/
├── pipelines/                              # Monitor pipeline logic
├── get_baselines_and_configs.py            # Retrieves baselines from Model Registry
│                                           # and populates config files
├── model-monitor-template.yml             # CloudFormation template for all 4 monitors
├── staging-monitoring-schedule-config.json # Staging monitor configuration
├── prod-monitoring-schedule-config.json    # Production monitor configuration
├── utils.py                               # Helper functions
├── buildspec.yml                          # CodeBuild instructions
└── __init__.py
```

---

## Enabling and Disabling Monitors

Each monitor type can be individually enabled or disabled in the configuration files:

```json
{
  "Parameters": {
    "EnableDataQualityMonitor": "yes",
    "EnableModelQualityMonitor": "yes",
    "EnableModelBiasMonitor": "yes",
    "EnableModelExplainabilityMonitor": "yes"
  }
}
```

For `ModelQuality` and `ModelBias` monitors, a ground truth S3 URI must be provided:

```json
{
  "Parameters": {
    "GroundTruthInput": "s3://your-bucket/ground-truth/"
  }
}
```

---

## Technologies Used

| Service / Tool | Purpose |
|---|---|
| **AWS SageMaker Model Monitor** | Automated scheduled monitoring of production endpoints |
| **AWS SageMaker Clarify** | Bias detection and feature attribution analysis |
| **AWS CloudFormation** | Infrastructure-as-code for monitor deployment |
| **AWS CodePipeline / CodeBuild** | CI/CD trigger and build execution |
| **Amazon CloudWatch** | Alerting on monitor violations |
| **SageMaker Model Registry** | Source of training baselines for drift comparison |
| **Python** | Baseline retrieval and configuration logic |

---

## Related Repositories

| Repository | Role |
|---|---|
| [beatit-ai-glue-redshift-tables](https://github.com/chetnapriyadarshini-iiit/beatit-ai-glue-redshift-tables) | Upstream data pipeline |
| [beatit-ai-model-train](https://github.com/chetnapriyadarshini-iiit/beatit-ai-model-train) | Produces training baselines used by this monitor |
| [beatit-ai-model-deploy](https://github.com/chetnapriyadarshini-iiit/beatit-ai-model-deploy) | Deploys the endpoints monitored here |
| [beatit_ai_common_utilites](https://github.com/chetnapriyadarshini-iiit/beatit_ai_common_utilites) | Shared utilities |

---

## Contact

Created by [@chetnapriyadarshini](https://github.com/chetnapriyadarshini) — feel free to reach out with questions or suggestions.
