# Правила Ruff — Airflow (AIR)

Ruff поддерживает более 900 правил линтинга, многие из которых вдохновлены такими инструментами, как Flake8, isort, pyupgrade и другими. Независимо от происхождения правила, Ruff перереализует каждое правило в Rust как встроенную возможность.

По умолчанию Ruff включает правила категории `F` из Flake8 вместе с подмножеством правил `E`, исключая стилистические правила, пересекающиеся с использованием форматтера (например `ruff format` или Black).

Подробнее см. [Airflow на PyPI](https://pypi.org/project/apache-airflow/).

## Легенда

- 🧪 Правило нестабильно и находится в режиме [«preview»](https://docs.astral.sh/ruff/faq/#what-is-preview).
- ⚠️ Правило устарело и будет удалено в будущем релизе.
- ❌ Правило удалено — доступна только документация.
- 🛠️ Правило автоматически исправляется опцией командной строки `--fix`.

Все правила, не помеченные как preview, deprecated или removed, считаются стабильными.

## Airflow (AIR)

| Код | Название | Сообщение |
| --- | --- | --- |
| AIR001 🛠️ | [airflow-variable-name-task-id-mismatch](#air001-airflow-variable-name-task-id-mismatch) | Имя переменной задачи должно совпадать с `task_id`: «{task_id}» |
| AIR002 | [airflow-dag-no-schedule-argument](#air002-airflow-dag-no-schedule-argument) | У `DAG` или `@dag` должен быть явный аргумент `schedule` |
| AIR301 🛠️ | [airflow3-removal](#air301-airflow3-removal) | `{deprecated}` удалён в Airflow 3.0 |
| AIR302 🛠️ | [airflow3-moved-to-provider](#air302-airflow3-moved-to-provider) | `{deprecated}` перенесён в провайдер `{provider}` в Airflow 3.0 |
| AIR303 🧪 | [airflow3-incompatible-function-signature](#air303-airflow3-incompatible-function-signature) | Сигнатура `{function_name}` изменена в Airflow 3.0 |
| AIR311 🛠️ | [airflow3-suggested-update](#air311-airflow3-suggested-update) | `{deprecated}` удалён в Airflow 3.0; в Airflow 3.0 пока работает, но ожидается удаление в будущей версии |
| AIR312 🛠️ | [airflow3-suggested-to-move-to-provider](#air312-airflow3-suggested-to-move-to-provider) | `{deprecated}` устарел и перенесён в провайдер `{provider}` в Airflow 3.0; в Airflow 3.0 пока работает, но ожидается удаление в будущей версии |
| AIR321 🧪🛠️ | [airflow31-moved](#air321-airflow31-moved) | `{deprecated}` перенесён в Airflow 3.1 |

---

## AIR001 — airflow-variable-name-task-id-mismatch

Добавлено в [v0.0.271](https://github.com/astral-sh/ruff/releases/tag/v0.0.271). Правило из линтера [Airflow](https://docs.astral.sh/ruff/rules/#airflow-air).

### Что проверяет

Проверяет, что имя переменной задачи совпадает со значением `task_id` для операторов Airflow.

### Почему это плохо

При инициализации оператора Airflow для единообразия имя переменной должно совпадать со значением `task_id`. Так проще отслеживать поток DAG.

### Пример

Плохо:

```python
from airflow.operators import PythonOperator


incorrect_name = PythonOperator(task_id="my_task")
```

Правильно:

```python
from airflow.operators import PythonOperator


my_task = PythonOperator(task_id="my_task")
```

---

## AIR002 — airflow-dag-no-schedule-argument

Добавлено в [0.13.0](https://github.com/astral-sh/ruff/releases/tag/0.13.0). Правило из линтера [Airflow](https://docs.astral.sh/ruff/rules/#airflow-air).

### Что проверяет

Ищет использование класса `DAG()` или декоратора `@dag()` без явного параметра `schedule` (или `schedule_interval` в Airflow 1).

### Почему это плохо

Значение по умолчанию параметра `schedule` в Airflow 2 и `schedule_interval` в Airflow 1 — `timedelta(days=1)`, что почти никогда не то, что нужно пользователю. В Airflow 3 значение по умолчанию изменено на `None`, и существующие DAG с неявным значением по умолчанию перестанут работать.

Если у DAG нет явного аргумента `schedule` / `schedule_interval`, в Airflow 2 запуск планируется каждый день (в момент, определяемый `start_date`). Такой DAG в Airflow 3 вообще не будет планироваться, без исключений и других сообщений для пользователя.

### Пример

Плохо:

```python
from airflow import DAG


# Использование неявного расписания по умолчанию.
dag = DAG(dag_id="my_dag")
```

Правильно:

```python
from datetime import timedelta

from airflow import DAG


dag = DAG(dag_id="my_dag", schedule=timedelta(days=1))
```

---

## AIR301 — airflow3-removal

Добавлено в [0.13.0](https://github.com/astral-sh/ruff/releases/tag/0.13.0). Правило из линтера [Airflow](https://docs.astral.sh/ruff/rules/#airflow-air). Исправление иногда доступно.

### Что проверяет

Ищет использование устаревших функций и значений Airflow.

### Почему это плохо

В Airflow 3.0 удалены различные устаревшие функции, члены и другие значения. У части есть современные замены. Другие сочтены слишком узкими и нецелесообразными для дальнейшей поддержки в Airflow.

### Пример

Плохо:

```python
from airflow.utils.dates import days_ago


yesterday = days_ago(today, 1)
```

Правильно:

```python
from datetime import timedelta


yesterday = today - timedelta(days=1)
```

---

## AIR302 — airflow3-moved-to-provider

Добавлено в [0.13.0](https://github.com/astral-sh/ruff/releases/tag/0.13.0). Правило из линтера [Airflow](https://docs.astral.sh/ruff/rules/#airflow-air). Исправление иногда доступно.

### Что проверяет

Ищет использование функций и значений Airflow, перенесённых в провайдеры (например `apache-airflow-providers-fab`).

### Почему это плохо

В Airflow 3.0 различные устаревшие функции, члены и другие значения перенесены в провайдеры. Пользователю нужно установить соответствующий провайдер и заменить исходное использование на использование из провайдера.

### Пример

Плохо:

```python
from airflow.auth.managers.fab.fab_auth_manager import FabAuthManager

fab_auth_manager_app = FabAuthManager().get_fastapi_app()
```

Правильно:

```python
from airflow.providers.fab.auth_manager.fab_auth_manager import FabAuthManager

fab_auth_manager_app = FabAuthManager().get_fastapi_app()
```

---

## AIR303 — airflow3-incompatible-function-signature

Preview (с [0.14.11](https://github.com/astral-sh/ruff/releases/tag/0.14.11)). Правило из линтера [Airflow](https://docs.astral.sh/ruff/rules/#airflow-air). Правило нестабильно и в режиме [preview](https://docs.astral.sh/ruff/preview/). Для использования нужен флаг `--preview`.

### Что проверяет

Ищет вызовы функций Airflow, которые в Airflow 3.0 приведут к ошибке времени выполнения из-за изменений сигнатуры: например функции, перешедшие на приём только keyword-аргументов, перестановка параметров или изменение типов параметров.

### Почему это плохо

В Airflow 3.0 изменены сигнатуры функций. Код, работавший в Airflow 2.x, в Airflow 3.0 вызовет ошибку времени выполнения, если его не обновить.

### Пример

Плохо:

```python
from airflow.lineage.hook import HookLineageCollector

collector = HookLineageCollector()
# Передача позиционных аргументов вызовет ошибку времени выполнения в Airflow 3.0
collector.create_asset("s3://bucket/key")
```

Правильно:

```python
from airflow.lineage.hook import HookLineageCollector

collector = HookLineageCollector()
# Передача аргументов как keyword-аргументов вместо позиционных
collector.create_asset(uri="s3://bucket/key")
```

---

## AIR311 — airflow3-suggested-update

Добавлено в [0.13.0](https://github.com/astral-sh/ruff/releases/tag/0.13.0). Правило из линтера [Airflow](https://docs.astral.sh/ruff/rules/#airflow-air). Исправление иногда доступно.

### Что проверяет

Ищет использование устаревших функций и значений Airflow, для которых ещё есть слой совместимости.

### Почему это плохо

В Airflow 3.0 удалены различные устаревшие функции, члены и другие значения. У части есть современные замены. Другие сочтены слишком узкими для дальнейшей поддержки в Airflow. Хотя эти символы в Airflow 3.0 пока работают, ожидается их удаление в будущей версии. По возможности пользователям стоит заменить удалённую функциональность новыми альтернативами.

### Пример

Плохо:

```python
from airflow import Dataset


Dataset(uri="test://test/")
```

Правильно:

```python
from airflow.sdk import Asset


Asset(uri="test://test/")
```

---

## AIR312 — airflow3-suggested-to-move-to-provider

Добавлено в [0.13.0](https://github.com/astral-sh/ruff/releases/tag/0.13.0). Правило из линтера [Airflow](https://docs.astral.sh/ruff/rules/#airflow-air). Исправление иногда доступно.

### Что проверяет

Ищет использование функций и значений Airflow, перенесённых в провайдеры, но с сохранённым слоем совместимости (например `apache-airflow-providers-standard`).

### Почему это плохо

В Airflow 3.0 различные устаревшие функции, члены и другие значения перенесены в провайдеры. Хотя эти символы в Airflow 3.0 пока работают, ожидается их удаление в будущей версии. Рекомендуется установить соответствующий провайдер и заменить исходное использование на использование из провайдера.

### Пример

Плохо:

```python
from airflow.operators.python import PythonOperator


def print_context(ds=None, **kwargs):
    print(kwargs)
    print(ds)


print_the_context = PythonOperator(
    task_id="print_the_context", python_callable=print_context
)
```

Правильно:

```python
from airflow.providers.standard.operators.python import PythonOperator


def print_context(ds=None, **kwargs):
    print(kwargs)
    print(ds)


print_the_context = PythonOperator(
    task_id="print_the_context", python_callable=print_context
)
```

---

## AIR321 — airflow31-moved

Preview (с [0.15.1](https://github.com/astral-sh/ruff/releases/tag/0.15.1)). Правило из линтера [Airflow](https://docs.astral.sh/ruff/rules/#airflow-air). Исправление иногда доступно. Правило нестабильно и в режиме [preview](https://docs.astral.sh/ruff/preview/). Для использования нужен флаг `--preview`.

### Что проверяет

Ищет использование устаревших или перенесённых функций и значений Airflow в Airflow 3.1.

### Почему это плохо

В Airflow 3.1 различные функции, члены и другие значения устарели или перенесены.

### Пример

Плохо:

```python
from airflow.utils.timezone import convert_to_utc
from datetime import datetime

convert_to_utc(datetime.now())
```

Правильно:

```python
from airflow.sdk.timezone import convert_to_utc
from datetime import datetime

convert_to_utc(datetime.now())
```
