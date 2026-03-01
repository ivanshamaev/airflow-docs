# Общий код между проектами (Sharing code)

> Эта страница ещё не обновлена под Airflow 3. Показанные концепции актуальны, но часть кода может потребовать правок. При запуске примеров обновите импорты и учтите возможные breaking changes.
>
> Информация

После [настройки проекта Astro](https://www.astronomer.io/docs/astro/cli/develop-project), когда вы начинаете реализовывать пайплайны, выносить переиспользуемые Python-функции и масштабировать операции, появляется несколько деплоев Airflow. Как переиспользовать код между проектами? В этом руководстве — варианты переиспользования кода и их плюсы и минусы.

Рассматриваются три варианта — от простой реализации с ограниченным переиспользованием до полноценной реализации с максимальным переиспользованием:

| Решение | Когда использовать |
| --- | --- |
| Общий Python-код в одном файле | Когда код нужен только в одном скрипте |
| Общий Python-код в папке `/include` | Когда код нужен в нескольких скриптах в рамках одного Git-репозитория |
| Общий Python-код в отдельном проекте как Python-пакет | Когда код нужен в нескольких Git-проектах |

Ниже в каждом разделе используется один и тот же DAG: он обращается к БД и возвращает результаты. Задачи в DAG дважды выполняют одну и ту же бизнес-логику. Обе функции (`_get_locations()` и `_get_purchases()`) создают клиент БД, выполняют запрос и возвращают результат; отличается только текст запроса. Любое изменение логики подключения к БД потребует правок в двух местах. При копировании таких функций для дополнительных запросов одна и та же логика будет дублироваться, и менять подключение к БД придётся во многих местах.

```python
import datetime

from airflow import DAG
from airflow.operators.python import PythonOperator

with DAG(dag_id="example", schedule=None, start_date=datetime.datetime(2023, 1, 1)):
    def _get_locations():
        # Псевдокод:
        client = db.client()
        query = "SELECT store_id, city FROM stores"
        result = client.execute(query)
        return result

    def _get_purchases():
        # Псевдокод:
        client = db.client()
        query = "SELECT customer_id, string_agg(store_id, ',') FROM customers GROUP BY customer_id"
        result = client.execute(query)
        return result

    PythonOperator(task_id="get_locations", python_callable=_get_locations)
    PythonOperator(task_id="get_purchases", python_callable=_get_purchases)
```

## Общий Python-код в одном файле

Чтобы не дублировать одну и ту же бизнес-логику и иметь единый способ запросов к БД, можно вынести код в одну функцию в том же DAG-файле:

```python
def query_db(query):
    # Псевдокод:
    client = db.client()
    result = client.execute(query)
    return result
```

Функция принимает аргумент `query`, а логика подключения к БД задаётся один раз для любого запроса. Так не нужно поддерживать одну и ту же логику в нескольких местах. Чтобы использовать функцию, укажите её в DAG:

```python
import datetime

from airflow import DAG
from airflow.operators.python import PythonOperator

def query_db(query):
    # Псевдокод:
    client = db.client()
    result = client.execute(query)
    return result

with DAG(dag_id="example", schedule=None, start_date=datetime.datetime(2023, 1, 1)):
    PythonOperator(task_id="get_locations", python_callable=query_db, op_kwargs={"query": "SELECT store_id, city FROM stores"})
    PythonOperator(task_id="get_purchases", python_callable=query_db, op_kwargs={"query": "SELECT customer_id, string_agg(store_id, ',') FROM customers GROUP BY customer_id"})
```

Реализация проста: одна функция используется в нескольких задачах Airflow в одном скрипте. Изменение логики подключения к БД потребует одной правки вместо двух. Но функцию нельзя переиспользовать в другом скрипте (без импорта). Вариант для нескольких скриптов — в следующем разделе.

## Общий Python-код в папке `/include`

Чтобы переиспользовать код в нескольких скриптах, он должен находиться в общем месте. В образе Astro Runtime для этого есть папка `/include`:

Сохраните функцию в отдельном файле, например `/include/db.py`:

```python
def query_db(query):
    # Псевдокод:
    client = db.client()
    result = client.execute(query)
    return result
```

Импортируйте функцию в DAG:

```python
import datetime

from airflow import DAG
from airflow.operators.python import PythonOperator

from include.db import query_db

with DAG(dag_id="example", schedule=None, start_date=datetime.datetime(2023, 1, 1)):
    PythonOperator(task_id="get_locations", python_callable=query_db, op_kwargs={"query": "SELECT store_id, city FROM stores"})
    PythonOperator(task_id="get_purchases", python_callable=query_db, op_kwargs={"query": "SELECT customer_id, string_agg(store_id, ',') FROM customers GROUP BY customer_id"})
```

Плюс такого подхода: функцию `query_db` можно импортировать из нескольких скриптов в рамках одного Git-репозитория.

## Общий Python-код в отдельном проекте как Python-пакет

Иногда один и тот же код нужен в разных деплоях Airflow. Например, на платформу Astronomer переходят несколько команд, у каждой свой репозиторий. Код из папки `/include` в таком случае не подойдёт — он лежит в другом Git-репозитории.

Чтобы переиспользовать код в нескольких проектах, его нужно хранить в отдельном Git-репозитории, доступном этим проектам. Удобный способ — оформить репозиторий как Python-пакет. Настройка сложнее, зато несколько команд с разными репозиториями могут поддерживать один общий код. Пример пакета: [custom-package-demo](https://github.com/astronomer/custom-package-demo).

Вариантов разработки, сборки и публикации Python-пакета много; ниже — общие шаги. Подробнее: [Structuring your project](https://docs.python-guide.org/writing/structure) и [Packaging Python projects](https://packaging.python.org/en/latest/tutorials/packaging-projects).

Настройка своего Python-пакета в общих чертах:

1. Проверьте пакет, установив его в проект через `requirements.txt`.
2. Убедитесь, что сборка и публикация работают, собрав и опубликовав первую версию пакета.
3. Настройте CI/CD для тестов, сборки и публикации пакета. Пример workflow для GitHub Actions: [custom package demo](https://github.com/astronomer/custom-package-demo/tree/main/.github/workflows).
4. Создайте папку для тестов, например `tests`.
5. Создайте папку для кода, например `my_company_airflow`.
6. Добавьте файл `pyproject.toml` — конфигурацию с требованиями к сборке проекта. Пример: [здесь](https://github.com/astronomer/custom-package-demo/blob/main/pyproject.toml).
7. Создайте отдельный Git-репозиторий для общего кода.

После этого добавьте общий код в пакет, чтобы другие проекты могли его использовать. Код должен находиться в модуле пакета, например `my_company_airflow/db.py`:

```python
def query_db(query):
    # Псевдокод:
    client = db.client()
    result = client.execute(query)
    return result
```

После установки пакета функцию можно импортировать в DAG так:

```python
import datetime

from airflow import DAG
from airflow.operators.python import PythonOperator

from my_company_airflow.db import query_db

with DAG(dag_id="example", schedule=None, start_date=datetime.datetime(2023, 1, 1)):
    PythonOperator(task_id="get_locations", python_callable=query_db, op_kwargs={"query": "SELECT store_id, city FROM stores"})
    PythonOperator(task_id="get_purchases", python_callable=query_db, op_kwargs={"query": "SELECT customer_id, string_agg(store_id, ',') FROM customers GROUP BY customer_id"})
```

Выше — общий порядок настройки; детали зависят от окружения и практик в организации. Рекомендации по своему пакету:

- Сначала доведите до конца CI/CD (сборка, тесты, публикация), затем добавляйте прикладной код.
- С самого начала задайте стандарты: линтер (например, Flake8), форматирование (например, Black).
- Определите, кто отвечает за общий Git-репозиторий.
- Решите, как распространять пакет: нужен ли внутренний репозиторий для Python-пакетов ([Artifactory](https://jfrog.com/artifactory), [devpi](https://www.devpi.net/) и т.п.)?

## Планирование на будущее

В руководстве рассмотрены варианты переиспользования кода: от кода в одном файле до кода в нескольких Git-репозиториях. Если сейчас у вас один репозиторий и один деплой на Astronomer, но в перспективе планируются несколько команд, лучше сразу заложить общий код в отдельный Git-репозиторий. Ввести стандарты и лучшие практики с самого начала проще, чем менять подход потом.

---

[← Setup/teardown](setup-teardown.md) | [К содержанию](README.md) | [Синхронное выполнение →](synchronous-execution.md)
