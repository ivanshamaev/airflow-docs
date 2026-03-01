# Операторы (Operators) в Airflow

Операторы — один из строительных блоков DAG в Airflow. В Airflow доступно много типов операторов. **PythonOperator** выполняет любую Python-функцию и по сути эквивалентен декоратору `@task`, а другие операторы содержат готовую логику для конкретных действий: выполнение Bash-скрипта (**BashOperator**), выполнение SQL-запроса в реляционной БД (**SQLExecuteQueryOperator**) и т.д. Операторы используются вместе с другими строительными блоками — [декораторами](../02.%20astronomer-dags/airflow-decorators.md) и [хуками](hooks.md) — для создания задач в DAG, написанном в task-oriented подходе. Классы операторов импортируются из пакетов провайдеров Airflow.

В этом руководстве вы узнаете основы использования операторов в Airflow.

Список доступных операторов по пакетам провайдеров: [Astronomer Registry](https://registry.astronomer.io/modules?typeName=Operators).

## Необходимая база

Чтобы получить максимум от руководства, нужно понимать:

- Основы Python. См. [Python Documentation](https://docs.python.org/3/tutorial/index.html).
- Основные концепции Airflow. См. [Введение в Apache Airflow](README.md).

## Основы операторов

Операторы — это Python-классы, которые инкапсулируют логику выполнения одной единицы работы. Их можно рассматривать как обёртку над каждой такой единицей: они задают выполняемые действия и скрывают большую часть кода, который пришлось бы писать вручную. Когда вы создаёте экземпляр оператора в DAG и передаёте ему нужные параметры, он становится **задачей (task)**.

Базовый набор операторов входит в пакет [Airflow standard provider](https://airflow.apache.org/docs/apache-airflow-providers-standard/stable/index.html), который предустановлен при использовании Astro CLI. Остальные операторы входят в специализированные пакеты провайдеров, часто ориентированные на конкретную технологию или сервис. Например, пакет [Airflow Snowflake Provider](https://registry.astronomer.io/providers/apache-airflow-providers-snowflake/versions/latest) содержит операторы для работы с Snowflake, а [Airflow Google provider](https://registry.astronomer.io/providers/apache-airflow-providers-google/versions/latest) — для сервисов Google Cloud. Есть также пакеты с операторами для целых групп сервисов:

- [Common Messaging](https://airflow.apache.org/docs/apache-airflow-providers-common-messaging/stable/index.html)
- [Common IO](https://registry.astronomer.io/providers/apache-airflow-providers-common-io/versions/latest)
- [Common SQL](https://registry.astronomer.io/providers/apache-airflow-providers-common-sql/versions/latest)

## Примеры операторов

Ниже — некоторые из самых часто используемых операторов Airflow. Указаны лишь часть параметров; полный список для каждого оператора см. в [Astronomer Registry](https://registry.astronomer.io/).

**EmptyOperator** — оператор «ничего не делающий» (no-op). Удобен для заглушек в DAG.

```python
from airflow.providers.standard.operators.empty import EmptyOperator

my_task = EmptyOperator(task_id="my_task")
```

**SQLExecuteQueryOperator** — выполняет SQL-запрос к реляционной БД.

```python
from airflow.providers.common.sql.operators.sql import SQLExecuteQueryOperator

my_task = SQLExecuteQueryOperator(
    task_id="my_task",
    sql="SELECT * FROM my_table",
    conn_id="my_conn_id",
)
```

**KubernetesPodOperator** — выполняет задачу, заданную Docker-образом, в Pod в Kubernetes. См. [Use the KubernetesPodOperator](../04.%20astronomer-advanced/kubernetes-pod-operator.md).

```python
from airflow.providers.cncf.kubernetes.operators.pod import KubernetesPodOperator

my_task = KubernetesPodOperator(
    task_id="my_task",
    kubernetes_conn_id="my_k8s_conn",
    name="my_pod",
    namespace="default",
    image="python:3.12-slim",
    cmds=["python", "-c"],
    arguments=["print('Hello world!')"],
)
```

**BashOperator** — выполняет bash-команду или скрипт. См. [BashOperator](bashoperator.md).

```python
from airflow.providers.standard.operators.bash import BashOperator

my_task = BashOperator(
    task_id="my_task",
    bash_command="echo 'Hello world!'",
)
```

**PythonOperator** — выполняет Python-функцию. Функционально эквивалентен декоратору `@task`. См. [Введение в TaskFlow API и декораторы Airflow](../02.%20astronomer-dags/airflow-decorators.md).

```python
from airflow.providers.standard.operators.python import PythonOperator

def _my_python_function():
    print("Hello world!")

my_task = PythonOperator(
    task_id="my_task",
    python_callable=_my_python_function,
)
```

Все операторы наследуются от абстрактного класса [BaseOperator](https://github.com/apache/airflow/blob/main/task-sdk/src/airflow/sdk/bases/operator.py), в котором реализовано выполнение работы оператора в контексте DAG.

Аргументы **BaseOperator** можно передавать во все операторы. Наиболее частые:

- **execution_timeout**: максимальное время выполнения задачи. Необязательный, по умолчанию `None`. Рекомендуется задавать, чтобы задачи не выполнялись бесконечно.
- **pool**: имя пула для задачи. Необязательный, по умолчанию `None`. См. [Пулы Airflow](../04.%20astronomer-advanced/airflow-pools.md).
- **retries**: число повторных попыток при сбое. Необязательный, по умолчанию 0. См. [Повторный запуск DAG и задач](../02.%20astronomer-dags/rerunning-dags.md).
- **task_id**: уникальный идентификатор задачи. Обязателен для всех операторов.

Эти и другие аргументы BaseOperator (кроме `task_id`, который должен быть уникален у каждой задачи) можно задать на уровне DAG для всех задач через словарь **default_args**. Для отдельной задачи их можно переопределить, указав те же аргументы в определении задачи.

## Пример DAG с операторами

В примере ниже DAG загружает данные из CSV в S3 и затем в Redshift с проверками целостности и качества. Используются операторы (LocalFilesystemToS3Operator, PostgresOperator, S3ToRedshiftOperator, SQLCheckOperator, EmptyOperator), задача с декоратором `@task` и TaskGroup.

```python
import hashlib
import json

from airflow.exceptions import AirflowException
from airflow.decorators import dag, task
from airflow.models import Variable
from airflow.models.baseoperator import chain
from airflow.operators.empty import EmptyOperator
from airflow.utils.dates import datetime
from airflow.providers.amazon.aws.hooks.s3 import S3Hook
from airflow.providers.amazon.aws.transfers.local_to_s3 import (
    LocalFilesystemToS3Operator,
)
from airflow.providers.amazon.aws.transfers.s3_to_redshift import S3ToRedshiftOperator
from airflow.providers.postgres.operators.postgres import PostgresOperator
from airflow.operators.sql import SQLCheckOperator
from airflow.utils.task_group import TaskGroup


CSV_FILE_NAME = "forestfires.csv"
CSV_FILE_PATH = f"include/sample_data/forestfire_data/{CSV_FILE_NAME}"


@dag(
    "simple_redshift_3",
    start_date=datetime(2021, 7, 7),
    description="""Sample DAG: load CSV to S3 and Redshift with integrity and quality checks.""",
    schedule=None,
    template_searchpath="/usr/local/airflow/include/sql/redshift_examples/",
    catchup=False,
)
def simple_redshift_3():
    """
    Before running, set Variable aws_configs with keys:
    s3_bucket, s3_key_prefix, redshift_table.
    """

    upload_file = LocalFilesystemToS3Operator(
        task_id="upload_to_s3",
        filename=CSV_FILE_PATH,
        dest_key="{{ var.json.aws_configs.s3_key_prefix }}/" + CSV_FILE_PATH,
        dest_bucket="{{ var.json.aws_configs.s3_bucket }}",
        aws_conn_id="aws_default",
        replace=True,
    )

    @task
    def validate_etag():
        """Check destination ETag vs local MD5."""
        s3 = S3Hook()
        aws_configs = Variable.get("aws_configs", deserialize_json=True)
        obj = s3.get_key(
            key=f"{aws_configs.get('s3_key_prefix')}/{CSV_FILE_PATH}",
            bucket_name=aws_configs.get("s3_bucket"),
        )
        obj_etag = obj.e_tag.strip('"')
        file_hash = hashlib.md5(open(CSV_FILE_PATH).read().encode("utf-8")).hexdigest()
        if obj_etag != file_hash:
            raise AirflowException("Upload Error: ETag in S3 did not match local file hash.")

    validate_file = validate_etag()

    create_redshift_table = PostgresOperator(
        task_id="create_table",
        sql="create_redshift_forestfire_table.sql",
        postgres_conn_id="redshift_default",
    )

    load_to_redshift = S3ToRedshiftOperator(
        task_id="load_to_redshift",
        s3_bucket="{{ var.json.aws_configs.s3_bucket }}",
        s3_key="{{ var.json.aws_configs.s3_key_prefix }}" + f"/{CSV_FILE_PATH}",
        schema="PUBLIC",
        table="{{ var.json.aws_configs.redshift_table }}",
        copy_options=["csv"],
    )

    validate_redshift = SQLCheckOperator(
        task_id="validate_redshift",
        conn_id="redshift_default",
        sql="validate_redshift_forestfire_load.sql",
        params={"filename": CSV_FILE_NAME},
    )

    with open("include/validation/forestfire_validation.json") as ffv:
        with TaskGroup(group_id="row_quality_checks") as quality_check_group:
            ffv_json = json.load(ffv)
            for id, values in ffv_json.items():
                values["id"] = id
                SQLCheckOperator(
                    task_id=f"forestfire_row_quality_check_{id}",
                    conn_id="redshift_default",
                    sql="row_quality_redshift_forestfire_check.sql",
                    params=values,
                )

    drop_redshift_table = PostgresOperator(
        task_id="drop_table",
        sql="drop_redshift_forestfire_table.sql",
        postgres_conn_id="redshift_default",
    )

    begin = EmptyOperator(task_id="begin")
    end = EmptyOperator(task_id="end")

    chain(
        begin,
        upload_file,
        validate_file,
        create_redshift_table,
        load_to_redshift,
        validate_redshift,
        quality_check_group,
        drop_redshift_table,
        end,
    )


simple_redshift_3()
```

## Рекомендации

Обычно оператору нужны лишь несколько параметров. При использовании операторов Airflow учитывайте следующее:

- Оператор, работающий с внешним сервисом, как правило, требует **подключение (connection)** для аутентификации. Подробнее: [Управление подключениями в Apache Airflow](connections.md).
- [Deferrable-операторы](../04.%20astronomer-advanced/deferrable-operators.md) освобождают слот воркера во время ожидания завершения работы. Это снижает затраты и улучшает масштабируемость. Astronomer рекомендует использовать deferrable-операторы, когда они есть для вашего сценария и задача выполняется дольше минуты. У многих операторов, которым нужно что-то ждать, есть режим deferrable — включить его можно параметром `deferrable=True`.
- [Сенсоры](sensors.md) — тип операторов, которые ждут наступления события. Их используют для отслеживания событий во внешних системах.
- Если под вашу задачу нет оператора, можно использовать свой Python-код в задаче с декоратором `@task` или в **PythonOperator**, либо расширить существующий оператор. Подробнее: [Custom hooks and operators](../02.%20astronomer-dags/custom-hooks-operators.md).
- Если оператор для вашей задачи есть — используйте его вместо своих функций или [хуков](hooks.md). Так DAG проще читать и поддерживать.
- Операторы и [декораторы](../02.%20astronomer-dags/airflow-decorators.md) можно свободно сочетать в одном DAG. Многие используют `@task` для большинства задач и добавляют операторы там, где есть подходящий специализированный оператор. В примере выше в DAG есть и оператор (BashOperator в других примерах), и задача с `@task`.
- Пакет [Airflow standard provider](https://airflow.apache.org/docs/apache-airflow-providers-standard/stable/index.html) включает базовые операторы вроде PythonOperator и BashOperator. Они доступны по умолчанию при использовании Astro CLI. Остальные операторы входят в пакеты провайдеров; часть из них нужно устанавливать отдельно в зависимости от дистрибутива Airflow.
- [Astronomer Registry](https://registry.astronomer.io/modules?types=operators) — лучший источник информации о доступных операторах и способах их использования.

---

[← Управление кодом](managing-airflow-code.md) | [К содержанию](README.md) | [BashOperator →](bashoperator.md)
