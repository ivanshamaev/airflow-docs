# Декораторы Airflow (Airflow decorators) и TaskFlow API

**TaskFlow API** — функциональный способ задавать DAG и задачи с помощью декораторов; он упрощает передачу данных между задачами и объявление зависимостей. Декораторы TaskFlow (например, `@task`) позволяют передавать данные между задачами, подставляя результат одной задачи аргументом в другую. Декораторы — более простой и чистый способ описывать задачи и DAG; их можно сочетать с традиционными операторами.

В этом руководстве: зачем нужны декораторы, какие декораторы есть в Airflow, пример DAG, когда их использовать и как сочетать с традиционными операторами в одном DAG.

## Необходимая база

Полезно понимать:

- Операторы Airflow. См. [Что такое оператор?](../01.%20astronomer-basic/operators.md).
- Основы Python. См. [документацию Python](https://docs.python.org/3/tutorial/index.html).

## Что такое декоратор?

В Python [декоратор](https://realpython.com/primer-on-python-decorators/) — это функция, которая принимает другую функцию и расширяет её поведение. Например, декоратор `@multiply_by_100_decorator` принимает любую функцию как аргумент `decorated_function` и возвращает результат этой функции, умноженный на 100.

```python
# определение функции-декоратора
def multiply_by_100_decorator(decorated_function):
    def wrapper(num1, num2):
        result = decorated_function(num1, num2) * 100
        return result
    return wrapper


@multiply_by_100_decorator
def add(num1, num2):
    return num1 + num2


@multiply_by_100_decorator
def subtract(num1, num2):
    return num1 - num2


# вызов задекорированных функций
print(add(1, 9))       # выведет 1000
print(subtract(4, 2))  # выведет 200
```

В Airflow декораторы делают больше, чем в этом примере, но идея та же: декоратор Airflow расширяет поведение обычной Python-функции и превращает её в задачу, группу задач или DAG.

## Когда использовать TaskFlow API

TaskFlow API в Airflow призван упростить написание DAG за счёт сокращения шаблонного кода по сравнению с традиционными операторами. В результате DAG часто становятся короче и понятнее.

В целом выбор между TaskFlow API и традиционным стилем — вопрос предпочтений. В большинстве случаев декоратор TaskFlow и соответствующий традиционный оператор ведут себя одинаково. В одном DAG можно [смешивать декораторы и традиционные операторы](airflow-decorators.md).

## Как использовать TaskFlow API

TaskFlow API позволяет описывать Python-задачи декораторами. Передача данных между задачами через XCom и зависимости между задачами определяются автоматически.

Задать задачу декоратором легко. Ниже — один и тот же ETL-DAG: получение данных из API, обработка и сохранение. Сначала традиционный вариант, затем тот же DAG на декораторах.

**Традиционный синтаксис:**

```python
import logging
from datetime import datetime

import requests
from airflow import DAG
from airflow.operators.python import PythonOperator

API = "https://api.coingecko.com/api/v3/simple/price?ids=bitcoin&vs_currencies=usd&include_market_cap=true&include_24hr_vol=true&include_24hr_change=true&include_last_updated_at=true"


def _extract_bitcoin_price():
    return requests.get(API).json()["bitcoin"]


def _process_data(ti):
    response = ti.xcom_pull(task_ids="extract_bitcoin_price")
    logging.info(response)
    processed_data = {"usd": response["usd"], "change": response["usd_24h_change"]}
    ti.xcom_push(key="processed_data", value=processed_data)


def _store_data(ti):
    data = ti.xcom_pull(task_ids="process_data", key="processed_data")
    logging.info(f"Store: {data['usd']} with change {data['change']}")


with DAG(
    "classic_dag", schedule="@daily", start_date=datetime(2021, 12, 1), catchup=False
):
    extract_bitcoin_price = PythonOperator(
        task_id="extract_bitcoin_price", python_callable=_extract_bitcoin_price
    )
    process_data = PythonOperator(task_id="process_data", python_callable=_process_data)
    store_data = PythonOperator(task_id="store_data", python_callable=_store_data)

    extract_bitcoin_price >> process_data >> store_data
```

**Вариант с декораторами:**

```python
import logging
from datetime import datetime
from typing import Dict

import requests
from airflow.decorators import dag, task

API = "https://api.coingecko.com/api/v3/simple/price?ids=bitcoin&vs_currencies=usd&include_market_cap=true&include_24hr_vol=true&include_24hr_change=true&include_last_updated_at=true"


@dag(schedule="@daily", start_date=datetime(2021, 12, 1), catchup=False)
def taskflow():
    @task(task_id="extract", retries=2)
    def extract_bitcoin_price() -> Dict[str, float]:
        return requests.get(API).json()["bitcoin"]

    @task(multiple_outputs=True)
    def process_data(response: Dict[str, float]) -> Dict[str, float]:
        logging.info(response)
        return {"usd": response["usd"], "change": response["usd_24h_change"]}

    @task
    def store_data(data: Dict[str, float]):
        logging.info(f"Store: {data['usd']} with change {data['change']}")

    store_data(process_data(extract_bitcoin_price()))


taskflow()
```

В варианте с декораторами не нужно явно создавать PythonOperator, кода меньше и его проще читать. Не требуется вручную вызывать `ti.xcom_pull` и `ti.xcom_push` для [передачи данных между задачами](passing-data-between-tasks.md): при задании зависимостей через `store_data(process_data(extract_bitcoin_price()))` всё делает TaskFlow API.

Дополнительно при использовании декораторов стоит помнить:

- **Результат вызванной задачи можно сохранить в переменную** и передать в несколько нижестоящих задач:

```python
from airflow.sdk import task
import random


@task
def get_fruit_options():
    return ["peach", "raspberry", "pineapple"]


@task
def eat_a_fruit(list):
    index = random.randint(0, len(list) - 1)
    print(f"I'm eating a {list[index]}!")


@task
def gift_a_fruit(list):
    index = random.randint(0, len(list) - 1)
    print(f"I'm giving you a {list[index]}!")


my_fruits = get_fruit_options()
eat_a_fruit(my_fruits)
gift_a_fruit(my_fruits)
```

- **Декорировать можно функцию, импортированную из другого файла.** Это удобно для длинных функций — DAG-файл остаётся короче:

```python
from airflow.sdk import task
from include.my_file import my_function


@task
def taskflow_func():
    my_function()
```

- **Если одну и ту же задачу вызывать несколько раз и не переопределять `task_id`**, Airflow создаёт уникальные task_id, добавляя номер к исходному (например, `say_hello`, `say_hello__1`, `say_hello__2` и т.д.):

```python
from airflow.sdk import task


@task
def say_hello(dog):
    return f"Hello {dog}!"


# 4 вызова — 4 задачи в DAG
say_hello("Avery")                                    # task_id `say_hello` → "Hello Avery!"
say_hello.override(task_id="greet_dog")("Piglet")     # task_id `greet_dog` → "Hello Piglet!"
say_hello("Peanut")                                   # task_id `say_hello__1` → "Hello Peanut!"
say_hello("Butter")                                   # task_id `say_hello__2` → "Hello Butter!"
```

- **Переопределить параметры задачи при вызове** можно методом `.override()`. Скобки `()` в конце строки вызывают задачу с уже подставленными параметрами:

```python
# создаётся задача с task_id `greeting`
taskflow_func.override(retries=5, pool="my_other_pool", task_id="greeting")()
```

- **При определении задачи `task_id` по умолчанию совпадает с именем функции.** Чтобы задать другой (как в задаче `extract` выше), передайте `task_id` в декоратор. Аналогично в декораторе можно задать и другие параметры BaseOperator (`retries`, `pool` и т.д.):

```python
from airflow.sdk import task


@task(
    task_id="say_hello_world",
    retries=3,
    pool="my_pool",
)
def taskflow_func():
    return "Hello World"


taskflow_func()  # создаётся задача с task_id `say_hello_world`
```

- **Все задекорированные функции в файле DAG должны быть вызваны**, чтобы Airflow зарегистрировал задачи и DAG. Например, в конце предыдущего примера вызывается `taskflow()` — функция DAG.

Больше примеров: [вебинары Astronomer](https://www.astronomer.io/events/webinars/writing-functional-dags-with-decorators/) и [TaskFlow API tutorial](https://airflow.apache.org/docs/apache-airflow/stable/tutorial/taskflow.html) в документации Apache Airflow.

## Смешение декораторов TaskFlow и традиционных операторов

Если в DAG уже используются PythonOperator и другие операторы без декораторов, декорированные функции и традиционные операторы можно сочетать в одном DAG. Например, к предыдущему примеру можно добавить EmailOperator:

```python
import logging
from datetime import datetime
from typing import Dict

import requests
from airflow.decorators import dag, task
from airflow.operators.email import EmailOperator

API = "https://api.coingecko.com/api/v3/simple/price?ids=bitcoin&vs_currencies=usd&include_market_cap=true&include_24hr_vol=true&include_24hr_change=true&include_last_updated_at=true"


@dag(schedule="@daily", start_date=datetime(2021, 12, 1), catchup=False)
def taskflow():
    @task(task_id="extract", retries=2)
    def extract_bitcoin_price() -> Dict[str, float]:
        return requests.get(API).json()["bitcoin"]

    @task(multiple_outputs=True)
    def process_data(response: Dict[str, float]) -> Dict[str, float]:
        logging.info(response)
        return {"usd": response["usd"], "change": response["usd_24h_change"]}

    @task
    def store_data(data: Dict[str, float]):
        logging.info(f"Store: {data['usd']} with change {data['change']}")

    email_notification = EmailOperator(
        task_id="email_notification",
        to="noreply@astronomer.io",
        subject="dag completed",
        html_content="the dag has finished",
    )

    store_data(process_data(extract_bitcoin_price())) >> email_notification


taskflow()
```

Зависимости с традиционными операторами по-прежнему задаются функцией `chain` или операторами сдвига (`>>`). Подробнее: [Управление зависимостями между задачами и группами задач](../01.%20astronomer-basic/task-dependencies.md).

Передавать данные между декорированными задачами и традиционными операторами можно через [XCom](passing-data-between-tasks.md). Ниже — примеры для разных комбинаций.

### TaskFlow → TaskFlow

Если обе задачи заданы через TaskFlow API, данные передаются напрямую: вызов вышестоящей задачи передаётся аргументом в нижестоящую. Зависимость между ними выводится автоматически.

```python
from airflow.sdk import task


@task
def get_23_TF():
    return 23


@task
def plus_10_TF(x):
    return x + 10


plus_10_TF(get_23_TF())  # plus_10_TF вернёт 33
# или plus_10_TF(x=get_23_TF()) при использовании kwargs
```

### Традиционный оператор → TaskFlow

Передайте значение **.output** традиционной задачи в вызов TaskFlow-задачи — зависимость создастся автоматически.

```python
from airflow.sdk import task
from airflow.providers.standard.operators.python import PythonOperator


def first_task_callable():
    return "hello world"


first_task = PythonOperator(
    task_id="first_task",
    python_callable=first_task_callable,
)

first_task_result = first_task.output


@task
def second_task(first_task_result_value):
    return f"{first_task_result_value} and hello again"


# first_task_result (результат традиционного оператора) передаётся аргументом в second_task
# (TaskFlow) — зависимость регистрируется автоматически
second_task(first_task_result)
```

Если данные не передаются аргументом, а нужно только обеспечить порядок выполнения, задайте зависимость через `chain` или `>>`:

```python
from airflow.sdk import task, chain
from airflow.providers.standard.operators.empty import EmptyOperator


@task
def first_task():
    return "hello world"


_second_task = EmptyOperator(task_id="second_task")
_first_task = first_task()

chain(_first_task, _second_task)
```

Что возвращает задача, зависит от оператора. В [Astronomer Registry](https://registry.astronomer.io/) можно посмотреть описание возвращаемых значений и параметров, влияющих на формат.

### TaskFlow → традиционный оператор

Вызов TaskFlow-задачи возвращает ссылку (внутри — XComArg), которую можно передать в [шаблонируемые поля](jinja-templating.md) традиционного оператора — зависимость между задачами создаётся автоматически.

Список шаблонируемых полей у каждого оператора свой. В [Astronomer Registry](https://registry.astronomer.io/) указано, какие поля по умолчанию шаблонируемые; [управление шаблонируемыми полями](https://www.astronomer.io/docs/learn/templating#templating-additional-fields) тоже возможно.

Пример передачи результата TaskFlow-функции в callable традиционного PythonOperator через аргумент:

```python
from airflow.sdk import task
from airflow.providers.standard.operators.python import PythonOperator


@task
def first_task():
    return "hello"


_first_task = first_task()


def second_task_callable(x):
    uppercase_text = x.upper()
    return f"Upper cased version of prior task result: {uppercase_text}"


# first_task_result (XComArg) передаётся в op_args — Airflow автоматически регистрирует,
# что second_task зависит от first_task
_second_task = PythonOperator(
    task_id="second_task",
    python_callable=second_task_callable,
    op_args=[_first_task],  # op_args — список XComArg
)
```

Когда важен только порядок выполнения, не передавайте результат первой задачи параметром во вторую — задайте зависимость через `chain`:

```python
from airflow.sdk import task, chain
from airflow.providers.standard.operators.empty import EmptyOperator


@task
def first_task():
    return "hello world"


_first_task = first_task()
_second_task = EmptyOperator(task_id="second_task")

chain(_first_task, _second_task)
```

### Традиционный оператор → традиционный оператор

Для полноты — пример использования выхода одного традиционного оператора в другом через атрибут **.output** вышестоящей задачи. Зависимость нужно задать явно через `chain`.

```python
from airflow.sdk import chain
from airflow.providers.standard.operators.python import PythonOperator


def get_23_traditional():
    return 23


def plus_10_traditional(x):
    return x + 10


get_23_task = PythonOperator(
    task_id="get_23_task",
    python_callable=get_23_traditional,
)

plus_10_task = PythonOperator(
    task_id="plus_10_task",
    python_callable=plus_10_traditional,
    op_args=[get_23_task.output],
)

# plus_10_task вернёт 33
# при использовании только традиционных операторов зависимости задаются явно
chain(get_23_task, plus_10_task)
```

> **Инфо.** Чтобы прочитать произвольный XCom (не только возвращаемое значение оператора), можно использовать метод `xcom_pull` внутри функции; пример — [доступ к ti / task_instance в контексте Airflow](airflow-context.md). Традиционные операторы могут получать данные из XCom через [Jinja-шаблоны](https://www.astronomer.io/docs/learn/templating) в шаблонируемых параметрах.

## Доступные декораторы Airflow

В Airflow доступно несколько декораторов. Краткий список:

- [**PySpark**](https://airflow.apache.org/docs/apache-airflow-providers-apache-spark/stable/decorators/pyspark.html) (`@task.pyspark()`) — задача с объектами SparkSession и SparkContext при их наличии.
- [**Декоратор сенсора**](https://www.astronomer.io/docs/learn/what-is-a-sensor#sensor-decorator--pythonsensor) (`@task.sensor()`) — превращает Python-функцию в сенсор.
- **Kubernetes pod** (`@task.kubernetes()`) — задача [KubernetesPodOperator](https://www.astronomer.io/docs/learn/kubepod-operator).
- [**BranchPythonVirtualenvOperator**](https://www.astronomer.io/docs/learn/airflow-branch-operator#other-branch-operators) (`@task.branch_virtualenv`) — ветвление в DAG с выполнением кода в новом виртуальном окружении (кэш через `venv_cache_path`).
- [**BranchExternalPython**](https://www.astronomer.io/docs/learn/airflow-branch-operator#other-branch-operators) (`@task.branch_external_python`) — ветвление с выполнением в уже существующем виртуальном окружении.
- [**Branch**](https://www.astronomer.io/docs/learn/airflow-branch-operator#taskbranch-branchpythonoperator) (`@task.branch()`) — ветвление по условию.
- [**Short circuit**](https://www.astronomer.io/docs/learn/airflow-branch-operator#taskshort_circuit-shortcircuitoperator) (`@task.short_circuit()`) — проверка условия; при False нижестоящие задачи пропускаются.
- **Docker** (`@task.docker()`) — задача [DockerOperator](https://registry.astronomer.io/providers/apache-airflow-providers-docker/versions/latest/modules/DockerOperator).
- **Python Virtual Env** (`@task.virtualenv()`) — выполнение Python-задачи в [виртуальном окружении](https://www.astronomer.io/events/webinars/running-airflow-tasks-in-isolated-environments/).
- **Bash** (`@task.bash()`) — задача [BashOperator](https://www.astronomer.io/docs/learn/bashoperator#how-to-use-the-bash-decorator).
- **Task** (`@task()`) — обычная Python-задача.
- **TaskGroup** (`@task_group()`) — [группа задач](https://www.astronomer.io/docs/learn/task-groups).
- **DAG** (`@dag()`) — DAG.

Также можно [создать свой декоратор задачи](https://airflow.apache.org/docs/apache-airflow/stable/howto/create-custom-decorator.html).

---

[← Контекст](airflow-context.md) | [К содержанию](README.md) | [Передача данных →](passing-data-between-tasks.md) | [Task groups →](task-groups.md)
