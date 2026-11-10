# FIFA Player Risk-Proxy MLOps (Labs 1-13)

This project adapts the supplied MLOps labs to a FIFA player CSV you are licensed to use, placed at `data/raw/fifa_players.csv`. The dataset is intentionally not included in this public repository; the supplied file appears to contain third-party FIFA/SoFIFA data. GitHub Actions uses fictional generated demo rows instead. The project demonstrates data validation and preparation, a reproducible classifier, MLflow tracking and registry, a quality gate, inference, monitoring, a FastAPI service, Docker packaging, tests, and GitHub Actions.

## Important target limitation

The FIFA snapshot contains 391 players and 110 columns, but the club, contract-expiry, and club-joining fields are entirely empty. It contains no observed transfers, future outcomes, or churn labels. **Actual player churn cannot be trained or validated from this dataset.**

To let the labs run end to end, the demo creates a clearly identified synthetic target, `churn_proxy`: `1` when `age >= 32`, otherwise `0`. The age is also a model input, so this is deliberately a demonstration of the MLOps workflow—not a meaningful churn model, transfer forecast, or measure of career risk. Replace this proxy with real longitudinal transfer/contract labels before using the system for player decisions.

## Run the labs

Use Python 3.12 and a virtual environment. For real-data runs, put your locally available, licensed `fifa_players.csv` in `data/raw/` first:

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
python pipelines\run_all_labs.py
python -m pytest -q
```

The pipeline validates the CSV, creates a stratified train/test split, trains and evaluates the model, tracks and registers it in MLflow, enforces a demo recall gate, sets a champion alias, runs inference, and writes a monitoring report. Generated data, models, reports, and MLflow state are excluded from Git.

To run the same workflow without supplying a dataset, generate fictional demo rows:

```powershell
python -m fifa_churn.create_demo_data --output data\ci\fifa_demo.csv
$env:FIFA_DATA_PATH = (Resolve-Path data\ci\fifa_demo.csv).Path
python pipelines\run_all_labs.py
python -m pytest -q
```

The demo rows are randomly generated fixtures, not real FIFA players or observed churn labels. Do not commit third-party datasets unless you have permission to redistribute them.

Individual lab runners for the original baseline, experiment tracking, and production-validation stages remain available:

```powershell
python pipelines\run_lab_3_baseline.py
python pipelines\run_lab_4_experiment.py
python pipelines\run_lab_5_production.py
```

## MLflow

Runs and the registered model use the local `mlflow.db` SQLite store:

```powershell
mlflow ui --backend-store-uri sqlite:///mlflow.db
```

The experiment is `FIFA_Player_Churn_Proxy`; its registered model is `FIFA_Player_Churn_Proxy_Model`. Registry metrics describe the synthetic proxy only.

## Serve predictions

After running the pipeline:

```powershell
$env:PYTHONPATH = "src"
uvicorn fifa_churn.api:app --app-dir src --reload
```

Open `http://127.0.0.1:8000/docs`. The service exposes `GET /health`, `POST /predict`, and `POST /predict/batch`. Example:

```json
{
  "age": 30,
  "overall": 75,
  "potential": 80,
  "height_cm": 175,
  "weight_kg": 70,
  "weak_foot": 3,
  "skill_moves": 3,
  "international_reputation": 1,
  "player_positions": "CM",
  "preferred_foot": "Right"
}
```

The response includes the proxy class, a model probability, and an explicit synthetic-target warning. Missing model files return HTTP 503 rather than a default prediction.

## Docker

Train locally before building; the Docker image includes the resulting model artifact:

```powershell
python pipelines\run_all_labs.py
docker build -t fifa-player-risk-proxy .
docker run --rm -p 8000:8000 fifa-player-risk-proxy
```

The API is then available at `http://localhost:8000/docs`. The image runs as a non-root user and has a `/health` container check.

The published public demo image uses the fictional CI fixture:

```powershell
docker pull pavankuarapati123/fifa-player-mlops:latest
docker run --rm -p 8000:8000 pavankuarapati123/fifa-player-mlops:latest
```

Its predictions are only for the synthetic age-based proxy and are not trained or validated as real player churn.

## Lab 13 monitoring and automation

`src/monitor.py` compares a current FIFA CSV with the reference snapshot using population stability index and high-risk prediction-rate shift. By default, it compares the snapshot to itself; pass a new compatible CSV batch to measure drift:

```powershell
python src\monitor.py --current-data data\raw\new_players.csv
```

The report is written to `reports/monitoring_report.json`. GitHub Actions shows monitoring as its own **“Lab 12 - Feature and prediction monitoring”** step, followed by tests and a Docker build, on pushes, pull requests, manual dispatch, and a weekly schedule. It uploads the monitoring report; it does not publish an image to a container registry. Because there are no observed labels, the workflow does not claim concept-drift or real-churn performance monitoring and does not automatically retrain on an unlabelled batch.
