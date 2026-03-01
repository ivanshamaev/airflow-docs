# Режим выполнения Docker (Docker Execution Mode)

Ниже — туториал по запуску Docker-операторов Cosmos dbt и необходимой настройке.

## Требования

- **Docker** с демоном Docker (Docker Desktop на macOS). Инструкция: [Docker installation guide](https://docs.docker.com/engine/install/).
- **Airflow**
- Пакет **astronomer-cosmos** с Docker-операторами dbt
- **Контейнер Postgres**
- **Docker-образ** с нужным dbt-проектом и DAG dbt
- **DAG dbt** с Docker-операторами dbt в каталоге DAGs Airflow для запуска в Airflow

Детали по пунктам 2–6 — ниже.

## Пошаговая инструкция

### 1. Установка Airflow и Cosmos

Создайте виртуальное окружение Python, активируйте его, обновите pip и установите [Apache Airflow®](https://airflow.apache.org/) и astronomer-cosmos с адаптером Postgres:

```bash
python -m venv venv
source venv/bin/activate
pip install --upgrade pip
pip install apache-airflow
pip install "astronomer-cosmos[dbt-postgres]"
```

### 2. Настройка базы Postgres

Нужна запущенная база Postgres для dbt-проекта. Запуск и проброс порта:

```bash
docker run --name some-postgres -e POSTGRES_PASSWORD="<postgres_password>" -e POSTGRES_USER=postgres -e POSTGRES_DB=postgres -p5432:5432 -d postgres
```

### 3. Сборка Docker-образа dbt

Чтобы Docker-операторы работали, нужен образ, который передаётся в параметр `image` операторов dbt в DAG.

Клонируйте репозиторий [cosmos-example](https://github.com/astronomer/cosmos-example.git):

```bash
git clone https://github.com/astronomer/cosmos-example.git
cd cosmos-example
```

Соберите образ с файлами dbt-проекта и профилем dbt, используя [Dockerfile](https://github.com/astronomer/cosmos-example/blob/main/Dockerfile.postgres_profile_docker_k8s):

```bash
docker build -t dbt-jaffle-shop:1.0.0 -f Dockerfile.postgres_profile_docker_k8s .
```

> На M1 при сбое сборки может понадобиться задать переменные окружения:
>
> Примечание

```bash
export DOCKER_BUILDKIT=0
export COMPOSE_DOCKER_CLI_BUILD=0
export DOCKER_DEFAULT_PLATFORM=linux/amd64
```

Просмотрите Dockerfile, чтобы понять, что он делает, и использовать его как образец в своём проекте.

- В образ добавляется [файл профиля dbt](https://github.com/astronomer/cosmos-example/blob/main/example_postgres_profile.yml).
- В образ добавляется каталог dags с [dbt-проектом jaffle_shop](https://github.com/astronomer/cosmos-example/tree/main/dags/dbt/jaffle_shop).
- Файл `dbt_project.yml` заменяется на [postgres_profile_dbt_project.yml](https://github.com/astronomer/cosmos-example/blob/main/postgres_profile_dbt_project.yml), где в ключе profile указан `postgres_profile`, так как для K8s/Docker-операторов создание профиля из подключений Airflow (как в режиме local) пока не поддерживается.

### 4. Настройка и запуск DAG в Airflow

Скопируйте каталог `dags` из репозитория cosmos-example в домашний каталог Airflow:

```bash
cp -r dags $AIRFLOW_HOME/
```

Запустите Airflow:

```bash
airflow standalone
```

> Если пользователь Airflow не имеет доступа к сокету Docker или к образам в кластере Kind, может потребоваться запуск `airflow standalone` с `sudo`.
>
> Примечание

Войдите в Airflow в браузере по адресу `http://localhost:8080/` (логин `airflow`, пароль — из файла `standalone_admin_password.txt`).

Включите и запустите DAG [jaffle_shop_docker](https://github.com/astronomer/cosmos-example/blob/main/dags/jaffle_shop_docker.py). После успешного выполнения DAG вы увидите завершённый прогон.

## Задание ProfileConfig

Начиная с Cosmos 1.8.0, в Docker-операторах Dbt DAG можно использовать аргумент `profile_config` и указывать профили dbt из файла `profiles.yml`. Путь к файлу задаётся параметром `profiles_yml_path` в `profile_config`.

В режиме `ExecutionMode.DOCKER` с `profile_config` работает только вариант с `profiles_yml_path`. Метод `profile_mapping` не подходит: подключения Airflow недоступны внутри контейнера, поэтому их нельзя сопоставить с профилем dbt.
