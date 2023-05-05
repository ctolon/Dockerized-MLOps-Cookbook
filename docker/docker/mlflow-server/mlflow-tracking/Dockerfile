FROM python:3.8.5
RUN apt-get update

# BASIC PACKAGES
RUN python -m pip install --upgrade setuptools pip
RUN apt-get install git

# python c deps
#RUN apk add --no-cache alpine-sdk postgresql-dev postgresql-client openssh openssl-dev 

# PERFORMANCE PROVIDERS (BEST PRACTICE)
#RUN apt-get install libyaml-cpp-dev libyaml-dev

RUN python -m pip install mlflow==2.3.0
RUN python -m pip install boto3
RUN python -m pip install protobuf==3.20.*
RUN python -m pip install psycopg2
RUN python -m pip install prometheus-flask-exporter
RUN python -m pip install dvc[all]

COPY run_server.sh .
RUN chmod +x ./run_server.sh

ARG MLFLOW_TRACKING_URI
ENV MLFLOW_TRACKING_URI=$MLFLOW_TRACKING_URI

ARG MLFLOW_TRACKING_INSECURE_TLS 
ENV MLFLOW_TRACKING_INSECURE_TLS=$MLFLOW_TRACKING_INSECURE_TLS

RUN mkdir -p /mnt/mlruns/
RUN chmod 777 -R /mnt/mlruns/

EXPOSE 5000/tcp


