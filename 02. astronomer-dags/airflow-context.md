# Контекст Airflow (Airflow context)

**Контекст Airflow** — это словарь с информацией о выполняющемся DAG и окружении Airflow, к которому можно обращаться из задачи. Один из самых частых элементов контекста — ключ **ti** / **task_instance** (см. раздел ниже), дающий доступ к атрибутам и методам [объекта TaskInstance](https://airflow.apache.org/docs/apache-airflow/stable/_api/airflow/models/taskinstance/index.html).

Другие типичные причины обращаться к контексту:

- Нужно явно отправлять и получать значения в [XCom](https://www.astronomer.io/docs/learn/airflow-passing-data-between-tasks#xcom) с произвольным ключом.
- Нужно использовать [логическую дату](https://www.astronomer.io/docs/learn/scheduling-in-airflow#scheduling-concepts) DAG run в задаче, например в имени файла.
- Нужно использовать [параметры уровня DAG](https://www.astronomer.io/docs/learn/airflow-params) в задачах.

В этом документе — какие данные хранятся в контексте Airflow и как к ним обращаться.

## Необходимая база

Полезно понимать:

- Операторы Airflow. См. [Что такое оператор?](../01.%20astronomer-basic/operators.md).
- Основы Python. См. [документацию Python](https://docs.python.org/3/tutorial/index.html).
- Основы Airflow. См. [Введение в Apache Airflow](https://www.astronomer.io/docs/learn/intro-to-airflow).

## Доступ к контексту Airflow

Контекст доступен во всех задачах Airflow. Получить из него данные можно так:

- Обратиться к аргументу **context** в методе `.execute` любого традиционного или кастомного оператора.
- Использовать [Jinja-шаблоны](https://www.astronomer.io/docs/learn/templating) в традиционных операторах Airflow.
- Передать аргумент **context** в функцию с декоратором `@asset`. См. [Ассеты и data-aware планирование](https://www.astronomer.io/docs/learn/airflow-datasets).
- Передать аргумент **\*\*context** в функцию, используемую в [задаче с декоратором `@task`](https://www.astronomer.io/docs/learn/airflow-decorators) или в [PythonOperator](https://registry.astronomer.io/providers/apache-airflow/versions/latest/modules/PythonOperator).

Обращаться к словарю контекста Airflow вне задачи нельзя.

### Получение контекста через декоратор @task или PythonOperator

Чтобы получить контекст в задаче с декоратором `@task` или в PythonOperator, добавьте в функцию задачи аргумент **\*\*context**. Контекст будет доступен как словарь.

Примеры вывода полного словаря контекста:

```python
from airflow.sdk import task
from pprint import pprint


@task
def print_context(**context):
    pprint(context)
```

```python
from airflow.providers.standard.operators.python import PythonOperator
from pprint import pprint


def print_context_func(**context):
    pprint(context)


print_context = PythonOperator(
    task_id="print_context",
    python_callable=print_context_func,
)
```

### Получение контекста через Jinja-шаблоны

К многим полям контекста можно обратиться через [Jinja-шаблоны](https://www.astronomer.io/docs/learn/templating). Список параметров оператора, поддерживающих шаблоны, хранится в атрибуте `.template_fields`.

Например, логическую дату DAG run в формате `YYYY-MM-DD` можно подставить в параметр `bash_command` BashOperator шаблоном `{{ ds }}`:

```python
from airflow.providers.standard.operators.bash import BashOperator

print_logical_date = BashOperator(
    task_id="print_logical_date",
    bash_command="echo {{ ds }}",
)
```

Часто Jinja используют, чтобы подставить в параметр традиционной задачи значение из [XCom](https://www.astronomer.io/docs/learn/airflow-passing-data-between-tasks#xcom). В примере ниже первая задача `return_greeting` кладёт в XCom строку "Hello", вторая `greet_friend` через шаблон достаёт это значение из объекта `ti` (task instance) контекста и выводит в лог `Hello friend! :)`.

```python
from airflow.providers.standard.operators.bash import BashOperator
from airflow.sdk import task


@task
def return_greeting():
    return "Hello"


greet_friend = BashOperator(
    task_id="greet_friend",
    bash_command="echo '{{ ti.xcom_pull(task_ids='return_greeting') }} friend! :)'",
)

return_greeting() >> greet_friend
```

Актуальный список доступных шаблонов: [документация Airflow](https://airflow.apache.org/docs/apache-airflow/stable/templates-ref.html). Передача данных между задачами через XCom: [Pass data between tasks](https://www.astronomer.io/docs/learn/airflow-passing-data-between-tasks).

### Получение контекста в кастомных операторах

В традиционном операторе контекст всегда передаётся в метод `.execute` аргументом **context**. В [кастомном операторе](https://www.astronomer.io/docs/learn/airflow-importing-custom-hooks-operators) в методе `execute` нужно объявить этот аргумент, как в примере:

```python
from airflow.sdk.bases.operator import BaseOperator


class PrintDAGIDOperator(BaseOperator):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)

    def execute(self, context):
        print(context["dag"].dag_id)
```

## Часто используемые ключи контекста

Ниже — наиболее употребительные ключи словаря контекста. Полный список ключей и их типов: [исходный код Airflow](https://github.com/apache/airflow/blob/main/airflow-core/src/airflow/utils/context.py).

### ti / task_instance

Ключ **ti** (или **task_instance**) содержит [объект TaskInstance](https://airflow.apache.org/docs/apache-airflow/stable/_api/airflow/models/taskinstance/index.html). Чаще всего используют атрибуты `.xcom_pull` и `.xcom_push` для работы с [XCom](https://www.astronomer.io/docs/learn/airflow-passing-data-between-tasks).

В следующем DAG контекст используется для явной передачи данных между задачами через `context["ti"].xcom_push(...)` и `context["ti"].xcom_pull(...)`:

```python
from pendulum import datetime
from airflow.decorators import dag, task


@dag(
    start_date=datetime(2023, 6, 1),
    schedule=None,
    catchup=False,
)
def context_and_xcom():
    @task
    def upstream_task(**context):
        context["ti"].xcom_push(key="my_explicitly_pushed_xcom", value=23)
        return 19

    @task
    def downstream_task(passed_num, **context):
        returned_num = context["ti"].xcom_pull(
            task_ids="upstream_task", key="return_value"
        )
        explicit_num = context["ti"].xcom_pull(
            task_ids="upstream_task", key="my_explicitly_pushed_xcom"
        )

        print("Returned Num: ", returned_num)
        print("Passed Num: ", passed_num)
        print("Explicit Num: ", explicit_num)

    downstream_task(upstream_task())


context_and_xcom()
```

В логах задачи `downstream_task` будет выведено:

```text
[2023-06-16, 13:14:11 UTC] {logging_mixin.py:149} INFO - Returned Num:  19
[2023-06-16, 13:14:11 UTC] {logging_mixin.py:149} INFO - Passed Num:  19
[2023-06-16, 13:14:11 UTC] {logging_mixin.py:149} INFO - Explicit Num:  23
```

### Ключи планирования (scheduling keys)

Часто контекст нужен, чтобы получить данные о планировании DAG. Типичный приём — использовать метку времени логической даты в именах файлов, создаваемых DAG, чтобы у каждого DAG run был свой файл.

В примере ниже для каждого DAG run создаётся текстовый файл в папке `include` с меткой времени в имени в формате `YYYY-MM-DDTHH:MM:SS+00:00`. Список ключей контекста, связанных с временем: [Templates reference](https://airflow.apache.org/docs/apache-airflow/stable/templates-ref.html). Подстановка этих значений в шаблонируемые параметры традиционных операторов: [Jinja templating](https://www.astronomer.io/docs/learn/templating).

```python
from airflow.sdk import task


@task
def write_file_with_ts(**context):
    ts = context["ts"]
    with open(f"include/{ts}_hello.txt", "a") as f:
        f.write("Hello, World!")
```

### dag_run

Ключ **dag_run** содержит [объект DAG run](https://github.com/apache/airflow/blob/main/airflow-core/src/airflow/models/dagrun.py). Часто используют атрибут `run_type` — он показывает, как был запущен DAG (scheduled, manual, dataset_triggered и т.д.).

```python
from airflow.sdk import task


@task
def print_dagrun_info(**context):
    print(context["dag_run"].run_type)
```

### params

Ключ **params** — словарь всех DAG- и task-level параметров, переданных данному экземпляру задачи. Отдельный параметр доступен по своему ключу.

```python
from airflow.sdk import task


@task
def print_param(**context):
    print(context["params"]["my_favorite_param"])
```

Подробнее о params: [Airflow params guide](https://www.astronomer.io/docs/learn/airflow-params).

### var

Ключ **var** даёт доступ ко всем [переменным Airflow](https://www.astronomer.io/docs/learn/airflow-variables) инстанса. Обычно это пары ключ–значение для редко меняющихся данных уровня инстанса.

```python
from airflow.sdk import task


@task
def get_var_from_context(**context):
    print(context["var"]["value"].get("my_regular_var"))
    print(context["var"]["json"].get("my_json_var")["num2"])
```

## Ключи контекста, связанные с метками времени

Набор ключей с метками времени в контексте зависит от типа DAG run (scheduled или asset-triggered). В примере ниже задача выводит полный список ключей контекста и все ключи, относящиеся к планированию:

```python
from airflow.sdk import dag, task


@dag
def my_context_dag():
    @task
    def print_context_keys(**context):
        print("All context keys: ", context.keys())
        print("--------------")
        print("DAG run details relating to timestamps:")
        print("run_id from the dag_run key: ", context["dag_run"].run_id)
        print("logical_date from the dag_run key: ", context["dag_run"].logical_date)
        print("data_interval_start from the dag_run key: ", context["dag_run"].data_interval_start)
        print("data_interval_end from the dag_run key: ", context["dag_run"].data_interval_end)
        print("run_after from the dag_run key: ", context["dag_run"].run_after)
        print("start_date from the dag_run key: ", context["dag_run"].start_date)
        print("end_date from the dag_run key: ", context["dag_run"].end_date)
        print("--------------")
        print("Top-level context keys relating to timestamps:")
        print("prev_start_date_success: ", context["prev_start_date_success"])
        print("prev_end_date_success: ", context["prev_end_date_success"])

        # Следующие ключи есть только при scheduled run или ручном/API запуске с указанной logical date.
        # При asset-triggered run или ручном запуске с logical_date=None этих ключей НЕТ —
        # обращение к ним вызовет KeyError!
        print("logical_date: ", context["logical_date"])
        print("ds: ", context["ds"])
        print("ds_nodash: ", context["ds_nodash"])
        print("ts: ", context["ts"])
        print("ts_nodash: ", context["ts_nodash"])
        print("data_interval_start: ", context["data_interval_start"])
        print("data_interval_end: ", context["data_interval_end"])
        print("previous_data_interval_start_success: ", context["prev_data_interval_start_success"])
        print("previous_data_interval_end_success: ", context["prev_data_interval_end_success"])

    print_context_keys()


my_context_dag()
```

Если DAG запущен по ассету или создан ручной/API запуск с явным `logical_date=None`, в словаре контекста **не будет** следующих ключей; обращение к ним приведёт к **KeyError**:

- `previous_data_interval_end_success`
- `previous_data_interval_start_success`
- `data_interval_end`
- `data_interval_start`
- `ts_nodash`
- `ts`
- `ds_nodash`
- `ds`
- `logical_date`

---

[← К содержанию](README.md) | [Декораторы →](airflow-decorators.md) | [XCom →](passing-data-between-tasks.md)
