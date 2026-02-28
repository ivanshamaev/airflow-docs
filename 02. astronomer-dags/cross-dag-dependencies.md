# Зависимости между DAG (Cross-DAG dependencies)

Обычно связанные задачи лучше держать в одном DAG, но иногда нужны зависимости между разными DAG. В таком случае узлом графа становится целый DAG, а не одна задача. В руководстве используются термины:

- **Downstream DAG** (нижестоящий DAG): DAG, который не может запуститься, пока вышестоящий DAG не перейдёт в заданное состояние.
- **Upstream DAG** (вышестоящий DAG): DAG, который должен достичь заданного состояния, прежде чем сможет выполниться нижестоящий DAG.

Тема [Cross-DAG Dependencies](https://airflow.apache.org/docs/apache-airflow/stable/howto/operator/external_task_sensor.html#cross-dag-dependencies) в документации Airflow описывает ситуации, когда такие зависимости полезны:

- Задача зависит от другой задачи, но с другой датой выполнения.
- Два DAG связаны по смыслу, но принадлежат разным командам.
- Два DAG связаны, но с разными расписаниями.
- DAG должен запускаться только после обновления одного или нескольких ассетов задачами из других DAG.

В этом руководстве — способы реализации зависимостей между DAG, в том числе когда зависимые DAG находятся в разных развёртываниях Airflow.

## Необходимая база

Полезно понимать:

- Сенсоры Airflow. См. [Что такое сенсор?](../01.%20astronomer-basic/sensors.md).
- Операторы Airflow. См. [Что такое оператор?](../01.%20astronomer-basic/operators.md).
- DAG в Airflow. См. [Введение в DAG](../01.%20astronomer-basic/dags.md).
- Зависимости в Airflow. См. [Управление зависимостями](../01.%20astronomer-basic/task-dependencies.md).

## Реализация зависимостей между DAG

Зависимости между DAG в Airflow можно задать несколькими способами:

- [REST API Airflow](https://airflow.apache.org/docs/apache-airflow/stable/stable-rest-api-ref.html).
- [ExternalTaskSensor](https://registry.astronomer.io/providers/apache-airflow/modules/externaltasksensor).
- [TriggerDagRunOperator](https://registry.astronomer.io/providers/apache-airflow/modules/triggerdagrunoperator).
- [Data-aware планирование с ассетами](https://www.astronomer.io/docs/learn/airflow-datasets).

Ниже — когда и как использовать каждый способ и как смотреть зависимости в UI Airflow.

### Зависимости через ассеты (Dataset dependencies)

Самый распространённый способ задать зависимости между DAG — [ассеты](../01.%20astronomer-basic/assets.md). DAG, работающие с одними и теми же данными, могут иметь явные связи, а запуск DAG можно привязать к обновлению этих данных.

Этот способ подходит, когда нижестоящий DAG должен запускаться только после обновления ассета вышестоящим DAG, особенно при нерегулярных обновлениях. Зависимости между DAG и ассетами при этом хорошо видны в UI Airflow.

При работе с ассетами используются понятия:

- **Consuming DAG** (потребляющий DAG): DAG, который запускается сразу после обновления указанного ассета.
- **Producing task** (производящая задача): задача, обновляющая ассет; задаётся параметром `outlets`.

Любая задача становится производящей, если в параметр `outlets` передать один или несколько ассетов. Пример:

```python
from airflow.sdk import Asset
from airflow.providers.standard.operators.empty import EmptyOperator

asset1 = Asset("asset1")

# производящая задача в вышестоящем DAG
EmptyOperator(
    task_id="producing_task",
    outlets=[asset1],  # помечает для Airflow, что asset1 обновлён
)
```

Нижестоящий DAG запускается после обновления `asset1`, если передать его в параметр `schedule`:

```python
from airflow.sdk import Asset, dag

asset1 = Asset("asset1")


@dag(schedule=[asset1])
def consuming_dag():
    ...
```

Подробнее: [Ассеты и data-aware планирование в Airflow](../01.%20astronomer-basic/assets.md).

### TriggerDagRunOperator

**TriggerDagRunOperator** — простой способ задать зависимость «сверху вниз»: задача в одном DAG запускает другой DAG в том же окружении Airflow. Подробнее: [TriggerDagRunOperator](https://registry.astronomer.io/providers/apache-airflow/modules/triggerdagrunoperator).

Запускать нижестоящий DAG можно из любой точки вышестоящего DAG. Если у оператора задать `wait_for_completion=True`, вышестоящий DAG приостанавливается и продолжается только после завершения нижестоящего. Ожидание можно переложить на triggerer, установив `deferrable=True`: оператор станет [deferrable](https://www.astronomer.io/docs/learn/deferrable-operators), что снижает нагрузку и может уменьшить затраты.

Типичный сценарий: вышестоящий DAG загружает тестовые данные для ML-пайплайна, обучает и тестирует модель, публикует предсказания. Если модель показывает слабый результат, TriggerDagRunOperator запускает отдельный DAG переобучения; вышестоящий DAG ждёт. После переобучения и проверки в нижестоящем DAG вышестоящий продолжается и публикует результаты новой модели.

[Расписание](https://www.astronomer.io/docs/learn/scheduling-in-airflow) нижестоящего DAG не связано с запусками через TriggerDagRunOperator. Чтобы DAG запускался только этим оператором, задайте ему `schedule=None`. Зависимый DAG должен быть включён (не на паузе).

В примере ниже TriggerDagRunOperator запускает DAG с `dag_id` `dependent_dag` между двумя другими задачами. У задачи `trigger_dependent_dag` заданы `wait_for_completion=True` и `deferrable=True`, поэтому задача откладывается до завершения `dependent_dag`. После завершения этой задачи выполняется `end_task`.

**TaskFlow:**

```python
from airflow.decorators import dag, task
from airflow.operators.trigger_dagrun import TriggerDagRunOperator
from pendulum import datetime, duration


@task
def start_task(task_type):
    return f"The {task_type} task has completed."


@task
def end_task(task_type):
    return f"The {task_type} task has completed."


default_args = {
    "owner": "airflow",
    "depends_on_past": False,
    "email_on_failure": False,
    "email_on_retry": False,
    "retries": 1,
    "retry_delay": duration(minutes=5),
}


@dag(
    start_date=datetime(2023, 1, 1),
    max_active_runs=1,
    schedule="@daily",
    default_args=default_args,
    catchup=False,
)
def trigger_dagrun_dag():
    trigger_dependent_dag = TriggerDagRunOperator(
        task_id="trigger_dependent_dag",
        trigger_dag_id="dependent_dag",
        wait_for_completion=True,
        deferrable=True,  # параметр доступен в Airflow 2.6+
    )
    start_task("starting") >> trigger_dependent_dag >> end_task("ending")


trigger_dagrun_dag()
```

**Традиционный вариант:**

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.operators.trigger_dagrun import TriggerDagRunOperator
from pendulum import datetime, duration


def print_task_type(**kwargs):
    print(f"The {kwargs['task_type']} task has completed.")


default_args = {
    "owner": "airflow",
    "depends_on_past": False,
    "email_on_failure": False,
    "email_on_retry": False,
    "retries": 1,
    "retry_delay": duration(minutes=5),
}

with DAG(
    dag_id="trigger_dagrun_dag",
    start_date=datetime(2023, 1, 1),
    max_active_runs=1,
    schedule="@daily",
    default_args=default_args,
    catchup=False,
) as dag:
    start_task = PythonOperator(
        task_id="start_task",
        python_callable=print_task_type,
        op_kwargs={"task_type": "starting"},
    )
    trigger_dependent_dag = TriggerDagRunOperator(
        task_id="trigger_dependent_dag",
        trigger_dag_id="dependent_dag",
        wait_for_completion=True,
        deferrable=True,  # параметр доступен в Airflow 2.6+
    )
    end_task = PythonOperator(
        task_id="end_task",
        python_callable=print_task_type,
        op_kwargs={"task_type": "ending"},
    )
    start_task >> trigger_dependent_dag >> end_task
```

Если нижестоящему DAG нужен конфиг или конкретная логическая дата, их можно передать параметрами `conf` и `logical_date`. Параметр `skip_when_already_exists=True` не даёт оператору пытаться запустить уже существующий DAG run (и падать при повторных запусках). В Airflow 3.1 параметр `fail_when_dag_is_paused` позволяет завершать задачу с ошибкой, если зависимый DAG на паузе.

### ExternalTaskSensor

Чтобы задать зависимость **со стороны нижестоящего DAG**, используйте один или несколько [ExternalTaskSensor](https://registry.astronomer.io/providers/apache-airflow/modules/externaltasksensor). Нижестоящий DAG будет ждать завершения указанной задачи в вышестоящем DAG, после чего продолжит выполнение.

Такой способ удобен, когда у нижестоящего DAG несколько веток и каждая зависит от своей задачи в одном или нескольких вышестоящих DAG. Вместо зависимости целого DAG от целого DAG (как при ассетах) можно указать, что конкретная задача нижестоящего DAG ждёт завершения конкретной задачи вышестоящего.

Например: вышестоящие задачи обновляют разные таблицы в хранилище, а один нижестоящий DAG запускает по ветке проверок качества для каждой таблицы. В начале каждой ветки ставится ExternalTaskSensor, чтобы проверки по таблице стартовали только после обновления этой таблицы.

Рекомендуется использовать ExternalTaskSensor в deferrable-режиме (`deferrable=True`). Подробнее: [Deferrable-операторы](https://www.astronomer.io/docs/learn/deferrable-operators).

В примере ниже три ExternalTaskSensor в начале трёх параллельных веток одного DAG.

**TaskFlow:**

```python
from airflow.decorators import dag, task
from airflow.sensors.external_task import ExternalTaskSensor
from airflow.operators.empty import EmptyOperator
from pendulum import datetime, duration


@task
def downstream_function_branch_1():
    print("Upstream DAG 1 has completed. Starting tasks of branch 1.")


@task
def downstream_function_branch_2():
    print("Upstream DAG 2 has completed. Starting tasks of branch 2.")


@task
def downstream_function_branch_3():
    print("Upstream DAG 3 has completed. Starting tasks of branch 3.")


default_args = {
    "owner": "airflow",
    "depends_on_past": False,
    "email_on_failure": False,
    "email_on_retry": False,
    "retries": 1,
    "retry_delay": duration(minutes=5),
}


@dag(
    start_date=datetime(2022, 8, 1),
    max_active_runs=3,
    schedule="*/1 * * * *",
    catchup=False,
)
def external_task_sensor_taskflow_dag():
    start = EmptyOperator(task_id="start")
    end = EmptyOperator(task_id="end")

    ets_branch_1 = ExternalTaskSensor(
        task_id="ets_branch_1",
        external_dag_id="upstream_dag_1",
        external_task_id="my_task",
        allowed_states=["success"],
        failed_states=["failed", "skipped"],
    )
    task_branch_1 = downstream_function_branch_1()

    ets_branch_2 = ExternalTaskSensor(
        task_id="ets_branch_2",
        external_dag_id="upstream_dag_2",
        external_task_id="my_task",
        allowed_states=["success"],
        failed_states=["failed", "skipped"],
    )
    task_branch_2 = downstream_function_branch_2()

    ets_branch_3 = ExternalTaskSensor(
        task_id="ets_branch_3",
        external_dag_id="upstream_dag_3",
        external_task_id="my_task",
        allowed_states=["success"],
        failed_states=["failed", "skipped"],
    )
    task_branch_3 = downstream_function_branch_3()

    start >> [ets_branch_1, ets_branch_2, ets_branch_3]
    ets_branch_1 >> task_branch_1
    ets_branch_2 >> task_branch_2
    ets_branch_3 >> task_branch_3
    [task_branch_1, task_branch_2, task_branch_3] >> end


external_task_sensor_taskflow_dag()
```

**Традиционный вариант:**

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.sensors.external_task import ExternalTaskSensor
from airflow.operators.empty import EmptyOperator
from pendulum import datetime, duration


def downstream_function_branch_1():
    print("Upstream DAG 1 has completed. Starting tasks of branch 1.")


def downstream_function_branch_2():
    print("Upstream DAG 2 has completed. Starting tasks of branch 2.")


def downstream_function_branch_3():
    print("Upstream DAG 3 has completed. Starting tasks of branch 3.")


default_args = {
    "owner": "airflow",
    "depends_on_past": False,
    "email_on_failure": False,
    "email_on_retry": False,
    "retries": 1,
    "retry_delay": duration(minutes=5),
}

with DAG(
    "external-task-sensor-dag",
    start_date=datetime(2022, 8, 1),
    max_active_runs=3,
    schedule="*/1 * * * *",
    catchup=False,
) as dag:
    start = EmptyOperator(task_id="start")
    end = EmptyOperator(task_id="end")

    ets_branch_1 = ExternalTaskSensor(
        task_id="ets_branch_1",
        external_dag_id="upstream_dag_1",
        external_task_id="my_task",
        allowed_states=["success"],
        failed_states=["failed", "skipped"],
    )
    task_branch_1 = PythonOperator(
        task_id="task_branch_1",
        python_callable=downstream_function_branch_1,
    )

    ets_branch_2 = ExternalTaskSensor(
        task_id="ets_branch_2",
        external_dag_id="upstream_dag_2",
        external_task_id="my_task",
        allowed_states=["success"],
        failed_states=["failed", "skipped"],
    )
    task_branch_2 = PythonOperator(
        task_id="task_branch_2",
        python_callable=downstream_function_branch_2,
    )

    ets_branch_3 = ExternalTaskSensor(
        task_id="ets_branch_3",
        external_dag_id="upstream_dag_3",
        external_task_id="my_task",
        allowed_states=["success"],
        failed_states=["failed", "skipped"],
    )
    task_branch_3 = PythonOperator(
        task_id="task_branch_3",
        python_callable=downstream_function_branch_3,
    )

    start >> [ets_branch_1, ets_branch_2, ets_branch_3]
    ets_branch_1 >> task_branch_1
    ets_branch_2 >> task_branch_2
    ets_branch_3 >> task_branch_3
    [task_branch_1, task_branch_2, task_branch_3] >> end
```

В этом DAG:

- `ets_branch_3` ждёт завершения задачи `my_task` в `upstream_dag_3`, затем выполняется `task_branch_3`.
- `ets_branch_2` ждёт завершения задачи `my_task` в `upstream_dag_2`, затем выполняется `task_branch_2`.
- `ets_branch_1` ждёт завершения задачи `my_task` в `upstream_dag_1`, затем выполняется `task_branch_1`.

Эти ожидания идут параллельно и не зависят друг от друга. В графе видно состояние после завершения `my_task` в `upstream_dag_1`: выполнились `ets_branch_1` и `task_branch_1`; `ets_branch_2` и `ets_branch_3` ещё ждут своих вышестоящих задач.

Чтобы нижестоящий DAG ждал завершения **всего** вышестоящего DAG, а не отдельной задачи, задайте `external_task_id=None`. В примере выше для срабатывания сенсора внешняя задача должна быть в состоянии `success` (параметры `allowed_states` и `failed_states`).

В примере вышестоящий DAG (`example_dag`) и нижестоящий (`external-task-sensor-dag`) должны иметь одинаковые `start_date` и интервал расписания: ExternalTaskSensor ищет завершение указанной задачи или DAG с тем же `logical_date`. Чтобы ждать завершения внешней задачи с другой датой, используйте параметры `execution_delta` или `execution_date_fn` (подробнее в документации по ссылке выше).

### REST API Airflow

[REST API Airflow](https://airflow.apache.org/docs/apache-airflow/stable/stable-rest-api-ref.html) тоже позволяет задавать зависимости между DAG. Чтобы запустить DAG run через API, отправьте POST-запрос на эндпоинт **DAGRuns**.

Пример скрипта запуска DAG через API (для cross-DAG зависимостей этот код можно выполнять внутри задачи с декоратором `@task` в вышестоящем DAG):

```python
import requests

# Замените на параметры вашего инстанса Airflow
USERNAME = "admin"
PASSWORD = "admin"
HOST = "http://localhost:8080/"

MY_DAG = "example_dag"  # id DAG, который нужно запустить


def get_jwt_token():
    token_url = f"{HOST}/auth/token"
    payload = {"username": USERNAME, "password": PASSWORD}
    headers = {"Content-Type": "application/json"}
    response = requests.post(token_url, json=payload, headers=headers)
    token = response.json().get("access_token")
    return token


def run_dag(dag_id, logical_date=None):
    event_payload = {"conf": {"param1": "Hello World"}, "logical_date": logical_date}
    token = get_jwt_token()

    if token:
        url = f"{HOST}/api/v2/dags/{dag_id}/dagRuns"
        headers = {"Authorization": f"Bearer {token}"}
        response = requests.post(url, json=event_payload, headers=headers)
        print(response.status_code)
        print(response.json())
    else:
        raise Exception("Failed to get JWT token")


if __name__ == "__main__":
    run_dag(dag_id=MY_DAG, logical_date=None)
```

Обновить ассет через API можно POST-запросом на эндпоинт **Assets**.

## Зависимости между развёртываниями

Чтобы реализовать зависимости между DAG в двух разных окружениях Airflow на Astro, см. [Cross-deployment dependencies](https://www.astronomer.io/docs/astro/best-practices/cross-deployment-dependencies).

---

[← К содержанию](README.md) | [Ассеты →](../01.%20astronomer-basic/assets.md) | [Сенсоры →](../01.%20astronomer-basic/sensors.md)
