# Pythonic DAG с TaskFlow API

В первом туториале вы собрали первый DAG на традиционных операторах вроде `BashOperator`. Теперь рассмотрим более современный и «питоновский» способ — **TaskFlow API** (появился в Airflow 2.0).

TaskFlow API упрощает код: вы пишете обычные Python-функции, вешаете на них декораторы, а Airflow сам создаёт задачи, связывает зависимости и передаёт данные между ними.

В этом туториале соберём простой ETL-пайплайн: Extract → Transform → Load.

## Общая картина: пайплайн на TaskFlow

Так выглядит полный пайплайн на TaskFlow. Ниже разберём его по шагам.

*Источник: [airflow/example_dags/tutorial_taskflow_api.py](https://airflow.apache.org/docs/apache-airflow/stable/_modules/airflow/example_dags/tutorial_taskflow_api.html)*

```python
import json
import pendulum

from airflow.sdk import dag, task

@dag(
    schedule=None,
    start_date=pendulum.datetime(2021, 1, 1, tz="UTC"),
    catchup=False,
    tags=["example"],
)
def tutorial_taskflow_api():
    """
    ### TaskFlow API Tutorial Documentation
    This is a simple data pipeline example which demonstrates the use of
    the TaskFlow API using three simple tasks for Extract, Transform, and Load.
    Documentation that goes along with the Airflow TaskFlow API tutorial is
    located
    [here](https://airflow.apache.org/docs/apache-airflow/stable/tutorial_taskflow_api.html)
    """
    @task()
    def extract():
        """
        #### Extract task
        A simple Extract task to get data ready for the rest of the data
        pipeline. In this case, getting data is simulated by reading from a
        hardcoded JSON string.
        """
        data_string = '{"1001": 301.27, "1002": 433.21, "1003": 502.22}'
        order_data_dict = json.loads(data_string)
        return order_data_dict

    @task(multiple_outputs=True)
    def transform(order_data_dict: dict):
        """
        #### Transform task
        A simple Transform task which takes in the collection of order data and
        computes the total order value.
        """
        total_order_value = 0
        for value in order_data_dict.values():
            total_order_value += value
        return {"total_order_value": total_order_value}

    @task()
    def load(total_order_value: float):
        """
        #### Load task
        A simple Load task which takes in the result of the Transform task and
        instead of saving it to end user review, just prints it out.
        """
        print(f"Total order value is: {total_order_value:.2f}")

    order_data = extract()
    order_summary = transform(order_data)
    load(order_summary["total_order_value"])

tutorial_taskflow_api()
```

## Шаг 1: Определение DAG

DAG по-прежнему задаётся Python-скриптом, который загружает и разбирает Airflow. На этот раз DAG объявляется через декоратор `@dag`.

*Источник: [airflow/example_dags/tutorial_taskflow_api.py](https://airflow.apache.org/docs/apache-airflow/stable/_modules/airflow/example_dags/tutorial_taskflow_api.html)*

```python
@dag(
    schedule=None,
    start_date=pendulum.datetime(2021, 1, 1, tz="UTC"),
    catchup=False,
    tags=["example"],
)
def tutorial_taskflow_api():
    """
    ### TaskFlow API Tutorial Documentation
    This is a simple data pipeline example which demonstrates the use of
    the TaskFlow API using three simple tasks for Extract, Transform, and Load.
    Documentation that goes along with the Airflow TaskFlow API tutorial is
    located
    [here](https://airflow.apache.org/docs/apache-airflow/stable/tutorial_taskflow_api.html)
    """
```

Чтобы Airflow обнаружил этот DAG, нужно вызвать задекорированную функцию:

*Источник: [airflow/example_dags/tutorial_taskflow_api.py](https://airflow.apache.org/docs/apache-airflow/stable/_modules/airflow/example_dags/tutorial_taskflow_api.html)*

```python
tutorial_taskflow_api()
```

*Изменено в 2.4:* при использовании декоратора `@dag` или блока `with` не обязательно присваивать DAG глобальной переменной — Airflow найдёт его сам.

Готовый DAG можно посмотреть в UI Airflow в виде графа (Graph View).

## Шаг 2: Задачи с декоратором @task

В TaskFlow каждая задача — обычная Python-функция. Декоратор `@task` превращает её в задачу, которую Airflow может планировать и запускать. Пример задачи `extract`:

*Источник: [airflow/example_dags/tutorial_taskflow_api.py](https://airflow.apache.org/docs/apache-airflow/stable/_modules/airflow/example_dags/tutorial_taskflow_api.html)*

```python
@task()
def extract():
    """
    #### Extract task
    A simple Extract task to get data ready for the rest of the data
    pipeline. In this case, getting data is simulated by reading from a
    hardcoded JSON string.
    """
    data_string = '{"1001": 301.27, "1002": 433.21, "1003": 502.22}'
    order_data_dict = json.loads(data_string)
    return order_data_dict
```

Возвращаемое значение передаётся в следующую задачу — вручную с XCom работать не нужно. TaskFlow сам использует XCom для передачи данных. Задачи `transform` и `load` задаются по тому же принципу.

Обратите внимание на `@task(multiple_outputs=True)`: так мы говорим Airflow, что функция возвращает словарь, который нужно разложить по отдельным XCom. Каждый ключ словаря станет своей записью XCom, и в следующих задачах можно обращаться к конкретным полям. Без `multiple_outputs=True` весь словарь сохраняется одним XCom и используется целиком.

## Шаг 3: Сборка потока

Когда задачи определены, пайплайн собирается простыми вызовами функций. По этим вызовам Airflow выстраивает зависимости и передачу данных.

*Источник: [airflow/example_dags/tutorial_taskflow_api.py](https://airflow.apache.org/docs/apache-airflow/stable/_modules/airflow/example_dags/tutorial_taskflow_api.html)*

```python
order_data = extract()
order_summary = transform(order_data)
load(order_summary["total_order_value"])
```

Этого достаточно — Airflow понимает, как планировать и запускать пайплайн.

## Запуск DAG

Чтобы включить и запустить DAG:

1. Откройте UI Airflow.
2. Найдите DAG в списке и включите его переключателем.
3. Запустите вручную кнопкой «Trigger Dag» или дождитесь запуска по расписанию.

## Что происходит под капотом?

По сравнению с Airflow 1.x это выглядит как магия. Ниже — сравнение со старым подходом.

### «Старый» способ: ручные зависимости и XCom

До TaskFlow данные между задачами передавали вручную через XCom, используя операторы вроде `PythonOperator`.

Тот же DAG в традиционном виде:

```python
import json
import pendulum
from airflow.sdk import DAG
from airflow.providers.standard.operators.python import PythonOperator


def extract():
    # Старый способ: имитация извлечения из JSON
    data_string = '{"1001": 301.27, "1002": 433.21, "1003": 502.22}'
    return json.loads(data_string)


def transform(ti):
    # Старый способ: вручную забираем из XCom
    order_data_dict = ti.xcom_pull(task_ids="extract")
    total_order_value = sum(order_data_dict.values())
    return {"total_order_value": total_order_value}


def load(ti):
    # Старый способ: вручную забираем из XCom
    total = ti.xcom_pull(task_ids="transform")["total_order_value"]
    print(f"Total order value is: {total:.2f}")


with DAG(
    dag_id="legacy_etl_pipeline",
    schedule=None,
    start_date=pendulum.datetime(2021, 1, 1, tz="UTC"),
    catchup=False,
    tags=["example"],
) as dag:
    extract_task = PythonOperator(task_id="extract", python_callable=extract)
    transform_task = PythonOperator(task_id="transform", python_callable=transform)
    load_task = PythonOperator(task_id="load", python_callable=load)

    extract_task >> transform_task >> load_task
```

> **Примечание.** Результат такой же, как у примера на TaskFlow API, но с явным управлением XCom и зависимостями.

### Подход TaskFlow

С TaskFlow всё это делается автоматически.

*Источник: [airflow/example_dags/tutorial_taskflow_api.py](https://airflow.apache.org/docs/apache-airflow/stable/_modules/airflow/example_dags/tutorial_taskflow_api.html)*

(Тот же код, что в начале раздела «Общая картина».)

Airflow по-прежнему использует XCom и строит граф зависимостей — просто это скрыто, и вы занимаетесь бизнес-логикой.

## Как работают XCom

Возвращаемые значения TaskFlow автоматически сохраняются в XCom. Их можно посмотреть в UI во вкладке «XCom». Для обычных операторов по-прежнему доступен ручной `xcom_pull()`.

## Обработка ошибок и повторы

Повторы (retries) настраиваются в декораторе задачи:

```python
@task(retries=3)
def my_task(): ...
```

Так временные сбои не обязательно приводят к падению задачи.

## Параметризация задач

Задекорированные задачи можно использовать в разных DAG и переопределять параметры вроде `task_id` или `retries`:

```python
start = add_task.override(task_id="start")(1, 2)
```

Задачи можно импортировать из общего модуля.

## Что изучить дальше

Дальнейшие шаги:

- Добавить новую задачу — например, фильтр или валидацию.
- Менять возвращаемые значения и передавать несколько выходов.
- Поэкспериментировать с retries и переопределениями через `.override(task_id="...")`.
- В UI посмотреть, как данные проходят между задачами (логи, зависимости).

См. также:

- Следующий шаг туториала: [Building a Simple Data Pipeline](https://airflow.apache.org/docs/apache-airflow/stable/tutorial/pipeline.html).
- [TaskFlow API](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/taskflow.html) и раздел ниже про продвинутые паттерны.
- [Core Concepts](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/index.html).

## Продвинутые паттерны TaskFlow

Когда базовый вариант освоен, можно переходить к более сложным приёмам.

### Переиспользование задекорированных задач

Одни и те же задекорированные задачи можно использовать в разных DAG и запусках — удобно для общих утилит и правил. Метод `.override()` меняет метаданные задачи (`task_id`, `retries` и т.д.):

```python
start = add_task.override(task_id="start")(1, 2)
```

Задачи можно импортировать из общего модуля.

### Конфликтующие зависимости

Иногда задаче нужны другие Python-зависимости (спецбиблиотеки, системные пакеты). TaskFlow поддерживает несколько окружений выполнения.

**Динамический virtualenv** — временное виртуальное окружение создаётся при запуске задачи. Подходит для экспериментов, но может давать задержку на «холодный» старт.

*Пример: example_python_decorator.py*

```python
@task.virtualenv(
    task_id="virtualenv_python", requirements=["colorama==0.4.0"], system_site_packages=False
)
def callable_virtualenv():
    """
    Example function that will be performed in a virtual environment.
    Importing at the module level ensures that it will not attempt to import the
    library before it is installed.
    """
    from time import sleep
    from colorama import Back, Fore, Style

    print(Fore.RED + "some red text")
    print(Back.GREEN + "and with a green background")
    # ...
    print("Finished")

virtualenv_task = callable_virtualenv()
```

**Внешний Python** — задача выполняется в уже установленном интерпретаторе (удобно для единого или общего virtualenv).

*Пример: example_python_decorator.py*

```python
@task.external_python(task_id="external_python", python=PATH_TO_PYTHON_BINARY)
def callable_external_python():
    import sys
    from time import sleep
    print(f"Running task via {sys.executable}")
    # ...
```

**Docker** — задача выполняется в Docker-контейнере. Всё необходимое упаковано в образ; на воркере должен быть Docker.

*Пример: example_taskflow_api_docker_virtualenv.py*

```python
@task.docker(image="python:3.9-slim-bookworm", multiple_outputs=True)
def transform(order_data_dict: dict):
    total_order_value = 0
    for value in order_data_dict.values():
        total_order_value += value
    return {"total_order_value": total_order_value}
```

> **Примечание.** Нужен Airflow 2.2 и провайдер Docker.

**KubernetesPodOperator** — задача выполняется в поде Kubernetes, изолированно от основного окружения Airflow. Подходит для тяжёлых задач или своих рантаймов.

*Пример: example_kubernetes_decorator.py*

```python
@task.kubernetes(
    image="python:3.9-slim-buster",
    name="k8s_test",
    namespace="default",
    in_cluster=False,
    config_file="/path/to/.kube/config",
)
def execute_in_k8s_pod():
    import time
    print("Hello from k8s pod")
    time.sleep(2)

@task.kubernetes(image="python:3.9-slim-buster", namespace="default", in_cluster=False)
def print_pattern():
    # ...

execute_in_k8s_pod_instance = execute_in_k8s_pod()
print_pattern_instance = print_pattern()
execute_in_k8s_pod_instance >> print_pattern_instance
```

> **Примечание.** Нужен Airflow 2.4 и провайдер Kubernetes.

### Сенсоры

Декоратор `@task.sensor` позволяет описывать сенсоры обычными Python-функциями. Поддерживаются режимы poke и reschedule.

*Пример: example_sensor_decorator.py*

```python
import pendulum
from airflow.sdk import PokeReturnValue, dag, task

@dag(
    schedule=None,
    start_date=pendulum.datetime(2021, 1, 1, tz="UTC"),
    catchup=False,
    tags=["example"],
)
def example_sensor_decorator():
    @task.sensor(poke_interval=60, timeout=3600, mode="reschedule")
    def wait_for_upstream() -> PokeReturnValue:
        return PokeReturnValue(is_done=True, xcom_value="xcom_value")

    @task
    def dummy_operator() -> None:
        pass

    wait_for_upstream() >> dummy_operator()

tutorial_etl_dag = example_sensor_decorator()
```

### Совмещение с обычными задачами

TaskFlow-задачи можно сочетать с классическими операторами — при использовании провайдеров или постепенном переходе на TaskFlow.

Цепочку можно строить через `>>`; данные от TaskFlow-задачи передавать через атрибут `.output`.

### Шаблонизация в TaskFlow

Как и у обычных задач, у задекорированных функций TaskFlow могут быть шаблонные аргументы — в том числе загрузка из файлов и runtime-параметры.

При вызове callable Airflow передаёт набор keyword-аргументов, соответствующих тому, что доступно в Jinja-шаблонах. Чтобы их получить, добавьте нужные ключи контекста как keyword-аргументы функции.

Пример — функция получает `ti` и `next_ds`:

```python
@task
def my_python_callable(*, ti, next_ds):
    pass
```

Можно принять весь контекст через `**kwargs`. Это может немного замедлить выполнение, так как подставляется полный контекст. Предпочтительнее явно перечислять нужные аргументы.

```python
@task
def my_python_callable(**kwargs):
    ti = kwargs["ti"]
    next_ds = kwargs["next_ds"]
```

Если контекст нужен где-то глубоко в коде и не хочется прокидывать его из callable, можно получить его через `get_current_context`:

```python
from airflow.sdk import get_current_context

def some_function_in_your_library():
    context = get_current_context()
    ti = context["ti"]
```

Аргументы, передаваемые в задекорированные функции, автоматически проходят шаблонизацию. Файлы можно подставлять через `templates_exts`:

```python
@task(templates_exts=[".sql"])
def read_sql(sql): ...
```

### Условное выполнение

Декораторы `@task.run_if()` и `@task.skip_if()` позволяют решать по условию во время выполнения, запускать задачу или пропускать её, не меняя структуру DAG.

```python
@task.run_if(lambda ctx: ctx["task_instance"].task_id == "run")
@task.bash()
def echo():
    return "echo 'run'"
```

## Что дальше

Дальнейшие шаги:

- [Asset-Aware Scheduling](https://airflow.apache.org/docs/apache-airflow/stable/authoring-and-scheduling/asset-scheduling.html) — пайплайны, ориентированные на ассеты.
- [Scheduling Options](https://airflow.apache.org/docs/apache-airflow/stable/authoring-and-scheduling/index.html#scheduling-section) — варианты расписаний.
- Следующий туториал: [Building a Simple Data Pipeline](https://airflow.apache.org/docs/apache-airflow/stable/tutorial/pipeline.html).

---

*Источник: [Airflow 3.1.7 — Tutorial: TaskFlow API](https://airflow.apache.org/docs/apache-airflow/stable/tutorial/taskflow.html). Перевод неофициальный.*
