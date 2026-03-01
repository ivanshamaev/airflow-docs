# Начало работы с Cosmos на открытом Airflow (Getting Started on Open Source Airflow)

При использовании открытого (open-source) Airflow окружение может быть разным. В этом руководстве предполагается, что у вас есть возможность править базовый образ.

## Создание виртуального окружения

Создайте виртуальное окружение в `Dockerfile` по примеру ниже. Замените `<your-dbt-adapter>` на нужный адаптер (например `dbt-redshift`, `dbt-snowflake`). Виртуальное окружение рекомендуется, так как у dbt и Airflow могут конфликтовать зависимости.

```dockerfile
FROM my-image:latest

# установка dbt в виртуальное окружение
RUN python -m venv dbt_venv && source dbt_venv/bin/activate && \
    pip install --no-cache-dir <your-dbt-adapter> && deactivate
```

## Установка Cosmos

Установите `astronomer-cosmos` тем способом, которым вы обычно ставите Python-пакеты в своём окружении.

## Размещение dbt-проекта в каталоге DAGs

Создайте папку `dbt` внутри локальной папки `dags`, скопируйте в неё свой dbt-проект и создайте файл `my_cosmos_dag.py` в корне каталога `dags`.

По умолчанию Cosmos ищет проект в каталоге `/usr/local/airflow/dags/dbt`, но dbt-проект может лежать в любом месте на образе Airflow. Путь можно задать аргументом `dbt_project_dir` при создании экземпляра DAG.

Например, если проект должен лежать в `/usr/local/airflow/dags/my_dbt_project`:

```python
import os
from datetime import datetime

from cosmos import DbtDag, ProjectConfig, ProfileConfig, ExecutionConfig
from cosmos.profiles import PostgresUserPasswordProfileMapping

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
        "/usr/local/airflow/dags/my_dbt_project",
    ),
    profile_config=profile_config,
    execution_config=ExecutionConfig(
        dbt_executable_path=f"{os.environ['AIRFLOW_HOME']}/dbt_venv/bin/dbt",
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
