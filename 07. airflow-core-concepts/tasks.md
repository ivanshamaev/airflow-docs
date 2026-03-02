# Tasks (задачи)

**Task** — базовая единица выполнения в Airflow. Задачи объединяются в [DAG](dags.md), между ними задаются зависимости upstream и downstream, определяющие порядок выполнения.

Существует три основных типа задач:

- **[Operators](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/operators.html)** — готовые шаблоны задач, из которых быстро собирают большую часть DAG.
- **[Sensors](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/sensors.html)** — особый подкласс операторов, которые только ждут наступления внешнего события.
- Задекорированная **[TaskFlow](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/taskflow.html)** функция `@task` — произвольная Python-функция, оформленная как задача.

Внутри все они являются подклассами `BaseOperator` в Airflow; понятия Task и Operator во многом совпадают, но удобно различать: операторы и сенсоры — это шаблоны, а при вызове в файле DAG вы создаёте задачу (Task).

## Связи между задачами

Главное при работе с задачами — задать их взаимные зависимости (в Airflow говорят об upstream- и downstream-задачах). Сначала объявляют задачи, затем — зависимости между ними.

> **Примечание.** Upstream-задачей называют задачу, которая **непосредственно** предшествует другой. Раньше использовали термин «родительская». Это не про иерархию «выше по графу» — только про прямого предшественника. То же для downstream: это непосредственный потомок задачи.

Зависимости задают двумя способами — операторами `>>` и `<<` (битовый сдвиг):

```python
first_task >> second_task >> [third_task, fourth_task]
```

Либо явными методами `set_upstream` и `set_downstream`:

```python
first_task.set_downstream(second_task)
third_task.set_upstream(second_task)
```

Оба варианта делают одно и то же; обычно рекомендуют операторы `>>`/`<<` — так нагляднее.

По умолчанию задача запускается, когда все её upstream (родительские) задачи завершились успешно. Это поведение можно менять: добавлять ветвление, ждать только часть upstream или менять логику в зависимости от места запуска в истории. Подробнее: [Control Flow](dags.md#управление-потоком-control-flow).

По умолчанию задачи не передают друг другу данные и выполняются независимо. Чтобы передавать данные между задачами, используйте [XComs](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/xcoms.html).

## Task Instances (экземпляры задач)

Так же, как DAG при каждом запуске превращается в [Dag Run](dag-run.md), каждая задача внутри DAG при запуске превращается в **Task Instance** (экземпляр задачи).

Экземпляр задачи — это один запуск этой задачи для данного DAG (и тем самым для данного data interval). У него есть состояние, показывающее, на каком этапе жизненного цикла он находится.

Возможные состояния Task Instance:

| Состояние | Описание |
|-----------|----------|
| `none` | Задача ещё не поставлена в очередь (зависимости не выполнены) |
| `scheduled` | Планировщик решил, что зависимости выполнены и задачу нужно запустить |
| `queued` | Задача назначена исполнителю и ждёт воркера |
| `running` | Задача выполняется на воркере (или локальном/синхронном executor) |
| `success` | Задача завершилась без ошибок |
| `restarting` | Задача выполнялась и по внешнему запросу перезапускается |
| `failed` | При выполнении произошла ошибка |
| `skipped` | Задача пропущена из-за ветвления, LatestOnly и т.п. |
| `upstream_failed` | Вышестоящая задача упала, и [trigger rule](dags.md#trigger-rules) требовал её успеха |
| `up_for_retry` | Задача упала, но остались попытки — будет перезапланирована |
| `up_for_reschedule` | Задача — [Sensor](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/sensors.html) в режиме `reschedule` |
| `deferred` | Задача [отложена на триггер](https://airflow.apache.org/docs/apache-airflow/stable/authoring-and-scheduling/deferring.html) |
| `removed` | Задача исчезла из DAG после начала запуска |

В норме задача переходит: `none` → `scheduled` → `queued` → `running` → `success`.

При выполнении любой кастомной задачи (оператора) ей передаётся копия экземпляра задачи; по нему можно читать метаданные и вызывать методы, в том числе для работы с [XComs](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/xcoms.html).

### Терминология связей

У каждого Task Instance есть два типа связей с другими экземплярами.

**Первый** — upstream и downstream задачи:

```python
task1 >> task2 >> task3
```

При запуске DAG создаются экземпляры этих задач, связанные как upstream/downstream и имеющие один и тот же data interval.

**Второй** — экземпляры той же задачи, но для других data interval (из других запусков того же DAG). Их называют previous и next — это другая связь, не upstream/downstream.

> **Примечание.** В старой документации Airflow «previous» иногда означало «upstream». Если встретите такое, можно сообщить об этом в проект.

## Таймауты

Чтобы ограничить максимальное время выполнения задачи, задайте атрибут `execution_timeout` значением `datetime.timedelta`. Это относится ко всем задачам Airflow, включая сенсоры. `execution_timeout` ограничивает время **каждого** запуска. При превышении задача завершается по таймауту и выбрасывается `AirflowTaskTimeout`.

У сенсоров есть ещё параметр `timeout`. Он важен только для сенсоров в режиме `reschedule`. `timeout` ограничивает суммарное время, в течение которого сенсор должен успешно завершиться. При превышении выбрасывается `AirflowSensorTimeout`, сенсор сразу помечается как failed без повторных попыток.

Пример с `SFTPSensor` в режиме `reschedule` (периодический запуск и перепланирование до успеха):

- Каждый «poke» к SFTP-серверу ограничен 60 секундами (`execution_timeout`).
- Если один poke длится больше 60 секунд, выбрасывается `AirflowTaskTimeout`; сенсор может делать retry (например, до 2 раз по `retries`).
- С момента первого запуска до успеха (например, появления файла `root/test`) сенсору отводится не более 3600 секунд (`timeout`). Если за 3600 секунд файл не появится, будет `AirflowSensorTimeout` — без retry.
- Если сенсор падает по другим причинам (сеть и т.д.) в течение этих 3600 секунд, он может делать до 2 retry по `retries`. Таймер `timeout` при retry не сбрасывается — всего по-прежнему не более 3600 секунд на успех.

```python
sensor = SFTPSensor(
    task_id="sensor",
    path="/root/test",
    execution_timeout=timedelta(seconds=60),
    timeout=3600,
    retries=2,
    mode="reschedule",
)
```

## SLA

Функция SLA из Airflow 2 удалена в 3.0 и в Airflow 3.1 заменена на [Deadline Alerts](https://airflow.apache.org/docs/apache-airflow/stable/howto/deadline-alerts.html).

## Специальные исключения

Чтобы изменить состояние задачи из кода кастомного оператора, в Airflow есть два исключения:

- **`AirflowSkipException`** — помечает текущую задачу как skipped.
- **`AirflowFailException`** — помечает текущую задачу как failed и отменяет оставшиеся попытки retry.

Их используют, когда код «знает» контекст и хочет быстрее завершить задачу как failed или skipped — например, пропуск при отсутствии данных или немедленный fail при невалидном API-ключе (retry не поможет).

## Task Instance Heartbeat Timeout

Экземпляры задач иногда «зависают»: остаются в состоянии `running`, хотя воркер уже не работает (например, процесс убит из-за нехватки памяти). Такие задачи раньше называли zombie. Airflow периодически их находит, сбрасывает и помечает Task Instance как failed или перезапускает при наличии retry. Таймаут heartbeat может наступить по разным причинам:

- Воркер Airflow убит из-за нехватки памяти (OOMKilled).
- Воркер не прошёл liveness probe, и система (например, Kubernetes) перезапустила его.
- Кластер уменьшили или перенесли воркер на другой узел.

### Воспроизведение таймаута heartbeat локально

Чтобы воспроизвести таймаут heartbeat в разработке или тестах:

1. Задайте переменные окружения (или соответствующие параметры в airflow.cfg):

```bash
export AIRFLOW__SCHEDULER__TASK_INSTANCE_HEARTBEAT_SEC=600
export AIRFLOW__SCHEDULER__TASK_INSTANCE_HEARTBEAT_TIMEOUT=2
export AIRFLOW__SCHEDULER__TASK_INSTANCE_HEARTBEAT_TIMEOUT_DETECTION_INTERVAL=5
```

2. Используйте DAG с задачей, которая выполняется около 10 минут. Пример:

```python
from airflow.sdk import dag
from airflow.providers.standard.operators.bash import BashOperator
from datetime import datetime


@dag(start_date=datetime(2021, 1, 1), schedule="@once", catchup=False)
def sleep_dag():
    t1 = BashOperator(
        task_id="sleep_10_minutes",
        bash_command="sleep 600",
    )


sleep_dag()
```

3. Запустите DAG и подождите. Через несколько секунд Task Instance будет помечен как failed.

## Конфигурация Executor

Некоторые [executors](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/executor/index.html) позволяют задавать настройки для отдельной задачи — например, `KubernetesExecutor` даёт указать образ для запуска задачи.

Для этого у задачи (оператора) используется аргумент `executor_config`. Пример задания Docker-образа для задачи на `KubernetesExecutor`:

```python
MyOperator(...,
    executor_config={
        "KubernetesExecutor":
            {"image": "myCustomDockerImage"}
    }
)
```

Допустимые значения `executor_config` зависят от executor; см. [документацию по каждому executor](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/executor/index.html).

---

*Источник: [Airflow 3.1.7 — Tasks](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/tasks.html). Перевод неофициальный.*
