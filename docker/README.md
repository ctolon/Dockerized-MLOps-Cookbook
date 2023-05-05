# Docker Compose Guide ➡️ MLflow ⟳ + Postgres ⛁  And Airflow ♲ + Postgres ⛁ + Data Version Control (DVC) 🗂

### Tech Stacks

* [PostgreSQL ⛁](https://www.postgresql.org/)
* [MLflow ⟳](https://mlflow.org/docs/latest/index.html)
* [Airflow ♲](https://airflow.apache.org/)
* [Jenkins 🤖](https://www.jenkins.io/)
* [MinIO 🦩](https://min.io/)
* [Data Version Control](https://dvc.org/)

This repo is combines several reference repos
* Postgres and Jenkins are uniquely prepared.
* MLFlow from https://github.com/crmne/mlflow-tracking
* Airflow from (It's a Lite version with local executor, for full installation guide see ref. article) https://airflow.apache.org/docs/apache-airflow/stable/docker-compose.yaml
* Also we have alternate Airflow (backup folder) from https://github.com/RBuractaon/docker-spark-jupyter-mlflow-hive-metastore
* Reference Article for Airflow https://pub.towardsai.net/you-can-install-airflow-with-docker-in-minutes-66a3de44374f
* MinIO Uniquely Prepared.
* DVC is Used as Pip package and works on minIO bucket.

### Overview

**Files**

  * **MLFlow**
    * [docker-compose.yml - MLFlow](docker/mlflow-server/docker-compose.yml) - MLflow Server Docker compose file.
      * [Dockerfile - MLFlow Tracking Server](docker/mlflow-server/mlflow-tracking/Dockerfile) - Mlflow tracking server Dockerfile.
      * [Dockerfile - MLFLow Artifact Server](docker/mlflow-artifact/mlflow-artifact/Dockerfile) - Mlflow artifact server Dockerfile.
      * [run_server.sh - MLFlow Tracking Server](docker/mlflow-server/mlflow-tracking/run_server.sh) - MLflow manage run tracking server script.
      * [run_server.sh - MLFlow Artifact Server](docker/mlflow-server/mlflow-artifact/run_server.sh) - MLflow manage run artifact server script.
      * [create_db.sql](docker/mlflow-server/postgres/init/create_db.sql) SQL query for create initial db named as "mlflow" (as postgres for backend storage).
  * **Airflow** 
      * [docker-compose.yml - Airflow](docker/airflow-server/docker-compose.yml) - Airflow Server Docker compose file.
      * Other files are experimental for alternative docker compose from big data europe. Don't mind them.
      * [Dockerfile - Airflow Server](docker/airflow-server/airflow/Dockerfile) - Airflow Customized Dockerfile.

  * **Python-env-3.6.8**
    * [docker-compose.yml - Dev env.](docker/python-env-3.6.8/docker-compose.yml) - Python 3.6.8 Development Env. docker compose file.
      * [Dockerfile - Dev env.](docker/python-env-3.6.8/dev/Dockerfile) - Python 3.6.8 Development Env. docker file..
      * [mlflow.yml - Conda dev env.](docker/python-env-3.6.8/dev/mlflow.yml) - Internal Conda Env. file for MLFlow v2.1.1.
      * [nmnlp.yml - Conda dev env.](docker/python-env-3.6.8/dev/nmnlp.yml) - Premaded Internal Conda Env. file for NMNLP.
  * All Global Environment variables defined in `.env` files


**Access MLFlow Artifacts (Deprecated, now we use minIO instead of this)**

In order to access ML artifacts between MLFlow container runs, we use Docker's volume feature to access data to share MLArtifacts directories:

```bash
volumes:
  shared-volume:
```

## Build and Launch Up Docker

### 1. Build the containers
```
docker-compose -f docker-compose.yml build
```

### 2. Launch the containers

if we are working over ssh and we do not want the terminal to be blocked when we start docker:

```
docker-compose -f docker-compose.yml up -d
```

If we are working over ssh and it is not important that the terminal is blocked when we enter docker, we can also install docker compose as an alternative to see the log messages:

```
docker-compose -f docker-compose.yml up
```

## Extra Docker commands

### Partially Compile per image

```
docker compose build <server_name> (ex. spark_hive)
```

### Connect a Docker container with interactive shell

```
docker exec -it <container_name> /bin/bash (ex. Mlflow-server)
```

### Delete docker container hard delete

```
docker-compose down --volumes --remove-orphans
```

### MLFlow + MinIO Server Config

For this figure, now we use minIO as artifact store instead of MLFlow artifact sever.

![plot](../docker/images/MLFlow-MinIO-Setup.jpg)

### MLFlow + MinIO + Airflow + New-Mind Server Config and MLOps Pipeline

For this figure, now we use minIO as artifact store instead of MLFlow artifact sever.

![plot](../docker/images/Level_1_MLOps_Arch.jpg)

### .env Config File Template examples

Template for MLFlow:

```bash
# Postgresql
# POSTGRES_USER=user # user can be defined, root user is postgres
POSTGRES_PASSWORD=postgreadmin
POSTGRESQL_VERSION=14.6
POSTGRESQL_CONNECTOR_VERSION=42.5.1
POSTGRES_HOST_AUTH_METHOD=md5
POSTGRES_DATABASE=mlflow
POSTGRES_PATH=/var/lib/postgresql/data

# MLFlow Tracking Server 
MLFLOW_VERSION=1.30.0
MLFLOW_TRACKING_INSECURE_TLS=true
MLFLOW_ARTIFACT_DIR=/mlartifacts
MLFLOW_TRACKING_URI=postgresql+psycopg2://postgres:postgreadmin@postgresql/mlflow?sslmode=disable
#MLFLOW_TRACKING_USERNAME=admin
#MLFLOW_TRACKING_PASSWORD=admin
```

Template for Airflow:

```bash
### Airflow
AIRFLOW_UID=1000
AIRFLOW_GID=0

### Postgres
POSTGRES_USER=airflow # user can be defined, root user is postgres
POSTGRES_PASSWORD=airflow
POSTGRES_DB=airflow
POSTGRESQL_VERSION=14.6
POSTGRESQL_CONNECTOR_VERSION=42.5.1
POSTGRES_HOST_AUTH_METHOD=md5
POSTGRES_DATABASE=mlflow
POSTGRES_PATH=/var/lib/postgresql/data
```

### Usernames and Passwords (Default)

#### MLflow Database

**Postgresql**: postgres - postgreadmin

Note: postgres is root user.

#### Airflow Database

**Postgresql**: airflow - airflow

Note: postgres is root user.

#### Airflow Webserver

**Admin**: airflow - airflow

#### MinIO WebServer

**Admin**: minioadmin - minioadmin

### Data Version Control (DVC) setup for repos

Here are the steps:

1. Clone the project repository:

```bash
git clone <repo>
```

2. Initialize DVC on repository: 

```bash
dvc init
```

3. Disable send analytics for privacy:

```bash
dvc config core.analytics false
```

4. Configure minio bucket for storage (for ex. mlflow is bucket in minio):

```bash
dvc remote add -d minio s3://mlflow 
```

5. Configure minio endpoint (for ex. if port exposed on 9000):

```bash
dvc remote modify minio endpointurl http://0.0.0.0:9000/minio/
```

6. Add remote access key id for accessing minio (for ex. default minio access key id is minioadmin):

```bash
dvc remote modify minio access_key_id minioadmin
```

7. Add minio_secret_access_key for accessing minio(for ex. default minio secret access key is minioadmin)

```bash
dvc remote modify minio secret_access_key minioadmin
```

8. Lastly add, Commit and Push:

```bash
git add.
git commit -m "dvc init"
git push origin master
```

Then you can check your configs in .dvc/config file. It should be like this (for ex. minio bucket is mlflow, minio exposed on 9000 port and access creds are minioadmin):

```bash
[core]
    analytics = false
    remote = minio
['remote "minio"']
    url = s3://mlflow
    endpointurl = http://0.0.0.0:9000/minio/
    access_key_id = minioadmin
    secret_access_key = minioadmin
```

### Usage of DVC for Data Versioning

#### Add Data to DVC

example for saving train.csv file on minio bucket with dvc for data versioning:

1. Add data to DVC system on minio:

```bash
dvc add /path/to/train.csv
```

2. Add to .dvc file to gitignore (best practice):

```bash
git add /path/to/train.csv.dvc .gitignore
```

3. Do Commit changes:

```bash
git commit -m "new data added"
```

4. Give a git tag for access data easily when needed later (best practice): 

```bash
git tag -a "2022-03-05-train.csv"
```

5. Push Changes on DVC:

```bash
dvc push
```

5. Push Changes on Git:

```bash
git push
```

6. Push Git Tags:

```bash
git push --tags
``` 


#### Fetch Data From DVC

Then we can fetch data with checkout to tag when we need data:

1. Checkout to branch with git tag which contains data saved:

```bash
git checkout 2022-03-05-train.csv
```

2. Also Checkout with DVC:

```bash
dvc checkout
```

3. Pull Data:

```bash
dvc pull
```

### Important Note

When running MLflow tracking and artifact server with docker compose, we can't switch ports with entrypoints and so it's our start server commands are hard coded commands in docker compose file (same issue for airflow).

