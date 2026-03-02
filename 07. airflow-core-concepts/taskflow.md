# TaskFlow

*Добавлено в версии 2.0.*

Если большую часть DAG вы пишете обычным Python-кодом, а не операторами, TaskFlow API упрощает описание DAG без лишнего шаблонного кода — всё через декоратор `@task`.

TaskFlow сам передаёт входы и выходы между задачами через XCom и автоматически выстраивает зависимости: при вызове функции с декоратором TaskFlow в файле DAG вы получаете не результат выполнения, а объект, представляющий XCom с результатом (`XComArg`), который можно передать во входы следующих задач или операторов. Пример:

```python
from airflow.sdk import task
from airflow.providers.smtp.operators.smtp import EmailOperator

@task
def get_ip():
    return my_ip_service.get_main_ip()

@task(multiple_outputs=True)
def compose_email(external_ip):
    return {
        'subject': f'Server connected from {external_ip}',
        'body': f'Your server executing Airflow is connected from the external IP {external_ip}<br>'
    }

email_info = compose_email(get_ip())

EmailOperator(
    task_id='send_email_notification',
    to='example@example.com',
    subject=email_info['subject'],
    html_content=email_info['body']
)
```

Здесь три задачи: `get_ip`, `compose_email` и `send_email_notification`.

Первые две объявлены через TaskFlow: возвращаемое значение `get_ip` автоматически передаётся в `compose_email` — и данные идут через XCom, и зависимость «`compose_email` downstream от `get_ip`» задаётся автоматически.

`send_email_notification` — обычный оператор, но и он может использовать результат `compose_email` в параметрах и автоматически оказывается downstream от `compose_email`.

TaskFlow-функцию можно вызывать и с обычным значением или переменной — поведение ожидаемое (код внутри задачи выполнится только при запуске DAG; до этого значение `name` хранится как параметр задачи):

```python
@task
def hello_name(name: str):
    print(f'Hello {name}!')

hello_name('Airflow users')
```

Подробнее: [TaskFlow tutorial](https://airflow.apache.org/docs/apache-airflow/stable/tutorial/taskflow.html).

## Context (контекст)

Доступ к [переменным контекста](https://airflow.apache.org/docs/apache-airflow/stable/templates-ref.html#templates-variables) Airflow — через keyword-аргументы с нужными именами:

```python
from airflow.models.taskinstance import TaskInstance
from airflow.models.dagrun import DagRun


@task
def print_ti_info(task_instance: TaskInstance, dag_run: DagRun):
    print(f"Run ID: {task_instance.run_id}")  # Run ID: scheduled__2023-08-09T00:00:00+00:00
    print(f"Duration: {task_instance.duration}")  # Duration: 0.972019
    print(f"Dag Run queued at: {dag_run.queued_at}")  # 2023-08-10 00:00:01+02:20
```

Либо добавьте в сигнатуру задачи `**kwargs` — все переменные контекста Airflow будут в словаре `kwargs`:

```python
from airflow.models.taskinstance import TaskInstance
from airflow.models.dagrun import DagRun


@task
def print_ti_info(**kwargs):
    ti: TaskInstance = kwargs["task_instance"]
    print(f"Run ID: {ti.run_id}")  # Run ID: scheduled__2023-08-09T00:00:00+00:00
    print(f"Duration: {ti.duration}")  # Duration: 0.972019

    dr: DagRun = kwargs["dag_run"]
    print(f"Dag Run queued at: {dr.queued_at}")  # 2023-08-10 00:00:01+02:20
```

Полный список переменных контекста: [context variables](https://airflow.apache.org/docs/apache-airflow/stable/templates-ref.html#templates-variables).

## Logging (логирование)

Чтобы логировать из задач, используйте стандартный модуль `logging`:

```python
logger = logging.getLogger("airflow.task")
```

Записи такого логгера попадают в лог задачи.

## Передача произвольных объектов в качестве аргументов

*Добавлено в версии 2.5.0.*

TaskFlow передаёт данные между задачами через XCom, поэтому аргументы должны быть сериализуемы. «Из коробки» Airflow поддерживает встроенные типы (int, str и т.д.) и объекты, задекорированные `@dataclass` или `@attr.define`. Пример с `Asset` (декоратор `@attr.define`) и TaskFlow:

> **Примечание.** При использовании `Asset` он автоматически регистрируется как `inlet`, если передан как входной аргумент, и как `outlet`, если задача возвращает `Asset` или `list[Asset]`.

```python
import json
import pendulum
import requests

from airflow import Asset
from airflow.sdk import dag, task

SRC = Asset(
    "https://www.ncei.noaa.gov/access/monitoring/climate-at-a-glance/global/time-series/globe/land_ocean/ytd/12/1880-2022.json"
)
now = pendulum.now()


@dag(start_date=now, schedule="@daily", catchup=False)
def etl():
    @task()
    def retrieve(src: Asset) -> dict:
        resp = requests.get(url=src.uri)
        data = resp.json()
        return data["data"]

    @task()
    def to_fahrenheit(temps: dict[int, dict[str, float]]) -> dict[int, float]:
        ret: dict[int, float] = {}
        for year, info in temps.items():
            ret[year] = float(info["anomaly"]) * 1.8 + 32

        return ret

    @task()
    def load(fahrenheit: dict[int, float]) -> Asset:
        filename = "/tmp/fahrenheit.json"
        s = json.dumps(fahrenheit)
        f = open(filename, "w")
        f.write(s)
        f.close()

        return Asset(f"file:///{filename}")

    data = retrieve(SRC)
    fahrenheit = to_fahrenheit(data)
    load(fahrenheit)


etl()
```

### Кастомные объекты

Можно передавать свои типы. Обычно класс помечают `@dataclass` или `@attr.define`, и Airflow сам с ними работает. Если нужна своя сериализация, добавьте в класс метод `serialize()` и статический метод `deserialize(data: dict, version: int)`:

```python
from typing import ClassVar


class MyCustom:
    __version__: ClassVar[int] = 1

    def __init__(self, x):
        self.x = x

    def serialize(self) -> dict:
        return dict({"x": self.x})

    @staticmethod
    def deserialize(data: dict, version: int):
        if version > 1:
            raise TypeError(f"version > {MyCustom.version}")
        return MyCustom(data["x"])
```

### Версионирование объектов

Объекты, участвующие в сериализации, лучше версионировать. Для этого у класса задают `__version__: ClassVar[int] = <число>`. Airflow считает классы обратно совместимыми: версия 2 должна уметь десериализовать данные версии 1. Специфичную логику десериализации задают в `deserialize(data: dict, version: int)`.

> **Примечание.** Тип для `__version__` обязателен и должен быть `ClassVar[int]`.

## Сенсоры и TaskFlow API

*Добавлено в версии 2.5.0.*

Пример сенсора через TaskFlow API: [Using the TaskFlow API with Sensor operators](https://airflow.apache.org/docs/apache-airflow/stable/tutorial/taskflow.html#taskflow-using-sensors).

## История

TaskFlow API появился в Airflow 2.0. В DAG для более старых версий ту же идею реализовывали через `PythonOperator` с бо́льшим объёмом кода.

Подробнее о появлении и дизайне TaskFlow API: [AIP-31: "TaskFlow API" for clearer/simpler Dag definition](https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=148638736).

---

*Источник: [Airflow 3.1.7 — TaskFlow](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/taskflow.html). Перевод неофициальный.*
