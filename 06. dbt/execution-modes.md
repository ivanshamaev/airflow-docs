# Режимы выполнения (Execution Modes)

Cosmos может запускать команды `dbt` разными способами — **режимами выполнения (execution modes)**:

- **local** — запуск команд `dbt` через локальную установку `dbt` (по умолчанию)
- **virtualenv** — запуск команд `dbt` из виртуальных окружений Python, которыми управляет Cosmos
- **docker** — запуск команд `dbt` из Docker-контейнеров, которыми управляет Cosmos (нужен готовый Docker-образ)
- **kubernetes** — запуск команд `dbt` из Pod в Kubernetes, которыми управляет Cosmos (нужен готовый Docker-образ)
- **aws_eks** — запуск команд `dbt` из Pod в AWS EKS, которыми управляет Cosmos (нужен готовый Docker-образ)
- **azure_container_instance** — запуск команд `dbt` из Azure Container Instances, которыми управляет Cosmos (нужен готовый Docker-образ)
- **gcp_cloud_run_job** — запуск команд `dbt` из GCP Cloud Run Job, которыми управляет Cosmos (нужен готовый Docker-образ)
- **aws_ecs** — запуск команд `dbt` из экземпляров AWS ECS, которыми управляет Cosmos (нужен готовый Docker-образ)
- **airflow_async** — (стабилен с Cosmos 1.9.0) асинхронный запуск ресурсов dbt: скомпилированный SQL отправляется в [Deferrable-операторы](https://airflow.apache.org/docs/apache-airflow/stable/authoring-and-scheduling/deferring.html) Apache Airflow
- **watcher** — (экспериментален с Cosmos 1.11.0) одна команда `dbt build` в задаче-продюсере и сенсорные задачи, следящие за прогрессом; ускорение DAG при сохранении графа задач в UI Airflow и возможности повторного запуска упавших задач. Подробнее: [Introducing ExecutionMode.WATCHER](https://astronomer.github.io/astronomer-cosmos/getting_started/watcher-execution-mode.html#watcher-execution-mode)
- **watcher_kubernetes** — (экспериментален с Cosmos 1.13.0) скорость режима watcher плюс изоляция Kubernetes. Подробнее: [ExecutionMode.WATCHER_KUBERNETES](https://astronomer.github.io/astronomer-cosmos/getting_started/watcher-kubernetes-execution-mode.html#watcher-kubernetes-execution-mode)

Выбор режима зависит от требований и ограничений. Ниже описаны отдельные режимы.

## Сравнение режимов выполнения

| Режим | Длительность задач | Изоляция окружения | Управление профилями Cosmos |
|-------|--------------------|--------------------|-----------------------------|
| Local | Быстро | Нет | Да |
| Virtualenv | Средне | Лёгкая | Да |
| Docker | Медленно | Средняя | Нет |
| Kubernetes | Медленно | Высокая | Нет |
| AWS_EKS | Медленно | Высокая | Нет |
| Azure Container Instance | Медленно | Высокая | Нет |
| GCP Cloud Run Job | Медленно | Высокая | Нет |
| AWS ECS | Медленно | Высокая | Нет |
| Airflow Async | Очень быстро | Средняя | Да |
| Watcher | Очень быстро | Нет | Да |
| Watcher Kubernetes | Быстро | Высокая | Нет |

## Local

По умолчанию Cosmos использует режим **local**.

Режим `local` — самый быстрый: не требуется установка `dbt` и сборка контейнеров. Но он может не подойти для управляемых сервисов вроде Google Cloud Composer: зависимости Airflow и dbt могут конфликтовать ([конфликты зависимостей](https://astronomer.github.io/astronomer-cosmos/getting_started/execution-modes-local-conflicts.html#execution-modes-local-conflicts)), и установить `dbt` в свой путь не всегда возможно.

В режиме `local` предполагается, что исполняемый файл `dbt` доступен на воркере Airflow.

Если `dbt` не установлен вместе с пакетами Cosmos, путь к нему можно задать аргументом `dbt_executable_path`.

> Начиная с версии 1.4, Cosmos использует partial parsing dbt (`partial_parse.msgpack`) для ускорения выполнения задач. Возможности ограничены [ограничениями partial parsing в dbt](https://docs.getdbt.com/reference/parsing#known-limitations). Подробнее: [Partial parsing](https://astronomer.github.io/astronomer-cosmos/configuration/partial-parsing.html#partial-parsing).
>
> Примечание

В режиме `local` Cosmos превращает подключения Airflow в нативный файл профилей dbt (`profiles.yml`).

Пример (когда `dbt` установлен вместе с Cosmos):

```python
basic_cosmos_dag = DbtDag(
    # параметры dbt/cosmos
    project_config=ProjectConfig(DBT_PROJECT_PATH),
    profile_config=profile_config,
    operator_args={
        "install_deps": True,   # установка зависимостей перед командами dbt
        "full_refresh": True,   # только для команд dbt с этим флагом
    },
    # обычные параметры DAG
    schedule="@daily",
    start_date=datetime(2023, 1, 1),
    catchup=False,
    dag_id="basic_cosmos_dag",
    default_args={"retries": 0},
)
```

## Virtualenv

Для управляемого Airflow на GCP (Cloud Composer) рекомендуется режим **virtualenv**.

Режим `virtualenv` изолирует зависимости воркера Airflow от `dbt`, создавая виртуальное окружение Python во время выполнения задачи и удаляя его после.

Версию `dbt` задаёт пользователь через аргумент `py_requirements`. Его можно передать в операторах или при создании `DbtDag` и `DbtTaskGroup` в `operator_args`.

Как и в режиме `local`, Cosmos преобразует подключения Airflow в профиль dbt (`profiles.yml`) и по умолчанию пытается использовать `partial_parse.msgpack` для ускорения разбора.

Минусы:

- Медленнее `local`, так как для каждой задачи Cosmos создаётся новое виртуальное окружение.
- Если dbt недоступен в планировщике Airflow, режим загрузки по умолчанию `LoadMode.DBT_LS` не сработает. Нужен [метод разбора](https://astronomer.github.io/astronomer-cosmos/configuration/parsing-methods.html#parsing-methods), не зависящий от dbt, например `LoadMode.MANIFEST`.
- Сейчас поддерживается только `InvocationMode.SUBPROCESS`; использование `InvocationMode.DBT_RUNNER` приведёт к ошибке.

Пример:

```python
@dag(
    schedule="@daily",
    start_date=datetime(2023, 1, 1),
    catchup=False,
)
def example_virtualenv() -> None:
    start_task = EmptyOperator(task_id="start-venv-examples")
    end_task = EmptyOperator(task_id="end-venv-examples")

    # Первая группа: новое виртуальное окружение Cosmos на каждую задачу, затем удаление
    # Значительно медленнее, чем при задании virtualenv_dir
    tmp_venv_task_group = DbtTaskGroup(
        group_id="tmp-venv-group",
        project_config=ProjectConfig(DBT_ROOT_PATH / "jaffle_shop"),
        profile_config=profile_config,
        execution_config=ExecutionConfig(
            execution_mode=ExecutionMode.VIRTUALENV,
            # без virtualenv_dir Cosmos создаёт новое venv для каждой dbt-задачи
        ),
        operator_args={
            "py_system_site_packages": False,
            "py_requirements": ["dbt-postgres"],
            "install_deps": True,
            "emit_datasets": False,
            # callback для загрузки файлов можно раскомментировать при необходимости
        },
    )

    # Вторая группа: переиспользует одно venv для всех задач, примерно на 70% быстрее
    cached_venv_task_group = DbtTaskGroup(
        group_id="cached-venv-group",
        project_config=ProjectConfig(DBT_ROOT_PATH / "jaffle_shop"),
        profile_config=profile_config,
        execution_config=ExecutionConfig(
            execution_mode=ExecutionMode.VIRTUALENV,
            virtualenv_dir=Path("/tmp/persistent-venv2"),
        ),
        operator_args={
            "py_system_site_packages": False,
            "py_requirements": ["dbt-postgres"],
            "install_deps": True,
        },
    )

    start_task >> [tmp_venv_task_group, cached_venv_task_group] >> end_task

example_virtualenv()
```

## Docker

В режиме **docker** предполагается уже собранный Docker-образ с пайплайнами dbt и `profiles.yml`, которые ведёт пользователь.

Изоляция окружения лучше, чем у `local` и `virtualenv`, но выше ответственность: актуальность файлов в образе и управление секретами (возможно, в нескольких местах).

Дополнительная сложность — воркер Airflow уже может работать в Docker, что порождает задачи [Docker-in-Docker](https://devops.stackexchange.com/questions/676/why-is-docker-in-docker-considered-bad).

Режим может быть заметно медленнее `virtualenv` из‑за сборки/запуска контейнера. Если dbt недоступен в планировщике Airflow, нужен метод разбора, не зависящий от dbt (например `LoadMode.MANIFEST`). Пошаговая инструкция: [Docker Execution Mode](https://astronomer.github.io/astronomer-cosmos/getting_started/docker.html#docker).

Пример DAG:

```python
docker_cosmos_dag = DbtDag(
    # ...
    execution_config=ExecutionConfig(
        execution_mode=ExecutionMode.DOCKER,
    ),
    operator_args={
        "image": "dbt-jaffle-shop:1.0.0",
        "network_mode": "bridge",
    },
)
```

## Kubernetes

Режим **kubernetes** даёт сильную изоляцию: команды dbt выполняются в Pod Kubernetes, как правило на отдельном хосте.

Нужен кластер Kubernetes. Пользователь должен следить за актуальностью dbt-пайплайнов и профилей в образе; секреты могут понадобиться и в Airflow, и в контейнере.

По скорости может уступать Docker и Virtualenv (сборка образа и поднятие Pod). Пошаговая инструкция: [Kubernetes Execution Mode](https://astronomer.github.io/astronomer-cosmos/getting_started/kubernetes.html#kubernetes).

Пример DAG:

```python
load_seeds = DbtSeedKubernetesOperator(
    task_id="load_seeds",
    project_dir=K8S_PROJECT_DIR,
    get_logs=True,
    schema="public",
    image=DBT_IMAGE,
    is_delete_operator_pod=False,
    secrets=[postgres_password_secret, postgres_host_secret],
    profile_config=ProfileConfig(
        profiles_yml_filepath="/root/.dbt/profiles.yml",
        profile_name="postgres_profile",
        target_name="dev",
    ),
    env_vars={
        "POSTGRES_DB": "postgres",
        "POSTGRES_SCHEMA": "public",
        "POSTGRES_USER": "postgres",
    },
)
```

## AWS_EKS

Режим **aws_eks** похож на `kubernetes`, но рассчитан на кластеры AWS EKS. Используется [EKSPodOperator](https://airflow.apache.org/docs/apache-airflow-providers-amazon/8.19.0/operators/eks.html#perform-a-task-on-an-amazon-eks-cluster). В `operator_args` нужно указать `cluster_name` для подключения к кластеру EKS.

Пример DAG:

```python
postgres_password_secret = Secret(
    deploy_type="env",
    deploy_target="POSTGRES_PASSWORD",
    secret="postgres-secrets",
    key="password",
)

docker_cosmos_dag = DbtDag(
    # ...
    execution_config=ExecutionConfig(
        execution_mode=ExecutionMode.AWS_EKS,
    ),
    operator_args={
        "image": "dbt-jaffle-shop:1.0.0",
        "cluster_name": CLUSTER_NAME,
        "get_logs": True,
        "is_delete_operator_pod": False,
        "secrets": [postgres_password_secret],
    },
)
```

## Azure Container Instance

Добавлено в версии 1.4.

Запуск в **Azure Container Instances** даёт сильную изоляцию: dbt выполняется в контейнере в Azure.

Нужна среда Azure с возможностью запуска Container Groups (подробнее: [Azure Container Instance Execution Mode](https://astronomer.github.io/astronomer-cosmos/getting_started/azure-container-instance.html#azure-container-instance)). Как и для Docker/Kubernetes, нужен образ с актуальными dbt-пайплайнами и профилями.

Каждая задача создаёт новый контейнер в Azure (полная изоляция), но с дополнительными накладными расходами по времени. Пошаговая инструкция — в документации по режиму Azure Container Instance.

```python
docker_cosmos_dag = DbtDag(
    # ...
    execution_config=ExecutionConfig(
        execution_mode=ExecutionMode.AZURE_CONTAINER_INSTANCE
    ),
    operator_args={
        "ci_conn_id": "aci",
        "registry_conn_id": "acr",
        "resource_group": "my-rg",
        "name": "my-aci-{{ ti.task_id.replace('.','-').replace('_','-') }}",
        "region": "West Europe",
        "image": "dbt-jaffle-shop:1.0.0",
    },
)
```

## GCP Cloud Run Job

Добавлено в версии 1.7.

Режим **gcp_cloud_run_job** удобен, когда нужно запускать dbt на инфраструктуре Google Cloud с масштабированием, изоляцией и управляемым сервисом Cloud Run.

Сначала нужно создать Cloud Run Job из собранного Docker-образа с актуальными dbt-пайплайнами и профилями ([создание Cloud Run Job](https://cloud.google.com/run/docs/create-jobs)).

Команды dbt Core CLI выполняются в экземпляре Cloud Run Job через `CloudRunExecuteJobOperator` провайдера Google Cloud. Каждая задача — новое выполнение Job (полная изоляция). Накладные расходы можно снизить параметром `concurrency` в `DbtDag` для параллельного выполнения моделей dbt.

```python
gcp_cloud_run_job_cosmos_dag = DbtDag(
    # ...
    execution_config=ExecutionConfig(execution_mode=ExecutionMode.GCP_CLOUD_RUN_JOB),
    operator_args={
        "project_id": "my-gcp-project-id",
        "region": "europe-west1",
        "job_name": "my-crj-{{ ti.task_id.replace('.','-').replace('_','-') }}",
    },
)
```

## AWS ECS

Добавлено в версии 1.9.0.

Режим **AWS Elastic Container Service (ECS)** запускает задачи dbt в изолированных и масштабируемых контейнерах в сервисе ECS.

Нужна настроенная среда AWS для запуска ECS-задач (подробнее в документации по [aws-ecs](https://astronomer.github.io/astronomer-cosmos/getting_started/aws-ecs.html)). Как и для Docker/Kubernetes, нужен образ с актуальными dbt-пайплайнами и профилями.

Каждая задача — новое выполнение ECS task (полная изоляция). Задержки связаны с запуском контейнера; их можно уменьшить настройкой task definition и кластера. Пошаговая инструкция — в документации по режиму AWS ECS.

```python
aws_ecs_cosmos_dag = DbtDag(
    # ...
    execution_config=ExecutionConfig(execution_mode=ExecutionMode.AWS_ECS),
    operator_args={
        "aws_conn_id": "aws_default",
        "cluster": "my-ecs-cluster",
        "task_definition": "my-dbt-task",
        "container_name": "dbt-container",
        "launch_type": "FARGATE",
        "deferrable": True,
        "network_configuration": {
            "awsvpcConfiguration": {
                "subnets": ["<<<YOUR SUBNET ID>>>"],
                "assignPublicIp": "ENABLED",
            },
        },
        "environment_variables": {"DBT_PROFILE_NAME": "default"},
    },
)
```

## Airflow Async

Добавлено в версии 1.9.0.

Рекомендуется использовать Cosmos 1.11+ с улучшениями производительности. По сравнению с `local`, режим **airflow_async** может сократить время выполнения dbt-проекта до 36%.

В режиме `airflow_async` ресурсы dbt запускаются через [Deferrable-операторы](https://airflow.apache.org/docs/apache-airflow/stable/authoring-and-scheduling/deferring.html) Apache Airflow. Режим удобен при длительных задачах и желании не занимать слоты воркеров: больше dbt-узлов могут выполняться параллельно.

Пример DAG:

```python
simple_dag_async = DbtDag(
    project_config=ProjectConfig(DBT_PROJECT_PATH),
    profile_config=profile_config,
    execution_config=ExecutionConfig(
        execution_mode=ExecutionMode.AIRFLOW_ASYNC,
        async_py_requirements=[f"dbt-bigquery=={DBT_ADAPTER_VERSION}"],
    ),
    render_config=RenderConfig(select=["path:models"], test_behavior=TestBehavior.NONE),
    schedule=None,
    start_date=datetime(2023, 1, 1),
    catchup=False,
    dag_id="simple_dag_async",
    tags=["simple"],
    operator_args={
        "location": "US",
        "install_deps": True,
        "full_refresh": True,
    },
)
```

Подробная инструкция и ограничения: [Airflow Async Execution Mode](https://astronomer.github.io/astronomer-cosmos/getting_started/async-execution-mode.html#async-execution-mode).

## Watcher (экспериментальный)

Добавлено в версии 1.11.0.

Режим **watcher** (экспериментальный): одна команда `dbt build` в задаче-продюсере и сенсорные задачи, следящие за прогрессом. Цель — ускорить DAG при сохранении графа задач в UI Airflow и возможности повторного запуска упавших задач.

Подробнее: [Introducing ExecutionMode.WATCHER](https://astronomer.github.io/astronomer-cosmos/getting_started/watcher-execution-mode.html#watcher-execution-mode).

## Watcher Kubernetes (экспериментальный)

Добавлено в версии 1.13.0.

Режим **watcher_kubernetes** сочетает скорость watcher и изоляцию kubernetes: одна команда `dbt build` выполняется в задаче-продюсере внутри Pod в Kubernetes, сенсоры следят за прогрессом.

Подробнее: [ExecutionMode.WATCHER_KUBERNETES](https://astronomer.github.io/astronomer-cosmos/getting_started/watcher-kubernetes-execution-mode.html#watcher-kubernetes-execution-mode).

# Режимы вызова (Invocation Modes)

Добавлено в версии 1.4.

Для режима **ExecutionMode.LOCAL** в Cosmos доступны два режима вызова:

- **InvocationMode.SUBPROCESS** — Cosmos запускает команды dbt через модуль `subprocess` и разбирает вывод для логов и исключений.
- **InvocationMode.DBT_RUNNER** — Cosmos использует [dbtRunner](https://docs.getdbt.com/reference/programmatic-invocations) для программного вызова dbt. dbt должен быть установлен в том же окружении. Нет накладных расходов на subprocess и разбор вывода, режим быстрее `SUBPROCESS`. Требуется dbt 1.5.0+. Конфликты зависимостей Airflow и dbt пользователь решает сам.

Режим вызова задаётся в `ExecutionConfig`:

```python
from cosmos.constants import InvocationMode

dag = DbtDag(
    # ...
    execution_config=ExecutionConfig(
        execution_mode=ExecutionMode.LOCAL,
        invocation_mode=InvocationMode.DBT_RUNNER,
    ),
)
```

Если режим не задан, Cosmos пробует использовать `InvocationMode.DBT_RUNNER`, если dbt установлен в том же окружении, что и воркер; иначе используется `InvocationMode.SUBPROCESS`.
