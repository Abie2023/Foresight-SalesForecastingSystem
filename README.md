<h1 align="center"> Sales Forecast MLOps at Scale </h1>

<p align="center"><b> ▶️ Highly scalable Cloud-native Machine Learning system ◀️ </b></p>

------

# Table of contents
- [Architecture](#architecture)
- [Tools / Technologies](#tools--technologies)
- [Development environment](#development-environment)
- [How things work](#how-things-work)
  - [Setup using Docker Compose](#with-docker-compose)
  - [With Kubernetes/Helm (on GCP)](#with-kuberneteshelm-on-gcp)
  - [Cleanup steps](#cleanup-steps)
- [References / Useful resources](#references--useful-resources)

------

## Architecture
<image src="./files/sfmlops_software_diagram.png">

## Tools / Technologies
- Platform: [Docker](https://www.docker.com/)
- Cloud platform: [Google Cloud Platform](https://cloud.google.com/)
- Experiment tracking / Model registry: [MLflow](https://mlflow.org/)
- Pipeline orchestrator: [Airflow](https://airflow.apache.org/)
- Model distributed training and scaling: [Ray](https://www.ray.io/)
- Reverse proxy: [Nginx](https://www.nginx.com/) and [ingress-nginx](https://github.com/kubernetes/ingress-nginx) (for Kubernetes)
- Web Interface: [Streamlit](https://streamlit.io/)
- Machine Learning service deployment: [FastAPI](https://fastapi.tiangolo.com/), [Uvicorn](https://www.uvicorn.org/), [Gunicorn](https://gunicorn.org/)
- Databases: [PostgreSQL](https://www.postgresql.org/), [Prometheus](https://prometheus.io/)
- Database UI for Postgres: [pgAdmin](https://www.pgadmin.org/)
- Overall system monitoring & dashboard: [Grafana](https://grafana.com/)
- Distributed data streaming: [Kafka](https://kafka.apache.org/)
- Forecast modeling framework: [Prophet](https://facebook.github.io/prophet/docs/quick_start.html)
- Stream processing: [Spark Streaming](https://spark.apache.org/streaming/)
- CICD: [GitHub Actions](https://github.com/features/actions)

------

## Development environment
1. Docker (ref: Docker version 24.0.6, build ed223bc)

## How things work
### 1. System Initialization

- The **Data Producer**:
  - Loads the last 5 months of data from `train_exclude_last_10d.csv`.
  - Adjusts the dates so that the latest entry becomes **yesterday**.
  - Inserts the data into **PostgreSQL**.
  - Begins streaming **today’s data** (from `train_only_last_19d.csv`) to the Kafka topic `sale_rossman_store` every **10 seconds** in an infinite loop.

### 2. Data Processing Pipelines via Airflow
#### 🗓️ Daily DAG
- Ingests data from Kafka.
- Processes and transforms it using **Spark Streaming**.
- Stores the cleaned data in the `rossman_sales` table in **PostgreSQL**.

#### 📅 Weekly DAG
- Pulls the past 4 months of sales data from PostgreSQL.
- Trains **Prophet** models in parallel using **Ray** (~11,115 models).
- Tracks and registers models using **MLflow**.
- Forecasts the next 7 days of sales.
- Stores results in the `forecast_results` table in PostgreSQL.

### 3. 📊 Monitoring
- **Prometheus** and **Grafana** are used to monitor infrastructure and system health.

### 4. 🗃️ Data Storage
| Table Name        | Description                        |
|-------------------|------------------------------------|
| `rossman_sales`   | Cleaned sales data from Kafka      |
| `forecast_results`| Weekly forecast results            |

- Use **pgAdmin** to explore and manage PostgreSQL tables.

### 5. 🌐 Streamlit Web App
- Served via **Nginx** reverse proxy.
- Displays the latest 7-day forecast for any store/product.
- Uses **Altair** charts for visualization.
- Chart subtitles show model ID and version.

### 6. 🔁 Weekly Forecast Logic
- Forecasts are refreshed weekly by the Weekly DAG.
- Users see the same forecast throughout the week (regardless of the day accessed).
- If a forecast appears inaccurate or outdated, users can initiate retraining.

### 7. 🛠️ On-Demand Model Retraining
- Clicking **Retrain** in the UI:
  - Triggers a job submitted to the **Training Service**.
  - The service uses **Ray** to retrain the selected model.
  - Model retraining completes in **under 1 minute**.
  - The new model version is immediately available.

### 8. 📈 On-Demand Forecasting
- After retraining, users can:
  - Select custom future dates.
  - Click **Forecast** to generate updated predictions.
- These ad-hoc results are:
  - **Displayed** directly on the chart.
  - **Not stored** in the database.

------

## Setup using Docker Compose
1. *(Optional)* In case you want to build (not pulling images):
   ```
   docker-compose build
   ```
2. ```
   docker-compose -f docker-compose.yml -f docker-compose-airflow.yml up -d
   ```
3. Sometimes, it can freeze or fail the first time, especially if your machine is not that high-spec. But you can wait a second, try the last command again, and it should start up fine.

## With Kubernetes/Helm (on GCP)
Prerequisites: GKE Cluster (Standard cluster, *NOT* Autopilot), Artifact Registry, Service Usage API, gcloud cli
1. Follow this [Medium blog](https://medium.com/@gravish316/setup-ci-cd-using-github-actions-to-deploy-to-google-kubernetes-engine-ef465a482fd). Instead of using the default Service Account (as done in the blog), I recommend creating a new Service Account with Owner role for a quick and dirty run (but of course, please consult your cloud engineer if you have security concerns).
2. Download your Service Account's JSON key
3. Activate your Service Account:
   ```
   gcloud auth activate-service-account --key-file=<PATH_TO_JSON_KEY>
   ```
4. Connect local kubectl to cloud:
   ```
   gcloud container clusters get-credentials <GKE_CLUSTER_NAME> --zone <GKE_ZONE> --project <PROJECT_NAME>
   ```
5. Now `kubectl` (and `helm`) will work in the context of the GKE environment.
6. Follow the steps in [With Kubernetes/Helm (Local cluster)](#with-kuberneteshelm-local-cluster) section
7. If you face a timeout error when running helm commands for airflow or the system struggles to set up and work correctly, I recommend trying to upgrade your machine type in the cluster.

**Note:** For the machine type of node pool in the GKE cluster, from experiments, `e2-medium` (default) is not quite enough, especially for Airflow and Ray. In my case, I went for `e2-standard-8` with 1 node (explanation on why only 1 node is in [Important note on MLflow on Cloud](#important-note-on-mlflow-on-cloud) section). I also found myself the need to increase the quota for PVC in IAM too.


## Cleanup steps
```
helm uninstall sfmlops-helm -n mlops
helm uninstall kafka-release -n kafka
helm uninstall airflow -n airflow
helm uninstall kube-Prometheus-stack -n monitoring
```

-------

# References / Useful resources
- Ray sample config: https://github.com/ray-project/kuberay/tree/master/ray-operator/config/samples
- Bitnami Kafka Helm: https://github.com/bitnami/charts/tree/main/bitnami/kafka
- Airflow Helm: https://airflow.apache.org/docs/helm-chart/stable/index.html
- Airflow Helm default values.yaml: https://github.com/apache/airflow/blob/main/chart/values.yaml
- dataset: https://www.kaggle.com/datasets/pratyushakar/rossmann-store-sales
- Original Airflow's docker-compose file: https://airflow.apache.org/docs/apache-airflow/2.8.3/docker-compose.yaml
