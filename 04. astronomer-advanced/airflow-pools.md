# Пуллы Airflow (Pools)

Одно из преимуществ Apache Airflow — заложенная масштабируемость. При подходящей инфраструктуре можно без проблем запускать много задач параллельно. При этом горизонтальное масштабирование требует ограничений. Например, множество задач могут обращаться к одной и той же системе — API или базе данных — и перегружать её запросами. [Пуллы (pools)](https://airflow.apache.org/docs/apache-airflow/stable/administration-and-deployment/pools.html) в Airflow как раз для таких сценариев.

Пуллы позволяют ограничивать параллелизм для произвольного набора задач и управлять тем, когда задачи выполняются. Их часто используют, когда нужно ограничить число параллельных задач определённого типа: обращающихся к одному API или БД или выполняющихся на GPU-ноде кластера Kubernetes.

В этом руководстве — базовые концепции пуллов Airflow, создание и назначение пуллов, возможности и ограничения. Также приведены примеры DAG с пуллами для типовых задач.

## Необходимая база

Чтобы получить максимум от руководства, нужно понимать:

- Основы масштабирования Airflow. См. [Scaling out Airflow](../03.%20astronomer-infra/scaling-airflow.md).
- Операторы Airflow. См. [Operators 101](../01.%20astronomer-basic/operators.md).

## Создание пула

Создавать и управлять пуллами в Airflow можно тремя способами:

- **REST API Airflow:** чтобы создать пул, отправьте POST-запрос с именем и числом слотов в теле. Подробнее о работе с пуллами через API: [документация API](https://airflow.apache.org/docs/apache-airflow/stable/stable-rest-api-ref.html#operation/post_pool).
- **CLI Airflow:** выполните команду `airflow pools` с подкомандой `set` для создания пула. Полный список команд для пуллов — в [документации CLI Airflow](https://airflow.apache.org/docs/apache-airflow/stable/cli-and-env-variables-ref.html#pools). Через CLI пуллы можно импортировать из JSON-файла подкомандой `import` — удобно при большом числе пуллов и программном задании.
- **Веб-интерфейс Airflow:** перейдите в Admin → Pools и добавьте запись. Можно задать имя, число слотов и описание.

![Пуллы в Airflow UI](images/3-0_pools_ui.png)

## Назначение задач пулу

По умолчанию все задачи в Airflow попадают в пул `default_pool` с 128 слотами. Число слотов можно изменить, но пул по умолчанию удалить нельзя. Назначить задачу другому пулу можно через параметр `pool`. Он входит в `BaseOperator`, поэтому доступен для любого оператора.

**TaskFlow:**

```python
@task(task_id="task_a", pool="my_pool")
def sleep_function():
    ...
```

**Традиционный вариант:**

```python
task_a = PythonOperator(
    task_id="task_a", python_callable=sleep_function, pool="my_pool"
)
```

При назначении задач пулу они планируются как обычно, пока не заполнятся все слоты пула. По мере освобождения слотов запускаются следующие задачи из очереди.

Если назначить задачу несуществующему пулу, при запуске DAG задача не будет запланирована. В UI Airflow ошибок или проверок для этого нет, поэтому стоит сверять имя пула при назначении.

Очередность выполнения задач в пуле задаётся [priority weights](https://airflow.apache.org/docs/apache-airflow/stable/administration-and-deployment/priority-weight.html). Они задаются на уровне пула параметром `priority_weights`. Большее значение — выше приоритет в очереди исполнителя. Можно задать свой [Custom Weight Rule](https://airflow.apache.org/docs/apache-airflow/stable/administration-and-deployment/priority-weight.html#custom-weight-rule).

В фрагменте DAG ниже `task_a` и `task_b` назначены пулу `single_task_pool` с одним слотом. У `task_b` priority weight 2, у `task_a` — значение по умолчанию 1. Поэтому первой выполнится `task_b`.

**TaskFlow:**

```python
@task
def sleep_function(x):
    time.sleep(x)

@dag
def pool_dag():
    sleep_function.override(task_id="task_a", pool="single_task_pool")(5)
    sleep_function.override(
        task_id="task_b", pool="single_task_pool", priority_weight=2
    )(10)

pool_dag()
```

**Традиционный вариант:**

```python
def sleep_function(x):
    time.sleep(x)

with DAG(
    dag_id="pool_dag",
) as dag:

    task_a = PythonOperator(
        task_id="task_a",
        python_callable=sleep_function,
        pool="single_task_pool",
        op_args=[5],
    )

    task_b = PythonOperator(
        task_id="task_b",
        python_callable=sleep_function,
        pool="single_task_pool",
        priority_weight=2,
        op_args=[10],
    )
```

Количество слотов, занимаемых задачей, настраивается параметром `pool_slots` (по умолчанию 1). Изменение полезно, когда пуллы используются для управления загрузкой ресурсов.

## Ограничения пуллов

При работе с пуллами учитывайте:

- Пуллы ограничивают параллелизм **экземпляров задач (task instances)**. Чтобы ограничить число одновременных DagRun для одного DAG или всех DAG, используйте параметры `max_active_runs` или `core.max_active_runs_per_dag`.
- Задача может быть назначена только одному пулу.

## Пример: ограничение задач, обращающихся к API

В этом примере пул ограничивает число задач, обращающихся к одному API.

Сценарий: пять задач из двух DAG обращаются к API и могут выполняться параллельно по расписанию DAG. Чтобы одновременно к API обращались не более трёх задач, создаётся пул `api_pool` с тремя слотами. Задачи из `pool_priority_dag` при заполнении пула должны иметь приоритет.

В DAG `pool_priority_dag` ниже все три задачи обращаются к API и должны быть в пуле, поэтому `pool` задаётся в `default_args` DAG для всех задач. Нужно также задать всем трём одинаковый priority weight и приоритет над задачами второго DAG — в `default_args` задаётся `priority_weight`: 3 (значение произвольное; для приоритета над вторым DAG достаточно любого целого числа больше, чем weight во втором DAG).

**TaskFlow:**

```python
from pendulum import datetime, duration

import requests
from airflow.decorators import dag, task


@task
def api_function(**kwargs):
    url = "http://catfact.ninja/fact"
    res = requests.get(url)
    return res.json()


@dag(
    start_date=datetime(2021, 8, 1),
    schedule="*/30 * * * *",
    catchup=False,
    default_args={
        "pool": "api_pool",
        "retries": 1,
        "retry_delay": duration(minutes=5),
        "priority_weight": 3,
    },
)
def pool_priority_dag():
    api_function.override(task_id="task_a")()

    api_function.override(task_id="task_b")()

    api_function.override(task_id="task_c")()


pool_priority_dag()
```

**Традиционный вариант:**

```python
from pendulum import datetime, duration

import requests
from airflow import DAG
from airflow.operators.python import PythonOperator


def api_function(**kwargs):
    url = "http://catfact.ninja/fact"
    res = requests.get(url)
    return res.json()


with DAG(
    "pool_priority_dag",
    start_date=datetime(2021, 8, 1),
    schedule="*/30 * * * *",
    catchup=False,
    default_args={
        "pool": "api_pool",
        "retries": 1,
        "retry_delay": duration(minutes=5),
        "priority_weight": 3,
    },
) as dag:
    task_a = PythonOperator(task_id="task_a", python_callable=api_function)

    task_b = PythonOperator(task_id="task_b", python_callable=api_function)

    task_c = PythonOperator(task_id="task_c", python_callable=api_function)
```

В DAG `pool_chill_dag` две задачи обращаются к API и должны быть в пуле, ещё две — нет. Поэтому пул и priority weight задаются только в нужных экземплярах `PythonOperator`.

Чтобы `task_x` имела приоритет над `task_y`, но обе были ниже задач первого DAG, для `task_x` задаётся priority weight 2, у `task_y` остаётся значение по умолчанию 1.

**TaskFlow:**

```python
from pendulum import datetime, duration
import requests
from airflow.decorators import dag, task
from airflow.operators.empty import EmptyOperator


@task
def api_function(**kwargs):
    url = "http://catfact.ninja/fact"
    res = requests.get(url)
    return res.json()


@dag(
    start_date=datetime(2023, 1, 1),
    schedule="*/30 * * * *",
    catchup=False,
    default_args={"retries": 1, "retry_delay": duration(minutes=5)},
)
def pool_unimportant_dag():
    task_w = EmptyOperator(task_id="start")

    task_x = api_function.override(
        task_id="task_x",
        pool="api_pool",
        priority_weight=2,
    )()

    task_y = api_function.override(task_id="task_y", pool="api_pool")()

    task_z = EmptyOperator(task_id="end")

    task_w >> [task_x, task_y] >> task_z


pool_unimportant_dag()
```

**Традиционный вариант:**

```python
from pendulum import datetime, duration
import requests
from airflow import DAG
from airflow.operators.empty import EmptyOperator
from airflow.operators.python import PythonOperator


def api_function(**kwargs):
    url = "http://catfact.ninja/fact"
    res = requests.get(url)
    return res.json()


with DAG(
    "pool_unimportant_dag",
    start_date=datetime(2023, 1, 1),
    schedule="*/30 * * * *",
    catchup=False,
    default_args={"retries": 1, "retry_delay": duration(minutes=5)},
) as dag:
    task_w = EmptyOperator(task_id="start")

    task_x = PythonOperator(
        task_id="task_x",
        python_callable=api_function,
        pool="api_pool",
        priority_weight=2,
    )

    task_y = PythonOperator(
        task_id="task_y", python_callable=api_function, pool="api_pool"
    )

    task_z = EmptyOperator(task_id="end")

    task_w >> [task_x, task_y] >> task_z
```

---

[← Плагины](airflow-plugins.md) | [К содержанию](README.md) | [Custom XCom →](custom-xcom-backends.md)
