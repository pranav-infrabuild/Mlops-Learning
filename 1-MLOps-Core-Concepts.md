# MLOps Core Concepts

These are my notes for the core concepts of MLOps. Written in simple English.

---

## 1. What is MLOps?

MLOps = Machine Learning + Operations.

Just like DevOps brought together software development and operations, MLOps brings together ML model development and production operations.

Simple definition: MLOps is a set of practices to reliably and efficiently deploy and run machine learning models in production.

---

## 2. Why is DevOps alone not enough?

ML has three extra challenges that normal software does not have:

**1. Data dependency**
Software code is reproducible. But an ML model needs the same code AND the same data AND the same hyperparameters AND the same random seed AND the same library versions to be reproducible. So we must version the data too, not just the code.

**2. Model decay (drift)**
Software keeps working the same way until the code changes. But an ML model gets worse over time on its own, because the real-world data changes (for example, fraud patterns keep changing). So we need continuous monitoring and retraining.

**3. Experiment tracking**
Normal software has no concept of "experiments". In ML we always try many experiments (different models, features, settings). We need to track all of them, and be able to reproduce any of them later.

---

## 3. MLOps Lifecycle

The lifecycle has three big parts, and it is a loop (not a straight line):

1. **Data management** - collection, validation, versioning, feature engineering
2. **Model development** - experiments, training, evaluation, registration, approval
3. **Production operations** - deploy, monitor, detect drift, trigger retraining

Production sends feedback back to data management, so the loop continues.

---

## 4. CI / CD / CT

- **CI (Continuous Integration)** - validate data + train model + evaluate
- **CD (Continuous Delivery)** - deploy the model endpoint
- **CT (Continuous Training)** - automatically retrain when drift is detected

CT is the new part that does not exist in normal DevOps.

---

## 5. Reproducibility

In a regulated industry (like banking), a regulator can ask: "Two years ago this model approved this transaction. What was the model and the data at that time? Why did it approve?"

To answer this, we must store for every training run:
- Code version (git commit)
- Data version (DVC tag)
- Hyperparameters (logged in MLflow)
- Environment (Docker image)
- Random seed
- Results (metrics)

With all this stored, we can reproduce the exact same run.

---

## 6. Experiment Tracking

Tools like MLflow track every experiment: parameters, metrics, and the model itself. This lets us compare experiments and pick the best one. Git only tracks code, so we need MLflow for experiments.

---

## 7. Model Registry

A central, versioned store for trained models. Models move through stages (for example: staging -> production). Only approved models go to production.

---

## 8. Feature Store

A central place to store and reuse features. The same features can be used by multiple models (for example, fraud detection and AML detection). It keeps features consistent between training and serving.

---

## 9. Data Versioning

Tools like DVC version datasets like Git versions code. This gives reproducibility and data lineage (we can trace where data came from).

---

## 10. Pipeline Orchestration

Tools like Airflow, Kubeflow, or Azure ML Pipelines run the steps of the ML pipeline in order, on a schedule or on a trigger.

---

## 11. Deployment Strategies

- **Batch inference** - predictions on a big batch of data, on a schedule
- **Real-time / online inference** - predictions one at a time, instantly (API)
- **Shadow deployment** - new model runs alongside old, but its output is not used yet (just compared)
- **Canary deployment** - new model gets a small % of traffic first
- **Blue-green deployment** - two environments, switch traffic from old to new
- **A/B testing** - compare two models on real traffic

---

## 12. Monitoring & Governance

- **Data drift** - input data distribution changes
- **Concept drift** - the relationship between input and output changes
- **Model decay** - model accuracy drops over time

We monitor these and trigger retraining when needed. Governance means audit logs, approvals, access control, and explainability - all very important in banking.

---

## Key Metrics I Learned

| Metric | Question it answers | Why it matters in fraud |
|--------|--------------------|--------------------------|
| Accuracy | Overall, how many correct? | Misleading when data is imbalanced |
| Recall | Of all real frauds, how many did we catch? | Missed fraud = regulatory risk |
| Precision | Of what we flagged, how many were really fraud? | Wrong flags annoy customers |
| F1 score | Balance of precision and recall | Overall health of the model |
| ROC-AUC | How well it separates classes | Good overall quality measure |

In fraud detection, recall is usually the most important, but we balance it with precision (using F1).

---

## What I Built

To practice these concepts I built an end-to-end fraud detection MLOps pipeline:
data generation -> feature engineering -> model training with MLflow -> experiment comparison -> model serving as an API.

The full project is in my separate project repository.
