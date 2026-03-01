# Начало работы с Cosmos на Astro (Getting Started on Astro)

Cosmos на Astro можно использовать со всеми [режимами выполнения](https://astronomer.github.io/astronomer-cosmos/getting_started/execution-modes.html#execution-modes), но рекомендуется режим `local` — он проще всего в настройке и использовании.

Готовый рабочий проект для запуска на Astro (CLI или Cloud): [cosmos-demo](https://github.com/astronomer/cosmos-demo).

Ниже — пошаговая инструкция по запуску своего dbt-проекта в Astro.

## Предварительные требования

Перед началом нужно иметь:

- Установленный **Astro CLI**. Инструкции: [здесь](https://docs.astronomer.io/astro/cli/install-cli).
- Проект Astro CLI. Новый проект создаётся командой `astro dev init`.
- Проект dbt. Подойдёт, например, [jaffle shop](https://github.com/dbt-labs/jaffle-shop-classic).

## Создание виртуального окружения

Создайте виртуальное окружение в `Dockerfile` по примеру ниже. Замените `<your-dbt-adapter>` на нужный адаптер (например `dbt-redshift`, `dbt-snowflake`). Виртуальное окружение рекомендуется, так как у dbt и Airflow могут конфликтовать зависимости.

```dockerfile
FROM quay.io/astronomer/astro-runtime:11.3.0

# установка dbt в виртуальное окружение
RUN python -m venv dbt_venv && source dbt_venv/bin/activate && \
    pip install --no-cache-dir <your-dbt-adapter> && deactivate
```

Пример адаптера: `dbt-postgres`.

## Установка Cosmos

Добавьте Cosmos в `requirements.txt` проекта:

```text
astronomer-cosmos
```

## Размещение dbt-проекта в каталоге DAGs

Создайте папку `dbt` внутри локальной папки `dags`, скопируйте в неё свой dbt-проект и создайте файл `my_cosmos_dag.py` в корне каталога `dags`. Структура проекта должна быть такой:

```text
├── dags/
│   ├── dbt/
│   │   └── my_dbt_project/
│   │       ├── dbt_project.yml
│   │       ├── models/
│   │       │   ├── my_model.sql
│   │       │   └── my_other_model.sql
│   │       └── macros/
│   │           ├── my_macro.sql
│   │           └── my_other_macro.sql
│   └── my_cosmos_dag.py
├── Dockerfile
├── requirements.txt
└── ...
```

По умолчанию Cosmos ищет проект в каталоге `/usr/local/airflow/dags/dbt`, но dbt-проект может лежать в любом месте на образе Airflow. Путь можно задать аргументом `dbt_project_dir` при создании экземпляра DAG.

Например, если проект должен лежать в `/usr/local/airflow/dags/my_dbt_project`:

```python
from cosmos import DbtDag, ProjectConfig

my_cosmos_dag = DbtDag(
    project_config=ProjectConfig(
        dbt_project_path="/usr/local/airflow/dags/my_dbt_project",
    ),
    # ...,
)
```

## Создание DAG-файла

В файле `my_cosmos_dag.py` импортируйте класс `DbtDag` из Cosmos и создайте экземпляр DAG. Обязательно укажите аргумент `dbt_executable_path`, чтобы он указывал на виртуальное окружение из шага 1.

```python
from cosmos import DbtDag, ProjectConfig, ProfileConfig, ExecutionConfig
from cosmos.profiles import PostgresUserPasswordProfileMapping

import os
from datetime import datetime

airflow_home = os.environ["AIRFLOW_HOME"]

profile_config = ProfileConfig(
    profile_name="default",
    target_name="dev",
    profile_mapping=PostgresUserPasswordProfileMapping(
        conn_id="airflow_db",
        profile_args={"schema": "public"},
    ),
)

my_cosmos_dag = DbtDag(
    project_config=ProjectConfig(
        f"{airflow_home}/dags/my_dbt_project",
    ),
    profile_config=profile_config,
    execution_config=ExecutionConfig(
        dbt_executable_path=f"{airflow_home}/dbt_venv/bin/dbt",
    ),
    # обычные параметры DAG
    schedule_interval="@daily",
    start_date=datetime(2023, 1, 1),
    catchup=False,
    dag_id="my_cosmos_dag",
    default_args={"retries": 2},
)
```

> В больших dbt-проектах иногда возникает ошибка `DagBag import timeout`. Её можно устранить, увеличив значение настройки Airflow [core.dagbag_import_timeout](https://airflow.apache.org/docs/apache-airflow/stable/configurations-ref.html#dagbag-import-timeout).
>
> Примечание

## Запуск проекта

Запустите проект командой `astro dev start`. DAG Airflow появится в UI Airflow (по умолчанию `localhost:8080`), откуда его можно запускать.
