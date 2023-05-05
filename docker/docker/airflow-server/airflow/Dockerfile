FROM apache/airflow:2.5.1
USER root

# System packages
RUN apt-get update \
  && apt-get install -y --no-install-recommends \
         vim nano git\
  && apt-get autoremove -yqq --purge \
  && apt-get clean \
  && rm -rf /var/lib/apt/lists/*

# Pip Dependencies
USER airflow
COPY requirements.txt /
RUN pip install --no-cache-dir -r /requirements.txt
