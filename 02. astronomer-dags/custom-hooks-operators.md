# Импорт и создание кастомных хуков и операторов

В Airflow есть множество [provider packages](https://registry.astronomer.io/) с хуками, операторами и сенсорами для типичных задач. При этом Airflow легко расширять: всё задаётся в Python. Если нужного хука, оператора или сенсора нет в open source, их можно описать самостоятельно.

В этом руководстве — как определять свои операторы и хуки и использовать их в DAG. Каталог готовых хуков, операторов и сенсоров: [Astronomer Registry](https://registry.astronomer.io/).

## Необходимая база

Полезно понимать:

- Структуру проекта Airflow. См. [Управление кодом Airflow](../01.%20astronomer-basic/managing-airflow-code.md).
- Хуки Airflow. См. [Что такое хук?](../01.%20astronomer-basic/hooks.md).
- Операторы Airflow. См. [Что такое оператор?](../01.%20astronomer-basic/operators.md).

## Создание кастомного оператора

Кастомный оператор — это Python-класс, который можно импортировать в DAG. Как и обычный оператор, при создании экземпляра он становится задачей Airflow.

Минимальные требования к кастомному оператору:

- Реализовать метод **`.execute()`**, который вызывается при выполнении задачи с этим оператором.
- Реализовать **`.__init__()`**, который вызывается при парсинге DAG.
- Наследоваться от **`BaseOperator`** или другого существующего оператора.

Опционально можно:

- Реализовать **`.post_execute()`** — выполняется после `.execute()`. Удобен для логирования, очистки или отправки дополнительных данных в XCom. Возвращаемое значение `.execute()` передаётся в `.post_execute()` аргументом `result`.
- Реализовать **`.pre_execute()`** — выполняется перед `.execute()`. Позволяет расширить существующий оператор без переопределения `.execute()`.

Пример кастомного оператора `MyOperator`:

```python
from airflow.sdk.bases.operator import BaseOperator


class MyOperator(BaseOperator):
    """
    Простой пример оператора: логирует один параметр и возвращает приветствие.
    :param my_parameter: (обязательный) произвольный параметр.
    """

    def __init__(self, my_parameter, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.my_parameter = my_parameter

    def pre_execute(self, context):
        self.log.info("Pre-execution step")

    def execute(self, context):
        self.log.info(self.my_parameter)
        # возвращаемое значение по умолчанию отправляется в XCom
        return "hi :)"

    def post_execute(self, context, result=None):
        self.log.info("Post-execution step")
```

Если кастомный оператор расширяет существующий, класс может наследоваться от него вместо BaseOperator. Подробнее: [Creating a custom Operator](https://airflow.apache.org/docs/apache-airflow/stable/howto/custom-operator.html).

> В любой оператор можно передать callable в параметры `pre_execute` или `post_execute`, чтобы добавить свою логику без создания кастомного оператора. Эта возможность считается [экспериментальной](https://airflow.apache.org/docs/apache-airflow/stable/release-process.html#experimental-features).

## Создание кастомного хука

Кастомный хук — это Python-класс, который можно импортировать в DAG. Как и обычные хуки, он используется для подключения к внешним системам из кода задач. В кастомном хуке обычно есть методы для работы с внешним API; их удобнее вызывать из кастомного оператора, чем обращаться к API напрямую.

Минимальные требования к кастомному хуку:

- Реализовать **`.__init__()`**, который вызывается при парсинге DAG.
- Наследоваться от **`BaseHook`** или другого существующего хука.

Во многих хуках есть метод **`.get_conn()`**, который оборачивает вызов `BaseHook.get_connection()` для получения данных из подключения Airflow. Часто `.get_conn()` вызывают в `.__init__()`. Ниже — минимальный рекомендуемый каркас для большинства кастомных хуков:

```python
from airflow.hooks.base import BaseHook


class MyHook(BaseHook):
    """
    Взаимодействие с <внешней системой>.
    :param my_conn_id: ID подключения к <внешней системе>
    """

    conn_name_attr = "my_conn_id"
    default_conn_name = "my_conn_default"
    conn_type = "general"
    hook_name = "MyHook"

    def __init__(
        self, my_conn_id: str = default_conn_name, *args, **kwargs
    ) -> None:
        super().__init__(*args, **kwargs)
        self.my_conn_id = my_conn_id
        self.get_conn()

    def get_conn(self):
        """Создаёт подключение к внешней системе."""
        conn_id = getattr(self, self.conn_name_attr)
        conn = self.get_connection(conn_id)
        return conn

    # добавьте методы для работы с внешней системой
```

## Импорт кастомных хуков и операторов

После определения хука или оператора их нужно сделать доступными для DAG. Регистрировать их как плагин Airflow не обязательно. Чтобы импортировать кастомный оператор или хук в DAG, файл с классом должен находиться в каталоге из **PYTHONPATH**. Подробнее: [Module management](https://airflow.apache.org/docs/apache-airflow/stable/administration-and-deployment/modules_management.html).

При использовании [Astro CLI](https://www.astronomer.io/docs/astro/cli/install-cli) файлы с кастомными операторами и хуками можно класть в каталог **`include`** проекта Astro. Имеет смысл создавать подкаталоги для навигации.

```text
.
├── .astro/
├── dags/
│   └── example_dag.py
├── include/
│   ├── custom_operators/
│   │   └── my_operator.py
│   └── custom_hooks/
│       └── my_hook.py
├── plugins/
├── tests/
├── .dockerignore
├── .env
├── .gitignore
├── .airflow_settings.yaml
├── Dockerfile
├── packages.txt
├── README.md
└── requirements.txt
```

Подробнее о рекомендуемой структуре: [Managing Airflow Code](../01.%20astronomer-basic/managing-airflow-code.md).

При такой структуре импорт в DAG может выглядеть так:

```python
from include.custom_operators.my_operator import MyOperator
from include.custom_hooks.my_hook import MyHook
```

## Пример реализации

Ниже определён класс **`MyBasicMathOperator`**. Он наследуется от BaseOperator и выполняет арифметику по двум числам и операции. Код сохраняется в каталоге `include` в файле `basic_math_operator.py`.

```python
from airflow.sdk.bases.operator import BaseOperator


class MyBasicMathOperator(BaseOperator):
    """
    Пример оператора для простой арифметики.
    :param first_number: первое число
    :param second_number: второе число
    :param operation: операция (+, -, *, /)
    """

    valid_operations = ("+", "-", "*", "/")
    template_fields = ("first_number", "second_number")

    def __init__(
        self,
        first_number: float,
        second_number: float,
        operation: str,
        *args,
        **kwargs,
    ):
        super().__init__(*args, **kwargs)
        self.first_number = first_number
        self.second_number = second_number
        self.operation = operation
        if self.operation not in self.valid_operations:
            raise ValueError(
                f"{self.operation} is not a valid operation. Choose one of {self.valid_operations}"
            )

    def execute(self, context):
        self.log.info(
            f"Equation: {self.first_number} {self.operation} {self.second_number}"
        )
        if self.operation == "+":
            res = self.first_number + self.second_number
            self.log.info(f"Result: {res}")
            return res
        if self.operation == "-":
            res = self.first_number - self.second_number
            self.log.info(f"Result: {res}")
            return res
        if self.operation == "*":
            res = self.first_number * self.second_number
            self.log.info(f"Result: {res}")
            return res
        if self.operation == "/":
            try:
                res = self.first_number / self.second_number
            except ZeroDivisionError:
                self.log.critical(
                    "If you have set up an equation where you are trying to divide by zero, you have done something WRONG. - Randall Munroe, 2006"
                )
                raise ZeroDivisionError
            self.log.info(f"Result: {res}")
            return res
```

В том же примере используется кастомный хук для подключения к CatFactAPI. Хук получает URL API из [подключения Airflow](../01.%20astronomer-basic/connections.md) и выполняет несколько запросов в цикле. Код можно положить в `include` в файл `cat_fact_hook.py`.

```python
"""Подключение к CatFactAPI."""

from airflow.hooks.base import BaseHook
import requests as re


class CatFactHook(BaseHook):
    """
    Взаимодействие с CatFactAPI.
    Подключается к API и получает факты о котах.
    :param cat_fact_conn_id: ID подключения с URL CatFactAPI.
    """

    conn_name_attr = "cat_conn_id"
    default_conn_name = "cat_conn_default"
    conn_type = "http"
    hook_name = "CatFact"

    def __init__(
        self, cat_fact_conn_id: str = default_conn_name, *args, **kwargs
    ) -> None:
        super().__init__(*args, **kwargs)
        self.cat_fact_conn_id = cat_fact_conn_id
        self.get_conn()

    def get_conn(self):
        """Подключение к CatFactAPI."""
        conn = self.get_connection(self.cat_fact_conn_id)
        return conn.host

    def log_cat_facts(self, number_of_cat_facts_needed: int = 1):
        """Пишет в лог от 1 до 10 фактов о котах."""
        if number_of_cat_facts_needed < 1:
            self.log.info(
                "You will need at least one catfact! Setting request number to 1."
            )
            number_of_cat_facts_needed = 1
        if number_of_cat_facts_needed > 10:
            self.log.info(
                f"{number_of_cat_facts_needed} are a bit many. Setting request number to 10."
            )
            number_of_cat_facts_needed = 10

        cat_fact_connection = self.get_conn()
        for i in range(number_of_cat_facts_needed):
            cat_fact = re.get(cat_fact_connection).json()
            self.log.info(cat_fact["fact"])
        return f"{i} catfacts written to the logs!"
```

Для работы хука нужно создать в Airflow подключение с ID `cat_fact_conn`, типом `HTTP` и полем Host `http://catfact.ninja/fact`.

Кастомный оператор и хук можно импортировать в DAG. У оператора заданы `template_fields` для `first_number` и `second_number`, поэтому в эти параметры можно подставлять значения из других задач через Jinja.

**TaskFlow:**

```python
from pendulum import datetime
from airflow.decorators import dag, task
from include.basic_math_operator import MyBasicMathOperator
from include.cat_fact_hook import CatFactHook


@dag(
    schedule_interval="@daily",
    start_date=datetime(2021, 1, 1),
    render_template_as_native_obj=True,
    catchup=False,
)
def my_math_cat_dag():
    add = MyBasicMathOperator(
        task_id="add",
        first_number=23,
        second_number=19,
        operation="+",
        doc_md="Addition Task.",
    )

    multiply = MyBasicMathOperator(
        task_id="multiply",
        first_number="{{ ti.xcom_pull(task_ids='add', key='return_value') }}",
        second_number=35,
        operation="-",
    )

    @task
    def use_cat_fact_hook(number):
        num_catfacts_needed = round(number)
        hook = CatFactHook("cat_fact_conn")
        hook.log_cat_facts(num_catfacts_needed)

    add >> multiply >> use_cat_fact_hook(multiply.output)


my_math_cat_dag()
```

**Традиционный вариант:**

```python
from pendulum import datetime
from airflow import DAG
from airflow.operators.python import PythonOperator
from include.basic_math_operator import MyBasicMathOperator
from include.cat_fact_hook import CatFactHook


def use_cat_fact_hook(number):
    num_catfacts_needed = round(number)
    hook = CatFactHook("cat_fact_conn")
    hook.log_cat_facts(num_catfacts_needed)


with DAG(
    dag_id="my_math_cat_dag",
    schedule_interval="@daily",
    start_date=datetime(2021, 1, 1),
    render_template_as_native_obj=True,
    catchup=False,
):
    add = MyBasicMathOperator(
        task_id="add",
        first_number=23,
        second_number=19,
        operation="+",
        doc_md="Addition Task.",
    )

    multiply = MyBasicMathOperator(
        task_id="multiply",
        first_number="{{ ti.xcom_pull(task_ids='add', key='return_value') }}",
        second_number=35,
        operation="-",
    )

    use_cat_fact_hook_task = PythonOperator(
        task_id="use_cat_fact_hook",
        python_callable=use_cat_fact_hook,
        op_args=[multiply.output],
    )

    add >> multiply >> use_cat_fact_hook_task
```

---

[← К содержанию](README.md) | [Хуки →](../01.%20astronomer-basic/hooks.md) | [Операторы →](../01.%20astronomer-basic/operators.md)
