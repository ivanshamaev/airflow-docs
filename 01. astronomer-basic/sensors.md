# Что такое сенсор (Sensor)?

[Сенсоры Apache Airflow](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/sensors.html) — это особая разновидность операторов, которые ждут наступления некоторого условия. При запуске сенсор проверяет, выполнено ли условие; когда оно выполнено, задача помечается успешной и нижестоящие задачи могут выполниться.

В этом руководстве вы узнаете, как используются сенсоры в Airflow, лучшие практики для продакшена и как применять отложенные (deferrable) версии сенсоров.

> **Совет.** Сенсоры используются, чтобы дождаться выполнения условия перед запуском нижестоящих задач. У многих сенсоров есть deferrable-режим: они освобождают слот воркера на время ожидания и повышают эффективность DAG. Подробнее: [Deferrable-операторы](https://www.astronomer.io/docs/learn/deferrable-operators).
>
> Если DAG должен запускаться по сообщениям в очереди, рассмотрите [event-driven планирование](https://www.astronomer.io/docs/learn/airflow-event-driven-scheduling) вместо сенсоров.

## Необходимая база

Чтобы получить максимум от руководства, нужно понимать:

- Основы Python. См. [документацию Python](https://docs.python.org/3/tutorial/index.html).
- Основные концепции Airflow. См. [Введение в Apache Airflow](https://www.astronomer.io/docs/learn/intro-to-airflow).

## Основы сенсоров

Сенсор — это тип оператора, который с заданным интервалом проверяет выполнение условия. Если условие выполнено, задача помечается успешной и DAG переходит к нижестоящим задачам. Если нет — сенсор ждёт следующий интервал и проверяет снова.

Все сенсоры наследуются от [`BaseSensorOperator`](https://github.com/apache/airflow/blob/main/task-sdk/src/airflow/sdk/bases/sensor.py) и имеют следующие параметры:

- **soft_fail**: при `True` задача помечается как пропущенная (skipped), если условие не выполнено к моменту `timeout`.
- **timeout**: максимальное время в секундах, в течение которого сенсор проверяет условие. Если за это время условие не выполнено, задача завершается с ошибкой.
- **exponential_backoff**: при `True` в режиме `poke` интервалы между проверками увеличиваются по экспоненте.
- **poke_interval**: в режиме `poke` — интервал в секундах между проверками. По умолчанию 60 секунд.
- **mode**: режим работы сенсора. Два варианта:
  - **poke** (по умолчанию): сенсор занимает слот воркера всё время выполнения и «дремлет» между проверками. Подходит, если ожидается короткое время работы сенсора.
  - **reschedule**: если условие не выполнено, сенсор освобождает слот воркера и планирует следующую проверку на более позднее время. Подходит при длительном ожидании: меньше нагружает ресурсы и освобождает воркеры для других задач.

У разных типов сенсоров свои детали реализации.

### Часто используемые сенсоры

Во многих пакетах провайдеров Airflow есть сенсоры для ожидания разных условий в разных системах. Ниже — некоторые из самых распространённых:

- [**SqlSensor**](https://registry.astronomer.io/providers/apache-airflow-providers-common-sql/modules/sqlsensor): ждёт появления данных в SQL-таблице. Удобен, когда DAG должен обрабатывать данные по мере поступления в БД.
- [**HttpSensor**](https://registry.astronomer.io/providers/http/modules/httpsensor): ждёт доступности API. Удобен, чтобы убедиться, что запросы к API будут успешными.
- [**ExternalTaskSensor**](https://registry.astronomer.io/providers/apache-airflow/modules/externaltasksensor): ждёт завершения задачи в Airflow. Удобен для [зависимостей между DAG](https://www.astronomer.io/docs/learn/cross-dag-dependencies) в одном окружении Airflow.
- [**DateTimeSensor**](https://registry.astronomer.io/providers/apache-airflow/modules/datetimesensor): ждёт наступления указанной даты и времени. Удобен, когда разные задачи одного DAG должны запускаться в разное время.
- [**S3KeySensor**](https://registry.astronomer.io/providers/amazon/modules/s3keysensor): ждёт появления ключа (файла) в бакете Amazon S3. Удобен, когда DAG обрабатывает файлы из S3 по мере появления.
- [**Декоратор @task.sensor**](https://airflow.apache.org/docs/apache-airflow/stable/tutorial/taskflow.html#using-the-taskflow-api-with-sensor-operators): превращает любую Python-функцию, возвращающую `PokeReturnValue`, в экземпляр [BaseSensorOperator](https://registry.astronomer.io/providers/apache-airflow/modules/basesensoroperator). Удобен при сложной логике проверки или при работе с API, для которого нет отдельного сенсора.

Полный список сенсоров: [Astronomer Registry](https://registry.astronomer.io/modules?types=sensors).

### Пример использования

В следующем примере DAG использует сенсор `SqlSensor`:

```python
from airflow.decorators import task, dag
from airflow.providers.common.sql.sensors.sql import SqlSensor

from typing import Dict
from pendulum import datetime


def _success_criteria(record):
    return record


def _failure_criteria(record):
    return True if not record else False


@dag(
    description="DAG in charge of processing partner data",
    start_date=datetime(2021, 1, 1),
    schedule="@daily",
    catchup=False,
)
def partner():
    waiting_for_partner = SqlSensor(
        task_id="waiting_for_partner",
        conn_id="postgres",
        sql="sql/CHECK_PARTNER.sql",
        parameters={"name": "partner_a"},
        success=_success_criteria,
        failure=_failure_criteria,
        fail_on_empty=False,
        poke_interval=20,
        mode="reschedule",
        timeout=60 * 5,
    )

    @task
    def validation() -> Dict[str, str]:
        return {"partner_name": "partner_a", "partner_validation": True}

    @task
    def storing():
        print("storing")

    waiting_for_partner >> validation() >> storing()


partner()
```

Вариант с контекстным менеджером DAG:

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.providers.common.sql.sensors.sql import SqlSensor

from typing import Dict
from pendulum import datetime


def _success_criteria(record):
    return record


def _failure_criteria(record):
    return True if not record else False


with DAG(
    dag_id="partner",
    description="DAG in charge of processing partner data",
    start_date=datetime(2021, 1, 1),
    schedule="@daily",
    catchup=False,
):
    waiting_for_partner = SqlSensor(
        task_id="waiting_for_partner",
        conn_id="postgres",
        sql="sql/CHECK_PARTNER.sql",
        parameters={"name": "partner_a"},
        success=_success_criteria,
        failure=_failure_criteria,
        fail_on_empty=False,
        poke_interval=20,
        mode="reschedule",
        timeout=60 * 5,
    )

    def validation_function() -> Dict[str, str]:
        return {"partner_name": "partner_a", "partner_validation": True}

    validation = PythonOperator(
        task_id="validation", python_callable=validation_function
    )

    def storing_function():
        print("storing")

    storing = PythonOperator(task_id="storing", python_callable=storing_function)

    waiting_for_partner >> validation >> storing
```

Этот DAG ждёт появления данных в базе Postgres, после чего запускает задачи validation и storing. `SqlSensor` выполняет SQL-запрос и помечается успешным, когда запрос возвращает данные (точнее, когда результат не в множестве (0, '0', '', None)). В примере сенсор `waiting_for_partner` запускает скрипт `CHECK_PARTNER.sql` каждые 20 секунд (`poke_interval`), пока данные не появятся. Режим `reschedule` означает, что между проверками задача не занимает слот воркера. `timeout` установлен в 5 минут — при отсутствии данных за это время задача завершится с ошибкой. Когда условие `SqlSensor` выполнено, DAG переходит к нижестоящим задачам.

## Декоратор сенсора и PythonSensor

Если под вашу задачу нет готового сенсора, можно создать свой с помощью декоратора `@task.sensor` или [PythonSensor](https://registry.astronomer.io/providers/apache-airflow/versions/latest/modules/PythonSensor). Декоратор `@task.sensor` возвращает `PokeReturnValue` и даёт экземпляр BaseSensorOperator. PythonSensor принимает `python_callable`, которая возвращает `True` или `False`.

В следующем DAG показано, как с помощью декоратора или PythonSensor создать один и тот же кастомный сенсор.

**Кастомный сенсор через @task.sensor:**

```python
"""
Кастомный сенсор с декоратором @task.sensor.
Проверяет доступность API.
"""

from airflow.decorators import dag, task
from pendulum import datetime
import requests

from airflow.sensors.base import PokeReturnValue


@dag(start_date=datetime(2022, 12, 1), schedule="@daily", catchup=False)
def sensor_decorator():
    @task.sensor(poke_interval=30, timeout=3600, mode="poke")
    def check_dog_availability() -> PokeReturnValue:
        r = requests.get("https://random.dog/woof.json")
        print(r.status_code)

        if r.status_code == 200:
            condition_met = True
            operator_return_value = r.json()
        else:
            condition_met = False
            operator_return_value = None
            print(f"Woof URL returned the status code {r.status_code}")

        # Функция должна возвращать PokeReturnValue.
        # is_done=True — сенсор завершается успешно;
        # is_done=False — сенсор повторит проверку (poke или reschedule).
        return PokeReturnValue(is_done=condition_met, xcom_value=operator_return_value)

    @task
    def print_dog_picture_url(url):
        print(url)

    print_dog_picture_url(check_dog_availability())


sensor_decorator()
```

Здесь `@task.sensor` применяется к функции `check_dog_availability()`, которая проверяет, возвращает ли API код 200. При 200 задача сенсора помечается успешной. При любом другом коде сенсор повторяет проверку через `poke_interval`.

Необязательный параметр `xcom_value` в `PokeReturnValue` задаёт данные, которые будут записаны в [XCom](https://www.astronomer.io/docs/learn/airflow-passing-data-between-tasks) при `is_done=True`. Эти данные можно использовать в любых нижестоящих задачах.

**Тот же сенсор через PythonSensor:**

```python
"""
Кастомный сенсор с PythonSensor.
Проверяет доступность API.
"""

from airflow.decorators import dag, task
from pendulum import datetime
import requests
from airflow.sensors.python import PythonSensor


def check_dog_availability_func(**context):
    r = requests.get("https://random.dog/woof.json")
    print(r.status_code)

    if r.status_code == 200:
        operator_return_value = r.json()
        context["ti"].xcom_push(key="return_value", value=operator_return_value)
        return True
    else:
        operator_return_value = None
        print(f"Woof URL returned the status code {r.status_code}")
        return False


@dag(
    start_date=datetime(2022, 12, 1),
    schedule=None,
    catchup=False,
    tags=["sensor"],
)
def pythonsensor_example():
    check_dog_availability = PythonSensor(
        task_id="check_dog_availability",
        poke_interval=10,
        timeout=3600,
        mode="reschedule",
        python_callable=check_dog_availability_func,
    )

    @task
    def print_dog_picture_url(url):
        print(url)

    print_dog_picture_url(check_dog_availability.output)


pythonsensor_example()
```

Здесь PythonSensor использует `check_dog_availability_func` для проверки кода ответа API. При 200 ответ записывается в [XCom](https://www.astronomer.io/docs/learn/airflow-passing-data-between-tasks), функция возвращает `True`, и задача сенсора помечается успешной. При другом коде функция возвращает `False`, и сенсор повторяет проверку через `poke_interval`.

## Рекомендации по сенсорам

При использовании сенсоров учитывайте следующее, чтобы избежать проблем с производительностью:

- Задавайте осмысленный **poke_interval** под вашу задачу. Нет смысла проверять условие каждые 60 секунд (значение по умолчанию), если общее время ожидания известно и составляет, например, 30 минут.
- Если **poke_interval** очень мал (меньше примерно 5 минут), используйте режим **poke**. В этом случае **reschedule** может перегрузить планировщик.
- По возможности, особенно для долго работающих сенсоров, используйте режим **deferrable**. Если он недоступен — режим **reschedule**. Оба варианта не занимают слот воркера постоянно и помогают избежать ситуаций, когда сенсоры забирают все свободные слоты.
- Всегда задавайте осмысленный параметр **timeout**. По умолчанию он может быть до семи дней — для большинства сценариев это слишком долго. Оцените, сколько времени сенсор может ждать, и задайте timeout соответственно.

## Режимы при сбоях сенсора

Поведение сенсора при исключении в методе poke можно настроить так:

- **never_fail=True**: при любом исключении в poke задача сенсора помечается как пропущенная (skipped). Несовместим с `soft_fail`.
- **silent_fail=True**: при исключении в poke, которое не является AirflowSensorTimeout, AirflowTaskTimeout, AirflowSkipException или AirflowFailException, ошибка логируется, но сенсор продолжает выполнение.
- **soft_fail=True**: при исключении в задаче она помечается как пропущенная; влияние на нижестоящие задачи определяется их [trigger rules](trigger-rules.md).

## Deferrable-операторы

[Deferrable-операторы](https://www.astronomer.io/docs/learn/deferrable-operators) (иногда их называют асинхронными) решают проблему постоянной занятости слота воркера на всё время работы оператора или сенсора. У многих операторов есть параметр **deferrable**, который можно установить в `True`. Для сенсоров без такого параметра существуют отдельные deferrable-версии в open source Airflow и в [Astronomer Providers](https://github.com/astronomer/astronomer-providers). Astronomer рекомендует по возможности использовать их, чтобы снизить нагрузку на ресурсы.

Для авторов DAG использование deferrable-сенсоров не отличается от обычных. Нужно только запустить процесс **triggerer** в Airflow и:

- заменить сенсор на его deferrable-аналог, если параметра `deferrable` нет; либо
- установить **deferrable=True** для нужных экземпляров сенсоров; либо
- включить конфиг Airflow `operators.default_deferrable=True`, чтобы все сенсоры с поддержкой deferrable работали в этом режиме по умолчанию.

Подробнее: [Deferrable-операторы](https://www.astronomer.io/docs/learn/deferrable-operators).

---

[← Операторы](operators.md) | [К содержанию](README.md) | [Зависимости задач →](task-dependencies.md)
