# 2 - MLOps Tools Deep Dive

Comprehensive notes covering production-grade MLOps tools. These notes are designed so anyone can learn from scratch.

---

## Table of Contents

1. [Introduction](#introduction)
2. [Docker - Containerization](#docker---containerization)
3. [Evidently - Model Drift Detection](#evidently---model-drift-detection)
4. [DVC - Data Version Control](#dvc---data-version-control)
5. [Azure ML - Cloud ML Platform](#azure-ml---cloud-ml-platform)
6. [Summary & Comparison](#summary--comparison)

---

## Introduction

In Phase 4, I built a fraud detection model with MLflow experiment tracking. The model worked great on my laptop. But real production systems need more:

- **Portability**: Model should run on any machine (not just mine)
- **Monitoring**: Know when model performance degrades
- **Reproducibility**: Recreate exact training data from 6 months ago
- **Scale**: Train on powerful cloud machines, not limited laptop

Phase 5 adds these production capabilities using industry-standard tools. By the end, my fraud detection project went from "works on my machine" to "production-ready cloud deployment."

---

# Docker - Containerization

## What is Docker?

**Simple definition:** Docker packages your application + all its dependencies into a "container" - a lightweight, portable box that runs the same way everywhere.

**Real-world analogy:** Think of shipping containers. Before containers, shipping was a nightmare - different goods needed different handling. Containers standardized everything. Similarly, Docker standardizes how software runs across different environments.

---

## Why is Docker needed for ML?

### Problem: "It works on my machine"

I trained my fraud detection model on my laptop:
- Windows 10
- Python 3.11
- Specific versions of numpy, pandas, scikit-learn
- Flask for the API

When I send this to production:
- Different OS (maybe Linux)
- Different Python version
- Different package versions
- Dependencies might conflict

**Result:** Model that worked perfectly on my laptop crashes in production. Or worse - gives different predictions (silent failure).

### The Docker solution

Docker creates a **complete snapshot** of your application environment:
- Base operating system (Linux)
- Python version
- All packages with exact versions
- Your code and model files

This snapshot (called an "image") runs the same way **everywhere** - your laptop, colleague's machine, cloud server, Kubernetes cluster.

---

## Key Docker Concepts

### 1. Docker Image

A **template** or blueprint. Like a recipe.

Example: My `fraud-api:v1` image contains:
- Python 3.11 on Linux
- Requirements: numpy, pandas, scikit-learn, flask
- My API code (`src/predict.py`)
- Trained model (`models/fraud_model.pkl`)

**Images are immutable** - once built, they don't change. If you need changes, you build a new image (version 2, 3, etc.).

### 2. Docker Container

A **running instance** of an image. Like a cake made from the recipe.

One image can create multiple containers:
- Container 1: API running on port 5001
- Container 2: API running on port 5002
- Container 3: API running on a cloud server

Each container is **isolated** - they don't interfere with each other.

### 3. Dockerfile

A **recipe file** that tells Docker how to build your image.

My Dockerfile for fraud detection API:

```dockerfile
FROM python:3.11-slim          # Start with Python base
WORKDIR /app                   # Set working directory
COPY requirements.txt .         # Copy dependencies list
RUN pip install -r requirements.txt  # Install packages
COPY src/ ./src/               # Copy code
COPY models/ ./models/         # Copy model
EXPOSE 5001                    # API will use this port
CMD ["python", "src/predict.py"]  # Run this on start
```

**Each line is a step.** Docker executes them top to bottom.

---

## How Docker Works (Simple Flow)

```
1. Write Dockerfile (recipe)
2. docker build → Creates image (snapshot)
3. docker run → Creates container (running app)
4. Container serves predictions
```

**In my project:**
```bash
docker build -t fraud-api:v1 .     # Build image
docker run -p 5001:5001 fraud-api:v1  # Run container
# API now live at http://localhost:5001
```

---

## Advantages of Docker for ML

| Problem | Without Docker | With Docker |
|---------|---------------|-------------|
| **Environment setup** | Install Python, packages manually on every machine (30+ minutes, error-prone) | One command: `docker run` (30 seconds) |
| **"Works on my machine"** | Code works locally, fails in production due to environment differences | Container is identical everywhere |
| **Dependency conflicts** | Package X needs numpy 1.x, package Y needs numpy 2.x - conflict! | Each container has isolated dependencies |
| **Onboarding new team members** | New developer spends hours setting up environment | New developer: `docker run` - done |
| **Versioning** | Hard to track which environment was used for which model | Image tags: `fraud-api:v1`, `fraud-api:v2` - clear versions |
| **Scaling** | Deploy on multiple servers? Install everything on each | Same Docker image on all servers |

---

## Docker in Production (Banking Context)

In the Spectre AI project (my work):

**Before Docker:**
- Data scientists train models locally
- DevOps team manually sets up production servers
- "It worked in dev, why not in prod?" - constant firefighting
- Model updates need environment reconfiguration

**After Docker:**
- Data scientists build Docker image with model
- Push image to Azure Container Registry
- Kubernetes pulls image and deploys
- Model updates = new Docker image version
- Rollback = switch to previous image version

**Compliance benefit:** Docker images are **immutable and versioned**. AUSTRAC audit: "Which environment was used for model v2.3?" → Image `fraud-model:v2.3` has exact snapshot.

---

## Dockerfile Best Practices (ML Projects)

### 1. Use slim base images
```dockerfile
FROM python:3.11-slim  # 40MB
# vs
FROM python:3.11       # 900MB
```
Smaller images = faster deployment.

### 2. Layer caching (optimize build time)
```dockerfile
# GOOD - requirements change less often than code
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY src/ ./src/  # Code changes often

# BAD - code change forces pip reinstall
COPY src/ ./src/
COPY requirements.txt .
RUN pip install -r requirements.txt
```

### 3. Don't include unnecessary files
Use `.dockerignore`:
```
venv/
.git/
*.pyc
mlruns/
```

---

## Common Docker Commands

| Command | What it does | Example |
|---------|--------------|---------|
| `docker build -t name:tag .` | Build image from Dockerfile | `docker build -t fraud-api:v1 .` |
| `docker run -p host:container image` | Run container, map ports | `docker run -p 5001:5001 fraud-api:v1` |
| `docker ps` | List running containers | See active API containers |
| `docker images` | List available images | See all built images |
| `docker stop <id>` | Stop a container | Stop API gracefully |
| `docker push name:tag` | Upload image to registry | `docker push myrepo/fraud-api:v1` |
| `docker pull name:tag` | Download image | `docker pull myrepo/fraud-api:v1` |

---

## Interview Questions - Docker

**Q1: What is the difference between a Docker image and a container?**

**A:** An image is a read-only template (blueprint) containing the application and its dependencies. A container is a running instance of that image. One image can create multiple containers. Analogy: Image is like a class in programming, container is like an object/instance.

**Q2: Why use Docker for ML models instead of just sharing code?**

**A:** Code alone doesn't guarantee reproducibility. The same code can produce different results with different package versions, OS, Python versions. Docker bundles everything - code, dependencies, environment - ensuring identical behavior everywhere. Critical for ML where small differences can change predictions.

**Q3: How does Docker help with model versioning?**

**A:** Each Docker image can be tagged with a version (`fraud-model:v1.0`, `v1.1`, etc.). The image is immutable - it captures the exact state of code, model, and dependencies at that point. You can deploy multiple versions side-by-side, roll back instantly, or compare predictions from different versions.

**Q4: What is the difference between Docker and a virtual machine?**

**A:** VMs virtualize hardware - each VM runs a full OS (heavy, GB-sized, slow startup). Docker containers virtualize the OS - they share the host OS kernel (lightweight, MB-sized, instant startup). For ML models, containers are ideal: faster deployment, less overhead, same isolation benefits.

---

# Evidently - Model Drift Detection

## What is Evidently?

**Simple definition:** Evidently is a Python library that detects when your production data is different from your training data. If the difference is too large (drift), your model's predictions become unreliable.

**Real-world analogy:** Imagine you trained a weather prediction model using data from summer. If you use it in winter, it will give wrong predictions because the input data (temperature patterns) has "drifted" from what it learned. Evidently detects such drift.

---

## Why is Model Drift Monitoring Needed?

### The Problem: Models Degrade Over Time

Unlike traditional software, ML models **decay automatically** even if code doesn't change.

**Example from fraud detection:**

- **January 2025:** I train a model on fraud patterns from 2024
  - Fraudsters use stolen card numbers
  - High-value single transactions
  - Model learns these patterns → 95% accuracy

- **June 2025:** Fraud patterns change
  - Fraudsters switch to "smurfing" (many small transactions to avoid detection)
  - Model still looks for high-value transactions
  - **Accuracy drops to 70%** - but no one notices immediately!

This is **concept drift** - the relationship between inputs and outputs changed.

### Types of Drift

#### 1. Data Drift (Feature Drift)
Input data distribution changes, but the concept stays the same.

**Example:**
- Training data: Average transaction $50, 80% day-time
- Production data: Average transaction $150, 60% day-time
- Features shifted, even though fraud = same concept

#### 2. Concept Drift
The relationship between input and output changes.

**Example:**
- Before: Transaction > $1000 at night = 80% fraud
- Now: Fraudsters learned, avoid large amounts
- Now: Transaction > $1000 at night = 20% fraud
- **Same input, different meaning**

#### 3. Prediction Drift
Model's output distribution changes unexpectedly.

**Example:**
- Training: 2% transactions flagged as fraud
- Production: Suddenly 20% flagged as fraud
- Something changed - need investigation

---

## How Evidently Works

Evidently compares two datasets:

1. **Reference dataset** (usually training data or recent good period)
2. **Current dataset** (recent production data)

It checks:
- Are feature distributions similar?
- Are there missing values where there weren't before?
- Has the range of values changed?
- Statistical tests (like Kolmogorov-Smirnov test) for each feature

**Output:** Visual HTML report + metrics (% of features drifted).

---

## Using Evidently - Step by Step

### Setup

```python
from evidently.report import Report
from evidently.metric_preset import DataDriftPreset

# Load reference data (training data)
reference = pd.read_csv('data/training.csv')

# Load production data (last week's transactions)
production = pd.read_csv('data/production_week.csv')
```

### Generate Drift Report

```python
# Create report
report = Report(metrics=[DataDriftPreset()])

# Run comparison
report.run(
    reference_data=reference,
    current_data=production
)

# Save as HTML
report.save_html('reports/drift_report.html')
```

### Set Up Alerts

```python
# Get drift metrics
result = report.as_dict()
drift_share = result['metrics'][0]['result']['share_of_drifted_columns']

# Alert if > 30% features drifted
if drift_share > 0.3:
    print("ALERT: Significant drift detected!")
    print("Recommendation: Retrain model")
    # Send alert to Slack/email
```

---

## Advantages of Evidently

| Without Evidently | With Evidently |
|-------------------|----------------|
| **No visibility:** Model silently degrades | **Proactive:** Drift detected before accuracy drops |
| **Reactive:** Discover poor performance from complaints | **Data-driven:** Know exactly which features changed |
| **Manual checks:** Manually compare data (time-consuming) | **Automated:** Scheduled reports (daily/weekly) |
| **Guesswork:** "Why is the model failing?" | **Root cause:** Visual reports show which feature drifted |
| **Blind deployments:** Hope new data works | **Confident:** Deploy knowing data is similar to training |

---

## Evidently in Production (Banking)

In my Spectre AI project context:

**Scenario:** Fraud detection model deployed in January.

**Without Evidently:**
- Model runs for months
- Fraud team notices: "We're missing frauds lately"
- Investigation starts - but what changed? Data? Model? Code?
- Weeks to debug

**With Evidently:**
- Automated weekly drift check
- Week 12: Alert "30% features drifted - transaction amounts shifted"
- Investigation immediate: "New merchant (Amazon Prime subscriptions) adding legitimate $9.99 recurring transactions"
- Solution: Retrain model with recent data, or add feature engineering to handle subscriptions
- **Total time: 1 day instead of weeks**

**Compliance benefit:** AUSTRAC audit: "Your model missed these frauds in March. Why?"

Without Evidently: "We don't know, investigating..."

With Evidently: "Our drift detection showed data distribution changed on March 5 due to merchant category shifts. We retrained the model on March 12. Here's the report."

---

## Drift Detection Strategy

### When to check for drift?

| Frequency | Use case |
|-----------|----------|
| **Real-time** | High-risk systems (credit card transactions) |
| **Daily** | Most production models |
| **Weekly** | Stable data (B2B fraud, slower-changing patterns) |
| **On-demand** | Before retraining, after major data pipeline changes |

### What threshold to use?

**Rule of thumb:**
- **10% features drifted:** Monitor closely
- **20-30% drifted:** Plan retraining soon
- **>30% drifted:** Urgent retraining needed

**But it depends:**
- Critical models (healthcare, fraud): Lower threshold (15%)
- Non-critical (recommendation): Higher threshold (40%)

---

## Common Mistakes

1. **Comparing unequal sample sizes**
   - Bad: 100K reference vs 100 current
   - Good: Equal samples, or weight appropriately

2. **Ignoring target drift**
   - Check: Is fraud rate 2% (like training) or suddenly 10%?
   - Target drift often indicates data quality issues

3. **Not acting on alerts**
   - Drift detection without retraining = wasted effort
   - Set up automated retraining pipelines

4. **Checking drift but not logging it**
   - Save reports for audit trail (compliance)

---

## Interview Questions - Evidently

**Q1: What is the difference between data drift and concept drift?**

**A:** Data drift is when the input feature distributions change (e.g., average transaction amount shifts from $50 to $100), but the relationship to the target stays same. Concept drift is when the relationship between inputs and target changes (e.g., previously transaction > $1000 = fraud, now it doesn't). Data drift is easier to detect (compare distributions); concept drift requires monitoring model performance.

**Q2: How do you decide when a model needs retraining?**

**A:** Multiple signals: (1) Data drift: if >30% of features show significant drift (2) Performance drift: if accuracy/precision/recall drops below threshold (3) Prediction drift: if output distribution changes dramatically (4) Time-based: regular retraining (monthly/quarterly) even without drift. In banking, combine all signals - compliance requires audit trail of why and when retraining happened.

**Q3: Can you use Evidently for target/label drift?**

**A:** Yes! If you have ground truth labels for production data (e.g., fraud labels after investigation), you can use Evidently's `TargetDriftPreset` to check if the fraud rate changed. However, in many cases you don't have labels immediately (predictions are real-time, labels come later). For this, use `PredictionDriftPreset` to monitor if the model's prediction distribution shifted.

**Q4: Why not just retrain the model every week automatically?**

**A:** Frequent unnecessary retraining has costs: (1) Compute resources (2) Risk of introducing bugs (3) Model behavior changes - users/systems adapted to current model (4) Compliance: every model version needs approval/audit. Drift detection makes retraining **data-driven** - retrain when needed, not on arbitrary schedule. However, combining scheduled retraining (quarterly) + drift-based retraining (as needed) is a good strategy.

---

# DVC - Data Version Control

## What is DVC?

**Simple definition:** DVC (Data Version Control) is like Git, but for data files and ML models. It tracks large files without bloating your Git repository.

**Real-world analogy:** Imagine writing a book with illustrations. Git tracks text (small files) well. But illustrations (large images) make Git slow. DVC handles illustrations separately while Git tracks "which illustration goes where." Together they give you complete version control.

---

## Why is DVC Needed?

### Problem: Git is Not Designed for Large Files

Git works great for code (text files, small). But ML projects have:
- Datasets: 100 MB to 100 GB+
- Trained models: 50 MB to 10 GB+
- Images, videos: Each file MB to GB range

**What happens if you put large files in Git:**
1. Repository becomes **huge** (cloning takes hours)
2. Git slows down (every operation loads everything)
3. GitHub/GitLab have **file size limits** (100 MB typically)
4. History bloats - even if you delete file later, it stays in history

**Example:**
- Code repo: 10 MB (fast, manageable)
- Add 1 GB dataset to Git: Now 1.01 GB
- Update dataset 5 times: Now 5 GB+ (Git stores every version)
- Clone time: hours, Git commands slow

### The DVC Solution

DVC stores large files **separately** (in cache/cloud). Git stores only **small pointer files** (~200 bytes).

**Workflow:**
```
1. Add data file to DVC: dvc add data/dataset.csv
2. DVC creates:
   - Actual data → stored in .dvc/cache/
   - Pointer file → data/dataset.csv.dvc (200 bytes)
3. Git tracks only the pointer: git add data/dataset.csv.dvc
4. Anyone clones repo: gets pointer, runs `dvc pull` to get actual data
```

**Result:**
- Git repo stays small (MB range)
- Data is versioned
- Team collaborates on data like code

---

## Key DVC Concepts

### 1. DVC Files (.dvc)

Small metadata files that Git tracks.

**Example:** `data/creditcard.csv.dvc`
```yaml
outs:
- md5: a3f42bc1e5d...
  size: 2379531
  path: creditcard.csv
```

**Contains:**
- File hash (unique identifier)
- File size
- Original filename

This goes in Git. Actual data goes in DVC cache.

### 2. DVC Cache

Local storage (`.dvc/cache/`) where actual data files live.

**Structure:**
```
.dvc/cache/
  a3/
    f42bc1e5d...  # <- your creditcard.csv (renamed by hash)
```

Files are **content-addressed** (filename = hash). Same file = same hash = stored once.

### 3. Remote Storage (Optional)

DVC can push/pull data to:
- Cloud: AWS S3, Azure Blob, Google Cloud Storage
- Network: SSH server, NAS
- Local: External drive

**Workflow with remote:**
```
Your laptop  →  dvc push  →  S3 bucket
                               ↓
Colleague's laptop  ← dvc pull ←
```

---

## Using DVC - Step by Step

### Initialize DVC

```bash
cd my-project
git init              # Git first
dvc init              # Then DVC
```

Creates `.dvc/` folder (like `.git/`).

### Track a Data File

```bash
dvc add data/raw/transactions.csv
```

**What happens:**
1. File moved to `.dvc/cache/`
2. Pointer file created: `data/raw/transactions.csv.dvc`
3. Original file added to `.gitignore` (so Git ignores it)

**Now:**
```bash
git add data/raw/transactions.csv.dvc data/raw/.gitignore
git commit -m "Track transactions dataset with DVC"
```

### Update Data (New Version)

```bash
# Replace file with new data
cp new_transactions.csv data/raw/transactions.csv

# Track new version
dvc add data/raw/transactions.csv

# Commit pointer file (new hash)
git add data/raw/transactions.csv.dvc
git commit -m "Update transactions dataset - added Feb data"
```

**DVC now has 2 versions in cache.** Git has 2 commits with different pointer files.

### Go Back to Old Version

```bash
git checkout HEAD~1 data/raw/transactions.csv.dvc  # Get old pointer
dvc checkout                                         # Get old data
```

DVC fetches the file matching the old pointer from cache.

---

## DVC with Remote Storage

### Configure Remote

```bash
dvc remote add -d myremote s3://my-bucket/dvc-storage
# or Azure:
dvc remote add -d myremote azure://mycontainer/path
```

### Push Data to Remote

```bash
dvc push
```

Uploads data from local cache to S3/Azure. Team members can now pull.

### Pull Data from Remote

```bash
git clone https://github.com/me/my-project.git
cd my-project
dvc pull  # Downloads data from remote to local cache
```

---

## Advantages of DVC

| Aspect | Without DVC | With DVC |
|--------|-------------|----------|
| **Repository size** | GB-sized (bloated with data) | MB-sized (only code + pointers) |
| **Clone speed** | Hours (downloading all data) | Minutes (just code), then `dvc pull` for data |
| **Data versioning** | Manual: `dataset_v1.csv`, `dataset_v2.csv` (error-prone) | Automatic: Git commits = data versions |
| **Team collaboration** | Email datasets, Dropbox, confusion | Unified: code + data versioned together |
| **Reproducibility** | "Which data was used 6 months ago?" Hard to answer | `git checkout <old-commit>; dvc checkout` |
| **Storage** | Data in Git history forever (bloat) | Old data in cache, easy to clean |
| **Audit trail** | Manual logs | Git log shows data changes with code |

---

## DVC in Production (Banking)

### Scenario: AUSTRAC Audit

**Question from regulator:** "Your model flagged transaction XYZ as fraud on March 15, 2024. What training data was used? Prove the model was valid."

**Without DVC:**
- Developer: "Uh, let me check... I think it was the February dataset?"
- Search emails, Slack, file servers
- "We can't find the exact data anymore"
- **Compliance risk**

**With DVC:**
```bash
# Find commit from March 15
git log --since="2024-03-01" --until="2024-03-20" --grep="model"
# Commit abc123: "Deploy fraud model v2.4"

# Checkout that state
git checkout abc123
dvc checkout

# Now you have EXACT data that was used
# Hash matches → guaranteed same data
# Show regulator: training_data.csv + commit log
```

**Audit passed.** Data lineage proven.

---

## DVC Best Practices

### 1. Don't track tiny files
```bash
# Bad: tracking 1KB config files with DVC (overhead)
dvc add config.json

# Good: track with Git directly
git add config.json
```

**Rule:** DVC for files >10 MB. Git for everything else.

### 2. Use meaningful commit messages
```bash
# Bad
git commit -m "update data"

# Good
git commit -m "Add Q1 2025 transactions (500K rows) - includes new merchant categories"
```

### 3. Setup remote from day 1
Local cache can be lost (laptop dies). Remote = backup + team collaboration.

### 4. Automate DVC in CI/CD
```yaml
# GitHub Actions example
- run: dvc pull
- run: python train.py  # Uses versioned data
- run: dvc push         # Push outputs (trained model)
```

---

## DVC vs Alternatives

| Tool | Purpose | When to use |
|------|---------|-------------|
| **DVC** | Version data with Git-like interface | Most ML projects |
| **Git LFS** | Store large files in Git | If you must use Git for everything (not recommended for ML) |
| **MLflow Artifacts** | Store model artifacts per experiment | Model files only, not datasets |
| **Delta Lake** | Version data in data lakes | Big data pipelines (Spark) |
| **Manual versioning** | Rename files: data_v1, data_v2 | Never (too error-prone) |

---

## Interview Questions - DVC

**Q1: How does DVC work with Git?**

**A:** DVC integrates with Git but stores data separately. When you `dvc add` a file, DVC stores the actual data in `.dvc/cache/` and creates a small metadata file (.dvc extension) that Git tracks. This metadata file contains the data file's hash and path. Git versioning of the .dvc file = data versioning. Running `dvc checkout` syncs data files to match the .dvc files in your current Git branch.

**Q2: What happens if two team members edit the same data file?**

**A:** DVC doesn't handle merge conflicts for data (data files are binary, can't merge like text). Best practice: one person owns each dataset. If conflicts occur, Git shows conflict in the .dvc file, but data itself can't merge - you choose one version or recreate data. For datasets that change frequently, consider versioning with timestamps in filename: `data_2025_01.csv`, `data_2025_02.csv`.

**Q3: Can you use DVC without remote storage?**

**A:** Yes, DVC works purely local (cache in `.dvc/cache/`). Useful for solo projects or when data is private/sensitive. But: (1) No backup - if cache deleted, data lost (2) No collaboration - team can't `dvc pull`. For production, always use remote storage (S3, Azure Blob, etc.) for backup and team access.

**Q4: How do you handle very large datasets with DVC?**

**A:** For TB-scale data: (1) Split into chunks: instead of one 1TB file, 100x 10GB files (2) Use DVC's external dependencies: data stays in data lake (S3), DVC tracks metadata only (3) Use `dvc import-url` for read-only external data (4) Combine with data lake tools (Delta Lake, Iceberg) for large-scale versioning, use DVC for metadata + models.

---

# Azure ML - Cloud ML Platform

## What is Azure ML?

**Simple definition:** Azure ML is Microsoft's cloud platform for the complete ML lifecycle - from data prep to training to deployment to monitoring - all in one place.

**Real-world analogy:** Imagine building a car. You could buy parts separately (engine from one vendor, wheels from another, assembly tools from elsewhere) and build it yourself. Or you could go to a car factory where everything is integrated. Azure ML is the factory - all tools in one ecosystem.

---

## Why is a Cloud ML Platform Needed?

### Problem: Laptop Limitations

**My laptop:**
- CPU: 4 cores
- RAM: 16 GB
- Training time: 30 minutes for fraud model (small dataset)

**Enterprise ML (Spectre AI):**
- Dataset: 10 million transactions (5 GB)
- Features: 200+
- Training time on my laptop: 10+ hours (if it doesn't crash)

**Cloud compute:**
- 64 cores
- 256 GB RAM
- GPU for deep learning
- Training time: 20 minutes
- **Cost:** Pay only when used (auto-shuts down)

### Problem: Team Collaboration

**Local training:**
- Everyone on their own laptop
- "It worked on my machine" syndrome
- No visibility into others' experiments
- Difficult to share models

**Azure ML:**
- Central platform for whole team
- Everyone sees all experiments
- Share models via registry
- Consistent environment

### Problem: Production Deployment

**Manual deployment:**
- SSH into server
- Install Python, packages
- Copy model file
- Setup Flask server
- Configure reverse proxy
- Monitor logs manually

**Azure ML:**
- One click deployment
- Auto-scaling
- Built-in monitoring
- API endpoints ready
- Rollback in seconds

---

## Key Azure ML Components

### 1. Workspace

**What:** The central hub. Everything lives here - data, compute, experiments, models, endpoints.

**Think of it as:** Your project folder, but in the cloud.

**Contains:**
- Compute resources
- Datasets
- Experiments/jobs
- Model registry
- Endpoints
- Pipelines

**In practice:**
- Spectre AI has workspace: `spectre-prod-workspace`
- Dev team has: `spectre-dev-workspace`
- Separation of dev/prod

### 2. Compute

**What:** The machines where code runs.

**Types:**

| Compute Type | Use Case | Cost | Persistence |
|--------------|----------|------|-------------|
| **Compute Instance** | Interactive development (like VM with Jupyter) | $$ (24/7) | Stays on |
| **Compute Cluster** | Training jobs (my fraud model) | $ (pay per job) | Auto-scales to zero |
| **Inference Cluster** | Real-time predictions (AKS) | $$$ (always on) | 24/7 |
| **Serverless Compute** | One-off jobs | $ (pay per use) | Fully managed |

**For fraud model training:** I used **Compute Cluster**
- Size: `STANDARD_DS3_V2` (4 cores, 14 GB RAM)
- Min instances: 0 (off when not in use)
- Max instances: 1 (scales up when job submitted)
- **Cost savings:** Only pay for 20 minutes (training time), not 24/7

### 3. Jobs (Experiments)

**What:** A single training run.

**Example - My fraud detection job:**
- **Code:** `src/train_azure.py`
- **Compute:** `cpu-cluster`
- **Environment:** Python 3.8 + scikit-learn
- **Input:** `data/processed/creditcard_processed.csv`
- **Output:** `outputs/fraud_model.pkl`
- **Metrics:** accuracy, recall, F1
- **Status:** Queued → Running → Completed
- **Time:** 7 minutes
- **Cost:** $0.05

**Benefits:**
- Full logs saved
- Metrics tracked
- Model stored
- Reproducible (can rerun exact same job)

### 4. Environment

**What:** The Python packages/dependencies for your job.

**Two options:**

**Curated environments** (pre-built by Microsoft):
```python
environment="AzureML-sklearn-1.0-ubuntu20.04-py38-cpu@latest"
```
Ready to use, common ML libraries included.

**Custom environments** (your Dockerfile):
```python
from azure.ai.ml.entities import Environment

env = Environment(
    name="fraud-detection-env",
    image="mcr.microsoft.com/azureml/openmpi4.1.0-ubuntu20.04",
    conda_file="environment.yml"
)
```

**I used:** Curated environment (sklearn) - faster, no Docker needed.

### 5. Model Registry

**What:** Version-controlled storage for trained models.

**Like:** Git, but for model files (.pkl, .h5, .onnx).

**Features:**
- Versions: fraud-model:v1, v2, v3
- Tags: "production", "staging", "archived"
- Metadata: accuracy, recall, training date
- Lineage: which data/code produced this model

**Example:**
```python
# Register model
ml_client.models.create_or_update(
    name="fraud-detection",
    version="1",
    path="outputs/fraud_model.pkl",
    tags={"accuracy": "0.992", "environment": "prod"}
)
```

**Access later:**
```python
model = ml_client.models.get("fraud-detection", version="1")
# Download for deployment
```

### 6. Endpoints

**What:** A live URL where you send data and get predictions.

**Two types:**

**Real-time endpoints:**
- REST API
- Low latency (<100ms)
- For: Transaction fraud detection (immediate decision)
- Example: `POST https://fraud-api.azure.com/score`

**Batch endpoints:**
- Process large batches
- Higher latency (minutes to hours)
- For: Monthly report generation
- Example: Score 1 million transactions overnight

**My project:** Real-time endpoint would be next step (after job training).

---

## Azure ML Workflow - End to End

**My fraud detection project on Azure:**

### Step 1: Setup Workspace
```python
from azure.ai.ml import MLClient
ml_client = MLClient(credentials, subscription, resource_group, workspace)
```

### Step 2: Create Compute
```python
from azure.ai.ml.entities import AmlCompute

compute = AmlCompute(
    name="cpu-cluster",
    size="STANDARD_DS3_V2",
    min_instances=0,
    max_instances=1
)
ml_client.compute.begin_create_or_update(compute)
```

### Step 3: Submit Training Job
```python
from azure.ai.ml import command

job = command(
    code="./",
    command="python src/train_azure.py",
    environment="AzureML-sklearn-1.0-ubuntu20.04-py38-cpu@latest",
    compute="cpu-cluster"
)

ml_client.jobs.create_or_update(job)
```

### Step 4: Monitor in Portal
- Open Azure ML Studio (web UI)
- See job status: Queued → Running → Completed
- View logs live
- Check metrics (accuracy, recall)

### Step 5: Register Model
```python
# Trained model in outputs/ folder
ml_client.models.create_or_update(
    name="fraud-detection",
    path="azureml://jobs/<job_id>/outputs/fraud_model.pkl"
)
```

### Step 6: Deploy (Next Step)
```python
# Deploy to real-time endpoint
endpoint = ManagedOnlineEndpoint(name="fraud-api")
deployment = ManagedOnlineDeployment(
    name="blue",
    endpoint_name="fraud-api",
    model=model,
    instance_type="Standard_DS2_v2"
)
ml_client.online_deployments.begin_create_or_update(deployment)
```

**Result:** Live API at `https://fraud-api.<region>.inference.ml.azure.com/score`

---

## Advantages of Azure ML

| Aspect | Local Development | Azure ML |
|--------|-------------------|----------|
| **Compute power** | Limited to laptop (4 cores, 16GB RAM) | Scalable (up to 128 cores, TB RAM, GPUs) |
| **Cost** | Free (your laptop) | Pay-per-use ($0.50 per hour for training, auto-shutoff) |
| **Team collaboration** | Everyone on own machine, hard to share | Central platform, everyone sees all experiments |
| **Experiment tracking** | MLflow on local disk | Built-in tracking, searchable, filterable |
| **Model registry** | Manual file management | Versioned, tagged, lineage tracked |
| **Deployment** | Manual server setup, Flask app | One-click, auto-scaling, monitoring included |
| **Reproducibility** | "Works on my machine" | Consistent environment, job history |
| **Audit trail** | Manual logs | Complete history: who trained, when, what data |
| **Monitoring** | Custom code | Built-in drift detection, performance tracking |

---

## Azure ML vs Alternatives

| Platform | Pros | Cons | Best For |
|----------|------|------|----------|
| **Azure ML** | Integrated with Azure services, enterprise features | Learning curve, Azure-locked | Companies already on Azure (like Spectre AI) |
| **AWS SageMaker** | Market leader, most features | Complex pricing, AWS-locked | Companies on AWS |
| **Google Vertex AI** | TensorFlow integration, AutoML | GCP-locked | Companies on GCP, TensorFlow users |
| **Databricks** | Spark integration, data + ML | Expensive, complex for small projects | Big data + ML together |
| **Local (MLflow)** | Free, full control, no vendor lock-in | Limited compute, no auto-scaling, manual deployment | Small projects, prototypes, learning |

---

## Azure ML in Production (Banking)

### Real-world scenario: Spectre AI

**Architecture:**
```
On-prem EDW → Azure Synapse → ADLS (data lake)
                                  ↓
                            Azure ML Workspace
                            - Compute clusters
                            - Training pipelines
                            - Model registry
                                  ↓
                            Azure ML Endpoint (REST API)
                                  ↓
                            Fraud scoring
                                  ↓
                            AUSTRAC reports
```

**Benefits for banking:**

1. **Compliance:**
   - Complete audit trail (every model version, data used, who approved)
   - Immutable history (can't delete/modify past runs)
   - Access control (RBAC - data scientists can train, only ops can deploy)

2. **Security:**
   - Data never leaves Azure (GDPR, data residency)
   - Managed Identity (no hardcoded passwords)
   - Key Vault integration (secrets encrypted)
   - Network isolation (models train in private VNet)

3. **Governance:**
   - Model approval workflow (dev → staging → prod gates)
   - Rollback capability (previous model version deployed in seconds)
   - A/B testing (10% traffic to new model, 90% to old)

4. **Monitoring:**
   - Built-in drift detection
   - Performance metrics (latency, throughput)
   - Alerts (if API errors spike, auto-notify team)

---

## Cost Management

**Azure ML costs:**

| Component | Cost | Optimization |
|-----------|------|--------------|
| **Workspace** | Free | - |
| **Storage** | $0.02/GB/month | Delete old experiment outputs |
| **Compute cluster** | $0.096/hour (DS3_v2) | Auto-scale to zero when idle |
| **Compute instance** | $0.46/hour (DS3_v2) | Stop when not developing |
| **Inference endpoint** | $0.30/hour | Use batch for non-real-time |
| **Data transfer** | Free (same region) | Keep data + compute in same region |

**My fraud model training:**
- Compute: 7 minutes = $0.011
- Storage: 100 MB = $0.002/month
- **Total: <$0.05 per training run**

**Free tier:**
- $200 credit (good for 100+ training runs)
- After credit: Pay-as-you-go

---

## Common Mistakes

### 1. Leaving compute running
```python
# Bad: Compute instance stays on 24/7 ($330/month)

# Good: Auto-shutoff after 30 min idle
compute_instance = ComputeInstance(
    name="dev-instance",
    idle_time_before_shutdown_minutes=30
)
```

### 2. Not using environments
```python
# Bad: Install packages in command
command="pip install pandas numpy && python train.py"

# Good: Use curated environment or custom Dockerfile
environment="AzureML-sklearn-1.0-ubuntu20.04-py38-cpu@latest"
```

### 3. Ignoring logs
```
Job failed → Check "Outputs + logs" tab → user_logs/std_log.txt
```

### 4. Large data in code upload
```python
# Bad: Upload entire folder (including 1GB dataset)
code="./"

# Good: Exclude data with .amlignore
# .amlignore contains: data/, models/, venv/
```

---

## Interview Questions - Azure ML

**Q1: What is the difference between a compute instance and a compute cluster in Azure ML?**

**A:** Compute instance is a single VM that stays on (for development/Jupyter notebooks). Compute cluster is a group of machines that auto-scale based on job queue (for training). Instance = you pay 24/7 even if idle. Cluster = pay only when jobs run, then scales to zero. Use instances for interactive work, clusters for batch training jobs.

**Q2: How does Azure ML handle model versioning?**

**A:** Azure ML Model Registry stores models with version numbers (fraud-model:v1, v2, etc.). Each version is immutable (can't change once registered). Versions can have tags like "production", "staging", "archived" to indicate deployment status. Registry also stores metadata (accuracy, training date, data used) and lineage (which job/code produced this model). You can roll back by deploying an older version.

**Q3: What is the difference between a real-time endpoint and a batch endpoint?**

**A:** Real-time endpoint provides immediate predictions via REST API (latency <100ms), ideal for interactive use cases like fraud detection where decision is needed instantly. Batch endpoint processes large datasets asynchronously (latency minutes to hours), ideal for offline scoring like monthly reports. Real-time = higher cost (always-on), batch = lower cost (pay per job).

**Q4: How do you ensure reproducibility in Azure ML?**

**A:** Reproducibility requires: (1) Versioned code (Git commit hash logged with job) (2) Versioned data (use DVC or Azure ML datasets with version tags) (3) Versioned environment (Dockerfile or conda.yml) (4) Fixed random seeds in code (5) Azure ML automatically logs all this in job history. To reproduce: checkout same code commit, use same data version, same environment, same hyperparameters → get identical model.

---

# Summary & Comparison

## Tool Selection Guide

| Need | Tool | Why |
|------|------|-----|
| "My model needs to run on any machine without setup" | **Docker** | Packages everything in a container |
| "I need to know when my model is performing poorly" | **Evidently** | Detects data/concept drift |
| "I need to version my datasets like I version code" | **DVC** | Git for large files |
| "I need to train on powerful machines without buying them" | **Azure ML** | Cloud compute on-demand |
| "My company uses AWS instead of Azure" | **AWS SageMaker** | Same concepts, different platform |
| "I want to experiment locally before going to cloud" | **MLflow + Docker** | Free, learn on laptop |

---

## The Production Stack

**Real MLOps pipeline** (like Spectre AI):

```
Data: DVC (version control)
  ↓
Training: Azure ML (cloud compute + tracking)
  ↓
Model: Azure ML Registry (versioning)
  ↓
Deployment: Docker + Kubernetes (containers)
  ↓
Monitoring: Evidently + Azure Monitor (drift + performance)
  ↓
Feedback: Retrain trigger → back to Training
```

---

## What I Learned

1. **Docker** solved portability - one command to run anywhere
2. **Evidently** solved monitoring - know when to retrain before accuracy tanks
3. **DVC** solved reproducibility - audit-ready data lineage
4. **Azure ML** solved scale - train in minutes vs hours

Together, these tools transformed my laptop project into a **production-grade ML system** that can handle enterprise requirements: compliance, scale, collaboration, and monitoring.

---

## Next Steps

After learning these tools, the natural next steps are:

1. **Orchestration** (Airflow/Kubeflow): Automate the pipeline (data → train → deploy)
2. **Advanced deployment** (Kubernetes/Seldon/BentoML): Scale to millions of requests
3. **Feature stores** (Feast): Centralize features across multiple models
4. **CI/CD for ML** (GitHub Actions + ML): Automated testing and deployment

But the 4 tools in Phase 5 are the **foundation** - everything else builds on these.
