# Тестирование Airflow (Testing Airflow)

> Эта страница ещё не обновлена под Airflow 3. Показанные концепции актуальны, но часть кода может потребовать правок. При запуске примеров обновите импорты и учтите возможные breaking changes.
>
> Информация

Эффективное тестирование DAG требует понимания их структуры и связи с другим кодом и данными в окружении. В этом руководстве — виды проверок DAG (валидация), юнит-тесты и где искать информацию о проверках качества данных.

> - Вебинар: [How to easily test your Airflow DAGs with the new dag.test() function](https://www.astronomer.io/events/webinars/how-to-easily-test-your-airflow-dags-with-the-new-dag-test-function/).
>
> По теме есть и другие материалы. См. также:
>
> Другие способы изучить тему

## Необходимая база

Чтобы получить максимум от руководства, нужно понимать:

- Основы Airflow и [Astro CLI](https://www.astronomer.io/docs/astro/cli/install-cli). См. [Get started with Airflow](https://www.astronomer.io/docs/learn/get-started-with-airflow).
- CI/CD для Python-скриптов. См. [Continuous Integration with Python: An Introduction](https://realpython.com/python-continuous-integration/).
- Хотя бы один Python test runner. В руководстве в основном используется [`pytest`](https://docs.pytest.org/en/stable/index.html), подойдут и другие: [`nose2`](https://docs.nose2.io/en/latest/getting_started.html), [`unittest`](https://docs.python.org/3/library/unittest.html).
- Основы тестирования в Python. См. [Getting Started with Testing in Python](https://realpython.com/python-testing/).

## Написание тестов валидации DAG

Тесты валидации DAG проверяют, что DAG удовлетворяют заданным критериям. С их помощью можно:

- Дать продвинутым пользователям возможность тестировать DAG из CLI.
- Автоматически тестировать DAG в CI/CD-пайплайне.
- Систематически проверять и обеспечивать выполнение собственных требований к DAG.
- Разрабатывать DAG без локального окружения Airflow.

Минимум — запускать тесты валидации для проверки [ошибок импорта](https://www.astronomer.io/docs/learn/testing-airflow#prevent-import-errors). Дополнительно можно проверять свою логику: например, что у всех DAG в инстансе задано `catchup=False` или что в DAG используются только `tags` из разрешённого списка.

Тесты валидации DAG применяются ко всем DAG в окружении Airflow, поэтому достаточно одного набора тестов.

### Типичные тесты валидации DAG

Ниже — распространённые типы тестов с полными примерами кода.

#### Проверка ошибок импорта

Частая проверка — наличие ошибок импорта. Тест быстрее, чем поднимать окружение Airflow и смотреть ошибки в UI. В примере ниже `get_import_errors` читает атрибут `.import_errors` текущего [DagBag](https://airflow.apache.org/docs/apache-airflow/stable/_api/airflow/models/dagbag/index.html#).

```python
import os
import pytest
from airflow.models import DagBag

def get_import_errors():
    """
    Формирует кортежи для ошибок импорта в DagBag
    """
    dag_bag = DagBag(include_examples=False)

    def strip_path_prefix(path):
        return os.path.relpath(path, os.environ.get("AIRFLOW_HOME"))

    # добавляем "(None,None)", чтобы тестовый объект всегда создавался, даже если это no op
    return [(None, None)] + [
        (strip_path_prefix(k), v.strip()) for k, v in dag_bag.import_errors.items()
    ]

@pytest.mark.parametrize(
    "rel_path,rv", get_import_errors(), ids=[x[0] for x in get_import_errors()]
)
def test_file_imports(rel_path, rv):
    """Проверка ошибок импорта в файле"""
    if rel_path and rv:
        raise Exception(f"{rel_path} failed to import with message \n {rv}")
```

#### Проверка собственных требований к коду

В DAG Airflow можно подключать плагины и свой код. Команды часто задают правила и стандарты для DAG и пишут тесты валидации, чтобы эти стандарты соблюдались.

В примере ниже тест проверяет, что у всех DAG параметр `tags` содержит только значения из списка `APPROVED_TAGS`.

```python
import os
import pytest
from airflow.models import DagBag

def get_dags():
    """
    Формирует кортежи (dag_id, объект DAG) из DagBag
    """
    dag_bag = DagBag(include_examples=False)

    def strip_path_prefix(path):
        return os.path.relpath(path, os.environ.get("AIRFLOW_HOME"))

    return [(k, v, strip_path_prefix(v.fileloc)) for k, v in dag_bag.dags.items()]

APPROVED_TAGS = {"customer_success", "op_analytics", "product"}

@pytest.mark.parametrize(
    "dag_id,dag,fileloc", get_dags(), ids=[x[2] for x in get_dags()]
)
def test_dag_tags(dag_id, dag, fileloc):
    """
    Проверка, что DAG имеет теги и они из утверждённого списка
    """
    assert dag.tags, f"{dag_id} в {fileloc} не имеет тегов"
    if APPROVED_TAGS:
        assert not set(dag.tags) - APPROVED_TAGS
```

> Атрибуты и методы модели `dag` описаны в [документации Airflow](https://airflow.apache.org/docs/apache-airflow/stable/stable-rest-api-ref.html#tag/DAG).
>
> Совет

Требования можно задавать и на уровне задач через атрибут `tasks` у модели `dag` (список всех задач DAG). Тест ниже проверяет, что у всех DAG есть хотя бы одна задача и у всех задач задано `trigger_rule="all_success"`.

```python
@pytest.mark.parametrize(
    "dag_id,dag,fileloc", get_dags(), ids=[x[2] for x in get_dags()]
)
def test_dag_tags(dag_id, dag, fileloc):
    """
    Проверка, что у всех DAG есть задачи и у всех задач trigger_rule = all_success
    """
    assert dag.tasks, f"{dag_id} в {fileloc} не имеет задач"
    for task in dag.tasks:
        t_rule = task.trigger_rule
        assert (
            t_rule == "all_success"
        ), f"{task} в {dag_id} имеет trigger rule {t_rule}"
```

## Запуск тестов валидации DAG

Запускать тесты валидации DAG можно разными способами с любым Python test runner. Ниже — самые распространённые. Если вы только начинаете тестировать DAG в Airflow, быстрый старт — команды Astro CLI.

### Airflow CLI

В Airflow CLI есть две команды для локального тестирования:

- [**`airflow tasks test`**](https://airflow.apache.org/docs/apache-airflow/stable/cli-and-env-variables-ref.html#test_repeat1): запуск одного экземпляра задачи без проверки зависимостей и без записи результата в метаданные БД.
- [**`airflow dags test`**](https://airflow.apache.org/docs/apache-airflow/stable/cli-and-env-variables-ref.html#test): по DAG ID и дате выполнения записывает результат одного DAG run в метаданные БД. Удобно для тестирования DAG и ручного запуска DAG run из командной строки. В Airflow 2.10+ можно пропускать задачи по их task id флагом `--mark-success-pattern` и запускать их с настроенным исполнителем через `--use-executor`.

В Astro CLI все команды Airflow CLI можно выполнять через [**`astro dev run`**](https://www.astronomer.io/docs/astro/cli/astro-dev-run). Например, запуск `airflow dags test` для DAG `my_dag` на дату выполнения `2023-01-29`:

```sh
astro dev run dags test my_dag '2023-01-29'
```

### Astro CLI

В Astro CLI входят команды для упрощения типичных сценариев тестирования. См. [Test your Astro project locally](https://www.astronomer.io/docs/astro/cli/test-your-astro-project-locally).

### Тестирование DAG в CI/CD-пайплайне

Код Airflow можно тестировать и деплоить через CI/CD. Установив Astro CLI в CI/CD, можно тестировать DAG перед выкладкой в продакшен. Примеры настройки: [set up CI/CD](https://www.astronomer.io/docs/astro/set-up-ci-cd).

> Клиенты Astronomer могут использовать интеграцию Astro с GitHub: автоматический деплой кода из репозитория GitHub в деплой Astro и просмотр метаданных Git в UI Astro. См. [Deploy code with the Astro GitHub integration](https://www.astronomer.io/docs/astro/deploy-github-integration).
>
> Информация

## Тестовые данные и файлы для локального тестирования

Папка `include` проекта Astro подходит для файлов локального тестирования: тестовые данные, конфиг dbt и т.д. Содержимое `include` попадает в деплой на Astro, но не парсится Airflow, поэтому такие файлы не нужно добавлять в `.airflowignore`, чтобы исключить их из парсинга.

При локальном запуске Airflow изменения подхватываются после обновления UI.

## Интерактивная отладка с dag.test()

Метод **`dag.test()`** запускает все задачи DAG в одном сериализованном Python-процессе без планировщика Airflow. С ним удобнее итерироваться и использовать отладку в IDE при разработке DAG.

Эта возможность заменяет устаревший DebugExecutor. Подробнее: [документация Airflow](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/executor/debug.html).

### Предварительные требования

В окружении для тестов должны быть:

- Инициализированная [метаданная БД Airflow](https://www.astronomer.io/docs/learn/airflow-database), если DAG использует её (например, XCom). БД создаётся при первом запуске Airflow в окружении. Проверка: `airflow db check`, инициализация новой БД: `airflow db migrate` (в версиях до 2.7 — `airflow db init`).
- Все провайдеры, которые использует DAG.
- [Airflow 2.5.0](https://airflow.apache.org/docs/apache-airflow/stable/start.html) или новее. Версию можно проверить командой `airflow version`.

Рекомендуется ставить зависимости и тестировать DAG в [virtualenv](https://virtualenv.pypa.io/en/latest/), чтобы избежать конфликтов в локальном окружении.

### Настройка

Для использования `dag.test()` достаточно добавить несколько строк в конец файла DAG. При классическом контексте DAG вызывайте `dag.test()` после объявления DAG. При декораторе `@dag` присвойте результат вызова функции DAG переменной и вызовите метод у этого объекта.

**Traditional**

```python
from airflow.models.dag import DAG
from pendulum import datetime
from airflow.operators.empty import EmptyOperator

with DAG(
    dag_id="simple_classic_dag",
    start_date=datetime(2023, 1, 1),
    schedule="@daily",
    catchup=False,
) as dag:  # присвоение контекста объекту обязательно для dag.test()

    t1 = EmptyOperator(task_id="t1")

if __name__ == "__main__":
    dag.test()
```

**Decorator**

```python
from airflow.decorators import dag
from pendulum import datetime
from airflow.operators.empty import EmptyOperator

@dag(
    start_date=datetime(2023, 1, 1),
    schedule="@daily",
    catchup=False,
)
def my_dag():

    t1 = EmptyOperator(task_id="t1")

dag_object = my_dag()

if __name__ == "__main__":
    dag_object.test()
```

Метод `.test()` можно запускать с обычными инструментами отладки:

- [Отладчик Python (pdb)](https://docs.python.org/3/library/pdb.html) и встроенная функция [`breakpoint()`](https://docs.python.org/3/library/functions.html#breakpoint): запуск `dag.test()` из командной строки через `python <путь_к_файлу_dag>`.
- [PyCharm](https://www.jetbrains.com/help/pycharm/debugging-your-first-python-application.html).
- [VSCode](https://code.visualstudio.com/docs/editor/debugging).

### Использование dag.test() с Astro CLI

Если вы используете только Astro CLI и пакет `airflow` локально не установлен, отладку с `dag.test()` всё равно можно выполнять: запустите `astro dev start`, зайдите в контейнер планировщика через `astro dev bash -s` и выполните `python <путь_к_файлу_dag>` внутри контейнера. В отличие от работы с базовым пакетом `airflow`, для такого тестирования нужно поднять полное окружение Airflow.

### Переменные и подключения в dag.test()

Чтобы отлаживать DAG в более реалистичном окружении, в `dag.test()` можно передать:

- Конфигурацию DAG в виде словаря.
- Переменные Airflow в виде файла `.yaml`.
- [Подключения Airflow](https://www.astronomer.io/docs/learn/connections) в виде файла `.yaml`.
- `execution_date` в виде объекта `pendulum.datetime`.

Это удобно для проверки DAG на разных датах или с разными подключениями и конфигурациями. Пример передачи параметров в `dag.test()`:

```python
from pendulum import datetime

if __name__ == "__main__":
    conn_path = "connections.yaml"
    variables_path = "variables.yaml"
    my_conf_var = 23

    dag.test(
        execution_date=datetime(2023, 1, 29),
        conn_file_path=conn_path,
        variable_file_path=variables_path,
        run_conf={"my_conf_var": my_conf_var},
    )
```

Файл `connections.yaml` должен перечислять подключения и их свойства, например:

```yaml
my_aws_conn:
  conn_type: amazon
  login: <your-AWS-key>
  password: <your-AWS-secret>
  conn_id: my_aws_conn
```

В `variables.yaml` переменные задаются парами `key` и `value`:

```yaml
my_variable:
  key: my_variable
  value: 42
```

> По умолчанию dag.test() выполняет задачи без исполнителя. В Airflow 2.10+ можно запускать задачи с настроенным исполнителем, задав в dag.test() параметр `use_executor=True`.
>
> Информация

### Пропуск задач при использовании dag.test()

В Airflow 2.10 добавлена возможность не выполнять задачи, чей id совпадает с заданным [regex](https://en.wikipedia.org/wiki/Regular_expression)-шаблоном при вызове dag.test(). Это полезно, когда в DAG есть сенсоры, которые при тесте нужно пропускать.

```python
if __name__ == "__main__":

    dag.test(
        # новое в Airflow 2.10
        mark_success_pattern="sensor.*",  # regex task id для автоматической пометки как success
    )
```

## Юнит-тестирование

[Юнит-тестирование](https://en.wikipedia.org/wiki/Unit_testing) — метод тестирования, при котором небольшие части кода проверяются по отдельности. Цель — вынести тестируемую логику в небольшие функции с понятными именами. Например:

```python
def test_function_returns_5():
    assert my_function(input) == 5
```

В контексте Airflow юнит-тесты можно писать для любой части DAG, но чаще всего их применяют к хукам и операторам. Все хуки, операторы и пакеты провайдеров в Airflow проходят юнит-тесты перед мержем. Пример: [AWS S3Hook](https://registry.astronomer.io/providers/amazon/modules/s3hook) и [связанные юнит-тесты](https://github.com/apache/airflow/tree/main/providers/amazon/tests/system/amazon/aws).

При использовании своих хуков и операторов Astronomer рекомендует покрывать их логику и поведение юнит-тестами. В примере ниже кастомный оператор проверяет, чётное ли число:

```python
from airflow.models import BaseOperator

class EvenNumberCheckOperator(BaseOperator):
    def __init__(self, my_operator_param, *args, **kwargs):
        self.operator_param = my_operator_param
        super().__init__(*args, **kwargs)

    def execute(self, context):
        if self.operator_param % 2:
            return True
        else:
            return False
```

Файл `test_evencheckoperator.py` с юнит-тестами может выглядеть так:

```python
import unittest
from datetime import datetime
from airflow.models.dag import DAG
from airflow.models import TaskInstance

DEFAULT_DATE = datetime(2021, 1, 1)

class EvenNumberCheckOperator(unittest.TestCase):
    def setUp(self):
        super().setUp()
        self.dag = DAG(
            "test_dag", default_args={"owner": "airflow", "start_date": DEFAULT_DATE}
        )
        self.even = 10
        self.odd = 11

    def test_even(self):
        """Проверка, что EvenNumberCheckOperator возвращает True для 10."""
        task = EvenNumberCheckOperator(
            my_operator_param=self.even, task_id="even", dag=self.dag
        )
        ti = TaskInstance(task=task, execution_date=datetime.now())
        result = task.execute(ti.get_template_context())
        assert result is True

    def test_odd(self):
        """Проверка, что EvenNumberCheckOperator возвращает False для 11."""
        task = EvenNumberCheckOperator(
            my_operator_param=self.odd, task_id="odd", dag=self.dag
        )
        ti = TaskInstance(task=task, execution_date=datetime.now())
        result = task.execute(ti.get_template_context())
        assert result is False
```

Если в DAG есть PythonOperator с вашими функциями, для этих функций тоже стоит писать юнит-тесты.

В продакшене юнит-тесты обычно автоматизируют в CI/CD: тесты запускает CI, при ошибках процесс деплоя останавливается.

### Моки (Mocking)

Мокирование — имитация внешней системы, набора данных или другого объекта. Например, мок нужен, когда вы тестируете подключение без доступа к метаданным БД, или когда тестируете оператор, вызывающий внешний сервис по API, но не хотите реально вызывать сервис для простого теста.

Мокирование широко используется в [тестах Airflow](https://github.com/apache/airflow/tree/main/airflow-core/tests). В статье [Testing and debugging Apache Airflow](https://godatadriven.com/blog/testing-and-debugging-apache-airflow/) разобраны моки в Airflow — хорошая отправная точка.

## Проверки качества данных

Тестирование DAG проверяет соответствие кода требованиям. Но даже при идеальном коде пайплайны могут ломаться или работать хуже из-за качества данных. Airflow как центр современного стека данных хорошо подходит для проверок качества данных.

Проверки качества данных отличаются от тестирования кода тем, что данные не статичны, в отличие от кода DAG. Рекомендуется встраивать проверки качества в DAG и использовать [зависимости в Airflow](https://www.astronomer.io/docs/learn/managing-dependencies) и [ветвление](https://www.astronomer.io/docs/learn/airflow-branch-operator), чтобы задать поведение при проблемах с качеством: от остановки пайплайна до [уведомлений](https://www.astronomer.io/docs/learn/error-notifications-in-airflow) ответственным за качество данных.

Интегрировать проверки качества в DAG можно разными способами:

- [Soda Core](https://www.astronomer.io/docs/learn/soda-data-quality): фреймворк для проверок качества данных; конфигурация в YAML для реляционных БД и Spark DataFrame.
- [Great Expectations](https://www.astronomer.io/docs/learn/airflow-great-expectations): набор проверок качества данных с [провайдером для Airflow](https://registry.astronomer.io/providers/great-expectations); проверки задаются в JSON для реляционных БД, Spark и pandas DataFrame.
- [SQL check operators](https://www.astronomer.io/docs/learn/airflow-sql-data-quality): встроенные в Airflow операторы для настраиваемых проверок качества на разных реляционных БД.

Проверки качества масштабируются лучше, если DAG загружают или обрабатывают данные инкрементально. Подробнее: [DAG Writing Best Practices in Apache Airflow](https://www.astronomer.io/docs/learn/dag-best-practices). Обработка небольших инкрементальных порций в каждом DAG Run ограничивает влияние проблем с качеством данных.

Дополнительно о качестве данных в Airflow:

- [Get Improved Data Quality Checks in Airflow with the Updated Great Expectations Operator](https://www.astronomer.io/blog/improved-data-quality-checks-in-airflow-with-great-expectations-operator/)
- [How to Keep Data Quality in Check with Airflow](https://www.astronomer.io/blog/how-to-keep-data-quality-in-check-with-airflow/)
- [Data quality and Airflow guide](https://www.astronomer.io/docs/learn/data-quality)

---

[← Синхронное выполнение](synchronous-execution.md) | [К содержанию](README.md)
