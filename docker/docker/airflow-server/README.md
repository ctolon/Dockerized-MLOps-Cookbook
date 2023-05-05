# Airflow Server Docker Konfigürasyonu


- [Airflow Server Docker Konfigürasyonu](#airflow-server-docker-konfigürasyonu)
  - [Docker Compose Ön Tanımlı Ayar Konfigürasyonları](#docker-compose-ön-tanımlı-ayar-konfigürasyonları)
    - [1. x-airflow-common](#1-x-airflow-common)
      - [Image Ayarları](#image-ayarları)
      - [Airflow Environment Genel Değişkenleri](#airflow-environment-genel-değişkenleri)
      - [Common Volumes](#common-volumes)
      - [User settings](#user-settings)
  - [Docker Compose Airflow Service Konfigürasyonları](#docker-compose-airflow-service-konfigürasyonları)
    - [1. airflow-webserver](#1-airflow-webserver)
      - [Genel Ayarlar](#genel-ayarlar)
    - [2. airlfow-scheduler](#2-airlfow-scheduler)
      - [Genel Ayarlar](#genel-ayarlar-1)
    - [3. airflow-init](#3-airflow-init)
    - [4. airflow-cli](#4-airflow-cli)
    - [5. docker-proxy](#5-docker-proxy)
    - [6. postgres](#6-postgres)
  - [Dockerfile Konfigürasyonu](#dockerfile-konfigürasyonu)

## Docker Compose Ön Tanımlı Ayar Konfigürasyonları 

### 1. x-airflow-common

Şablon

```bash
x-airflow-common:
  &airflow-common
  #image: ${AIRFLOW_IMAGE_NAME:-apache/airflow:2.5.1}
  build:
    context: ./airflow
    dockerfile: Dockerfile
  environment:
    &airflow-common-env
    AIRFLOW__CORE__EXECUTOR: LocalExecutor
    AIRFLOW__DATABASE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow@postgres/airflow
    AIRFLOW__CORE__SQL_ALCHEMY_CONN: postgresql+psycopg2://airflow:airflow@postgres/airflow
    AIRFLOW__CORE__FERNET_KEY: ''
    AIRFLOW__CORE__DAGS_ARE_PAUSED_AT_CREATION: 'true'
    AIRFLOW__CORE__LOAD_EXAMPLES: 'false'
    AIRFLOW__API__AUTH_BACKENDS: 'airflow.api.auth.backend.basic_auth,airflow.api.auth.backend.session'
    #_PIP_ADDITIONAL_REQUIREMENTS: ${_PIP_ADDITIONAL_REQUIREMENTS:- apache-airflow-providers-docker}
    AIRFLOW__CORE__ENABLE_XCOM_PICKLING: 'true'
  volumes:
    - ${AIRFLOW_PROJ_DIR:-.}/dags:/opt/airflow/dags
    - ${AIRFLOW_PROJ_DIR:-.}/logs:/opt/airflow/logs
    - ${AIRFLOW_PROJ_DIR:-.}/plugins:/opt/airflow/plugins
    #- "/var/run/docker.sock:/var/run/docker.sock"
  user: "${AIRFLOW_UID:-50000}:0"
  depends_on:
    &airflow-common-depends-on
    postgres:
      condition: service_healthy
```


#### Image Ayarları

Airflow için docker service hazırlarken 2 yaklaşım kullanılabilir:

1. Doğrudan Airflow Image'ını kullanıp build etmek
2. Airflow Image'ını Customize ederek build etmek

Bu yaklaşımlardan önerilen olan yaklaşım 2. yaklaşımdır. Çünkü ilk yaklaşımda bir özelleştirme yapamıyoruz. Yaptığımız özelleştirmeler yeniliklere açık olmamaktadır. Örneğin _PIP_ADDITIONAL_REQUIREMENTS değişkeni ile airflow'a eklemek istediğimiz yeni pip paketlerini yükleyebiliyoruz fakat bu şekilde eklediğimizde docker compose ile docker'ı her ayağa kaldırdığımızda bu paketleri tekrar tekrar yükler ve bu istediğimiz bir şey değildir. Dolayısıyla image parametresi devre dışı bırakılmalı, ayrı bir `Dockerfile` hazırlanması daha sağlıklıdır. Bunun için bir `Dockerfile` hazırlanmıştır ve `airflow` klasörünün altındadır.


#### Airflow Environment Genel Değişkenleri

* **AIRFLOW__CORE__EXECUTOR**: Airflow'un hangi Executor (yürütücü) ile çalıştırılacağını belirtir. Yürütücüler yerelde ve uzakta çalışanlar olmak üzere ikiye ayrılır. Varsayılan yürütücü `Sequential Executor` olarak gelir. Kullanılabilecek yürütücüler listesi:
  * Yerel Yürütücüler:
    * Local Executor
    * Sequential Executor
  * Uzaktan Yürütücüer:
    * Celery Executor
    * CeleryKubernetes Executor
    * Dask Executor
    * Kubernetes Executor
    * LocalKubernetes Executor

TODO: Yürütücüleri ayrıntılı açıkla

* **AIRFLOW__DATABASE__SQL__ALCHEMY__CONN**: Airflow'un meta datasının hangi veritabanında tutulacağına göre SQL connection ayarları.
* **AIRFLOW__CORE__SQL_ALCHEMY_CONN**: Bu ayarda sql connection ayarı olmasına karşın `airflow version 2.3`'ten sonra deprecated olmuştur. Versiyon uyumluluğu açısından `AIRFLOW__DATABASE__SQL__ALCHEMY__CONN` ile aynı ayarların sağlanması önerilir.
* **AIRFLOW__CORE__FERNET_KEY**
* **AIRFLOW__CORE__DAGS_ARE_PAUSED_AT_CREATION**:
* **AIRFLOW__CORE__LOAD_EXAMPLES**: Airflow'un default olarak sağladığı DAG örneklerinin yüklenip yüklenmeyeceğini belirten parametredir. `'true'` ya da `'false'` değerini alır.
* **AIRFLOW__API__AUTH_BACKENDS**:
* **_PIP_ADDITIONAL_REQUIREMENTS**: Airflow'a ekstra pip paketlerini yüklemeye yaran parametre. Daha öncede açıklandığı üzere esnek olmadığı için kullanılması tavsiye edilmez.
* **AIRFLOW__CORE__ENABLE_XCOM_PICKLING**: `'true'` ya da `'false'` değerini alır. Docker operatörünün düzgün çalışması için her zaman `true` olarak yapılandırılmalıdır! (_run_image of the DockerOperator returns now a python string, not a byte string" Ref: https://github.com/apache/airflow/issues/13487)

#### Common Volumes

- ${AIRFLOW_PROJ_DIR:-.}/dags:/opt/airflow/dags
    - ${AIRFLOW_PROJ_DIR:-.}/logs:/opt/airflow/logs
    - ${AIRFLOW_PROJ_DIR:-.}/plugins:/opt/airflow/plugins
    #- "/var/run/docker.sock:/var/run/docker.sock" # We will pass the Docker Deamon as a volume to allow the webserver containers start docker images. Ref: https://stackoverflow.com/q/51342810/7024760


#### User settings

"${AIRFLOW_UID:-50000}:0"

## Docker Compose Airflow Service Konfigürasyonları

### 1. airflow-webserver

Şablon:

```bash
airflow-webserver:
  <<: *airflow-common
  command: webserver -p 8086
  ports:
    - 8086:8086
  healthcheck:
    test: ["CMD", "curl", "--fail", "http://localhost:8086/health"]
    interval: 10s
    timeout: 10s
    retries: 5
  restart: always
  depends_on:
    <<: *airflow-common-depends-on
    airflow-init:
      condition: service_completed_successfully
```
#### Genel Ayarlar

* **`<<: *airflow-common**`**: airflow-commonda uygulanan her ayarın airflow-webserver için de kullanılmasını sağlar (bir nevi kalıtım diyebiliriz).

* **`command`**: airflow-webserver'ının hangi komutla ayağa kaldırılacağını belirtir. webserver -p `<port-numarası>` olmak üzere çalıştırılır. komutta verilen port numarası ile `ports` ayarındaki port numaraları aynı olmalıdır.

* **`ports`**: airflow-webserver port yönlendirmesi

* **`healtcheck`**: airflow-webserver sağlık testleri.

* **`restart`**:

* **`depends_on`**:

  * **`airflow-init`**:

### 2. airlfow-scheduler

Şablon:

```bash
airflow-scheduler:
  <<: *airflow-common
  command: scheduler
  healthcheck:
    test: ["CMD-SHELL", 'airflow jobs check --job-type SchedulerJob --hostname "$${HOSTNAME}"']
    interval: 10s
    timeout: 10s
    retries: 5
  restart: always
  depends_on:
    <<: *airflow-common-depends-on
    airflow-init:
      condition: service_completed_successfully
```

#### Genel Ayarlar

* **`<<: *airflow-common**`**: airflow-commonda uygulanan her ayarın airflow-webserver için de kullanılmasını sağlar (bir nevi kalıtım diyebiliriz).

* **`command`**: airflow-webserver'ının hangi komutla ayağa kaldırılacağını belirtir. webserver -p `<port-numarası>` olmak üzere çalıştırılır. komutta verilen port numarası ile `ports` ayarındaki port numaraları aynı olmalıdır.

* **`ports`**: airflow-webserver port yönlendirmesi

* **`healtcheck`**: airflow-webserver sağlık testleri.

* **`restart`**:

* **`depends_on`**:

  * **`airflow-init`**:


### 3. airflow-init

Şablon:

```bash
airflow-init:
  <<: *airflow-common
  entrypoint: /bin/bash
  # yamllint disable rule:line-length
  command:
    - -c
    - |
      function ver() {
        printf "%04d%04d%04d%04d" $${1//./ }
      }
      airflow_version=$$(AIRFLOW__LOGGING__LOGGING_LEVEL=INFO && gosu airflow airflow version)
      airflow_version_comparable=$$(ver $${airflow_version})
      min_airflow_version=2.2.0
      min_airflow_version_comparable=$$(ver $${min_airflow_version})
      if (( airflow_version_comparable < min_airflow_version_comparable )); then
        echo
        echo -e "\033[1;31mERROR!!!: Too old Airflow version $${airflow_version}!\e[0m"
        echo "The minimum Airflow version supported: $${min_airflow_version}. Only use this or higher!"
        echo
        exit 1
      fi
      if [[ -z "${AIRFLOW_UID}" ]]; then
        echo
        echo -e "\033[1;33mWARNING!!!: AIRFLOW_UID not set!\e[0m"
        echo "If you are on Linux, you SHOULD follow the instructions below to set "
        echo "AIRFLOW_UID environment variable, otherwise files will be owned by root."
        echo "For other operating systems you can get rid of the warning with manually created .env file:"
        echo "    See: https://airflow.apache.org/docs/apache-airflow/stable/howto/docker-compose/index.html#setting-the-right-airflow-user"
        echo
      fi
      one_meg=1048576
      mem_available=$$(($$(getconf _PHYS_PAGES) * $$(getconf PAGE_SIZE) / one_meg))
      cpus_available=$$(grep -cE 'cpu[0-9]+' /proc/stat)
      disk_available=$$(df / | tail -1 | awk '{print $$4}')
      warning_resources="false"
      if (( mem_available < 4000 )) ; then
        echo
        echo -e "\033[1;33mWARNING!!!: Not enough memory available for Docker.\e[0m"
        echo "At least 4GB of memory required. You have $$(numfmt --to iec $$((mem_available * one_meg)))"
        echo
        warning_resources="true"
      fi
      if (( cpus_available < 2 )); then
        echo
        echo -e "\033[1;33mWARNING!!!: Not enough CPUS available for Docker.\e[0m"
        echo "At least 2 CPUs recommended. You have $${cpus_available}"
        echo
        warning_resources="true"
      fi
      if (( disk_available < one_meg * 10 )); then
        echo
        echo -e "\033[1;33mWARNING!!!: Not enough Disk space available for Docker.\e[0m"
        echo "At least 10 GBs recommended. You have $$(numfmt --to iec $$((disk_available * 1024 )))"
        echo
        warning_resources="true"
      fi
      if [[ $${warning_resources} == "true" ]]; then
        echo
        echo -e "\033[1;33mWARNING!!!: You have not enough resources to run Airflow (see above)!\e[0m"
        echo "Please follow the instructions to increase amount of resources available:"
        echo "   https://airflow.apache.org/docs/apache-airflow/stable/howto/docker-compose/index.html#before-you-begin"
        echo
      fi
      mkdir -p /sources/logs /sources/dags /sources/plugins
      chown -R "${AIRFLOW_UID}:0" /sources/{logs,dags,plugins}
      exec /entrypoint airflow version
  # yamllint enable rule:line-length
  environment:
    <<: *airflow-common-env
    _AIRFLOW_DB_UPGRADE: 'true'
    _AIRFLOW_WWW_USER_CREATE: 'true'
    _AIRFLOW_WWW_USER_USERNAME: ${_AIRFLOW_WWW_USER_USERNAME:-airflow}
    _AIRFLOW_WWW_USER_PASSWORD: ${_AIRFLOW_WWW_USER_PASSWORD:-airflow}
    _PIP_ADDITIONAL_REQUIREMENTS: ''
  user: "0:0"
  volumes:
    - ${AIRFLOW_PROJ_DIR:-.}:/sources
```

### 4. airflow-cli

Şablon:

```bash
airflow-cli:
  <<: *airflow-common
  profiles:
    - debug
  environment:
    <<: *airflow-common-env
    CONNECTION_CHECK_MAX_COUNT: "0"
  # Workaround for entrypoint issue. See: https://github.com/apache/airflow/issues/16252
  command:
    - bash
    - -c
    - airflow
```


### 5. docker-proxy

Şablon:

```bash
# Reference: https://onedevblog.com/how-to-fix-a-permission-denied-when-using-dockeroperator-in-airflow/
docker-proxy:
  image: bobrik/socat
  command: "TCP4-LISTEN:2375,fork,reuseaddr UNIX-CONNECT:/var/run/docker.sock"
  ports:
    - "2376:2375"
  volumes:
    - /var/run/docker.sock:/var/run/docker.sock
  restart: always
```

### 6. postgres

Şablon:


```bash
postgres:
  image: postgres:13
  environment:
    POSTGRES_USER: airflow
    POSTGRES_PASSWORD: airflow
    POSTGRES_DB: airflow
  volumes:
    - postgres-db-volume:/var/lib/postgresql/data
  healthcheck:
    test: ["CMD", "pg_isready", "-U", "airflow"]
    interval: 5s
    retries: 5
  restart: always
```

## Dockerfile Konfigürasyonu

Şablon:

```bash

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

```