# DAG (Dag)

**Dag** — это модель, в которой собрано всё необходимое для выполнения рабочего процесса. Основные атрибуты DAG:

- **Расписание (Schedule):** когда должен запускаться процесс.
- **Задачи (Tasks):** [задачи](tasks.md) — отдельные единицы работы, выполняемые на воркерах.
- **Зависимости задач:** порядок и условия выполнения [задач](tasks.md).
- **Обратные вызовы (Callbacks):** действия по завершении всего процесса.
- **Дополнительные параметры** и другие эксплуатационные детали.

Пример простого DAG: четыре задачи (A, B, C, D), порядок выполнения и зависимости между ними, а также расписание — например, «каждые 5 минут с завтрашнего дня» или «ежедневно с 1 января 2020». Сам DAG не задаёт, что делают задачи; он определяет **как** их выполнять — порядок, повторы, таймауты и т.д.

> **Примечание.** Термин «DAG» восходит к «ориентированный ациклический граф», но в Airflow он используется в более широком смысле. В документации принято написание **Dag**.

## Объявление DAG

DAG можно объявить тремя способами.

**1. Контекстный менеджер `with`** — всё внутри блока автоматически относится к DAG:

```python
import datetime

from airflow.sdk import DAG
from airflow.providers.standard.operators.empty import EmptyOperator

with DAG(
    dag_id="my_dag_name",
    start_date=datetime.datetime(2021, 1, 1),
    schedule="@daily",
):
    EmptyOperator(task_id="task")
```

**2. Обычный конструктор** — DAG передаётся в операторы явно:

```python
import datetime

from airflow.sdk import DAG
from airflow.providers.standard.operators.empty import EmptyOperator

my_dag = DAG(
    dag_id="my_dag_name",
    start_date=datetime.datetime(2021, 1, 1),
    schedule="@daily",
)
EmptyOperator(task_id="task", dag=my_dag)
```

**3. Декоратор `@dag`** — функция превращается в генератор DAG:

```python
import datetime

from airflow.sdk import dag
from airflow.providers.standard.operators.empty import EmptyOperator


@dag(start_date=datetime.datetime(2021, 1, 1), schedule="@daily")
def generate_dag():
    EmptyOperator(task_id="task")


generate_dag()
```

Внутри DAG размещаются [задачи](tasks.md) в виде операторов, сенсоров или [TaskFlow](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/taskflow.html).

### Зависимости между задачами

Задачи связаны зависимостями: у каждой есть вышестоящие (upstream) и нижестоящие (downstream) задачи. Рекомендуемый способ — операторы `>>` и `<<`:

```python
first_task >> [second_task, third_task]
third_task << fourth_task
```

Либо явные методы `set_upstream` / `set_downstream`:

```python
first_task.set_downstream([second_task, third_task])
third_task.set_upstream(fourth_task)
```

Для сложных зависимостей: `cross_downstream` — список задач от одного списка к другому; `chain` — цепочка или попарные зависимости:

```python
from airflow.sdk import cross_downstream, chain

# [op1, op2] -> op3, op4
cross_downstream([op1, op2], [op3, op4])

# op1 >> op2 >> op3 >> op4
chain(op1, op2, op3, op4)
```

## Загрузка DAG

Airflow загружает DAG из Python-файлов в каталоге (или бандле) DAG. Файл выполняется, из него извлекаются объекты DAG. В одном файле может быть несколько DAG, один сложный DAG можно разнести по нескольким файлам через импорты.

Важно: загружаются только объекты DAG на верхнем уровне (в `globals()`). DAG, созданный внутри функции, не будет подхвачен.

По умолчанию Airflow обрабатывает только те Python-файлы, в которых встречаются подстроки `airflow` и `dag` (без учёта регистра). Чтобы учитывать все файлы, отключите конфиг `DAG_DISCOVERY_SAFE_MODE`. Игнорировать файлы можно через `.airflowignore` в каталоге DAG.

## Запуск DAG

DAG запускается:

- вручную или через API;
- по расписанию, заданному в DAG (аргумент `schedule`).

Расписание не обязательно, но обычно его задают:

```python
with DAG("my_daily_dag", schedule="@daily"):
    ...

with DAG("my_daily_dag", schedule="0 0 * * *"):  # cron
    ...

with DAG("my_one_time_dag", schedule="@once"):
    ...

with DAG("my_continuous_dag", schedule="@continuous"):
    ...
```

Каждый запуск DAG создаёт **Dag Run** — экземпляр DAG с заданным интервалом данных (data interval). Несколько Dag Run одного DAG могут выполняться параллельно; при backfill создаётся по одному запуску на каждый интервал (например, на каждый день за три месяца).

У Dag Run есть даты начала и окончания и **логическая дата** (logical date, раньше — execution date) — момент, на который запланирован или запущен этот запуск.

Задачи внутри DAG при каждом запуске превращаются в [экземпляры задач (Task Instances)](tasks.md#экземпляры-задач-task-instances).

## Присвоение DAG задаче

Каждый оператор/задача должен быть привязан к DAG. DAG определяется автоматически, если:

- оператор объявлен внутри `with DAG(...)` или внутри функции с декоратором `@dag`;
- оператор связан зависимостью с задачей, у которой уже задан DAG.

Иначе DAG нужно передавать явно: `dag=my_dag`.

## Аргументы по умолчанию

Через `default_args` в DAG можно задать общие аргументы для всех операторов в этом DAG (например, `retries`, `email_on_failure`). Операторы могут переопределять эти значения.

---

[← Задачи](tasks.md) | [К основным концепциям](README.md) | [К содержанию](../toc.md)
