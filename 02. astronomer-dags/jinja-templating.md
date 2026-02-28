# Шаблонирование в Airflow (Jinja)

> Эта страница ещё не обновлена под Airflow 3. Описанные концепции актуальны, но часть примеров кода может потребовать правок. При запуске примеров обновляйте импорты и учитывайте возможные breaking changes.
>
> Информация

Шаблонирование позволяет передавать в экземпляры задач динамические данные в момент выполнения. Например, следующая команда выводит день недели при каждом запуске задачи:

```python
BashOperator(
    task_id="print_day_of_week",
    bash_command="echo Today is {{ execution_date.format('dddd') }}",
)
```

В этом примере выражение в двойных фигурных скобках `{{ }}` — шаблонный код, который вычисляется в рантайме. При запуске в среду BashOperator выведет `Today is Wednesday`. Шаблоны применяются во многих сценариях: создание каталога по дате выполнения (`/data/path/20210824`), выбор партиции (`/data/path/yyyy=2021/mm=08/dd=24`) для чтения только данных за нужную дату и т.д.

В качестве движка шаблонов Airflow использует [Jinja](https://jinja.palletsprojects.com/) — фреймворк шаблонирования для Python. В этом руководстве:

- как рендерить шаблоны в строки и в нативный Python-код;
- как подключать свои переменные и функции;
- как проверять шаблоны;
- какие поля операторов шаблонируются, а какие нет;
- какие переменные и функции доступны в шаблонах;
- как применять Jinja-шаблоны в коде.

> Astronomer Academy: модуль [Airflow: Templating](https://academy.astronomer.io/astro-runtime-templating).
>
> Дополнительные материалы по теме см. в разделе «Other ways to learn».

## Необходимая база

Полезно понимать:

- Основы Jinja. См. [Jinja basics](https://jinja.palletsprojects.com/en/3.1.x/api/#basics).
- Операторы Airflow. См. [Что такое оператор?](../01.%20astronomer-basic/operators.md).

## Переменные шаблонирования в Airflow

Шаблонирование в Airflow устроено так же, как в Jinja: выражение в двойных фигурных скобках вычисляется в момент выполнения.

Часто используемые переменные Airflow в шаблонах:

- **`{{ data_interval_end }}`** — конец интервала данных.
- **`{{ data_interval_start }}`** — начало интервала данных.
- **`{{ ds_nodash }}`** — логическая дата DAG run в формате `YYYYMMDD`.
- **`{{ ds }}`** — логическая дата DAG run в формате `YYYY-MM-DD`.

> Чтобы использовать Jinja-шаблон внутри Python f-string, добавьте дополнительные фигурные скобки: `name_string = f"my name is {{{{ var.value.get('var_name') }}}}"`
>
> Совет

Полный список переменных: [Templates reference](https://airflow.apache.org/docs/apache-airflow/stable/macros-ref.html#default-variables) в документации Airflow.

В Airflow 2.10+ в шаблонируемые поля можно передавать Python-callable вместо Jinja-шаблона, см. [Использование Python-callable для полей шаблонов](#use-a-python-callable-for-template-fields).

## Шаблонируемые поля и скрипты

Шаблоны применимы не ко всем аргументам оператора. В BaseOperator за это отвечают два атрибута:

- **`template_ext`** — расширения файлов, содержимое которых можно шаблонировать.
- **`template_fields`** — аргументы оператора, в которых допускаются шаблонные значения.

Упрощённый пример BashOperator:

```python
class BashOperator(BaseOperator):
    template_fields = ('bash_command', 'env')
    template_ext = ('.sh', '.bash')

    def __init__(
        self,
        *,
        bash_command,
        env: None,
        output_encoding: 'utf-8',
        **kwargs,
    ):
        super().__init__(**kwargs)
        self.bash_command = bash_command
        self.env = env
        self.output_encoding = output_encoding
```

В `template_fields` перечислены поля, поддерживающие шаблоны. Этот список также отображается в UI Airflow.

В `template_ext` перечислены расширения файлов, которые читаются и шаблонируются в рантайме. Вместо команды в `bash_command` можно указать путь к скрипту `.sh` с шаблонными выражениями:

```python
run_this = BashOperator(
    task_id="run_this",
    bash_command="script.sh",
)
```

BashOperator читает содержимое скрипта, подставляет шаблоны и выполняет его:

```bash
# script.sh
echo "Today is {{ execution_date.format('dddd') }}"
```

Шаблонирование из файлов удобно: IDE может подсвечивать синтаксис языка в скрипте, чего нельзя сделать для большой строки внутри кода DAG.

По умолчанию Airflow ищет скрипты относительно каталога, в котором лежит файл DAG. Если DAG в `/path/to/dag.py`, а скрипт в `/path/to/scripts/script.sh`, в примере выше укажите `scripts/script.sh`.

Базовый путь для шаблонов можно задать на уровне DAG аргументом **`template_searchpath`**. Тогда следующий DAG будет искать `script.sh` в `/tmp/script.sh`:

```python
@dag(..., template_searchpath="/tmp")
def my_dag():
    run_this = BashOperator(task_id="run_this", bash_command="script.sh")
```

```python
with DAG(..., template_searchpath="/tmp") as dag:
    run_this = BashOperator(task_id="run_this", bash_command="script.sh")
```

### Шаблонирование дополнительных полей {#templating-additional-fields}

Если нужно шаблонировать поле, которого нет в `template_fields` оператора, можно изменить атрибут `template_fields` у конкретной задачи или вынести логику в свой оператор. Ниже — оба способа на примере поля `cwd` у BashOperator.

**Изменение атрибута у задачи.** После создания задачи и присвоения её переменной можно изменить её `template_fields`. Так можно включить Jinja для любого поля. Удобно, когда шаблонировать поле нужно только в одном месте.

```python
from airflow.decorators import dag
from airflow.operators.bash import BashOperator
from airflow.utils.dates import days_ago


@dag(schedule=None, start_date=days_ago(1))
def templating_dag():
    bash_task = BashOperator(
        task_id="set_template_field",
        bash_command="script.sh",
        cwd="/usr/local/airflow/{{ ds }}",
    )
    bash_task.template_fields = ("bash_command", "env", "cwd")


templating_dag()
```

**Свой оператор.** Можно наследовать нужный оператор и добавить поле в `template_fields`. В примере ниже TemplatedBashOperator наследует BashOperator и разрешает шаблонирование поля `cwd`. Такой способ удобен, если шаблонировать поле нужно во многих местах.

В существующих проектах можно назвать свой оператор так же, как стандартный — тогда при рефакторинге достаточно поменять импорты.

```python
from airflow.decorators import dag
from airflow.operators.bash import BashOperator
from airflow.utils.dates import days_ago
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from collections.abc import Sequence


class TemplatedBashOperator(BashOperator):
    template_fields: Sequence[str] = ("bash_command", "env", "cwd")


@dag(schedule=None, start_date=days_ago(1))
def templating_dag():
    bash_task = TemplatedBashOperator(
        task_id="custom_operator",
        bash_command="script.sh",
        cwd="/usr/local/airflow/{{ ds }}",
    )


templating_dag()
```

### Отключение шаблонирования

Начиная с Airflow 2.8 можно отключить шаблонирование для значения, передаваемого в шаблонируемое поле, не меняя сам оператор. Это нужно, когда в оператор нужно передать строку с синтаксисом Jinja без её рендеринга (например, шаблон для BashOperator, который не должен подставляться). Для этого строку оборачивают в функцию **`literal`**:

```python
from airflow.utils.template import literal

BashOperator(
    task_id="use_literal_wrapper_to_ignore_jinja_template",
    bash_command=literal("echo {{ params.the_best_number }}"),
)
```

В логах будет выведено `{{ params.the_best_number }}`, а не результат подстановки `params.the_best_number`.

## Использование Python-callable для полей шаблонов {#use-a-python-callable-for-template-fields}

В Airflow 2.10+ в шаблонируемые поля можно передавать Python-callable. Это удобно, когда значение строится сложной логикой, которую трудно или невозможно выразить в Jinja.

В примере ниже у TriggerDagRunOperator параметр `conf` формируется из значения в JSON-файле через callable вместо Jinja-шаблона. У передаваемого callable обязательно должны быть два именованных аргумента: **`context`** и **`jinja_env`**.

```python
def build_conf(context, jinja_env):
    import json

    with open("include/configuration.json", "r") as file:
        data = json.load(file)
    value = data.get("time_value", None)
    return {"sleep_time": value}


tdro = TriggerDagRunOperator(
    task_id="tdro",
    trigger_dag_id="tdro_downstream",
    conf=build_conf,
)
```

Подробнее: [Jinja Templating](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/operators.html#jinja-templating) в документации Airflow.

## Проверка шаблонов

Результат подстановки шаблонов можно посмотреть в UI и в CLI Airflow. В CLI не нужно запускать задачи, чтобы увидеть результат.

Команда CLI **`airflow tasks render`** рендерит все шаблонируемые атрибуты указанной задачи. Для заданных `dag_id`, `task_id` и `execution_date` вывод выглядит примерно так:

```bash
$ airflow tasks render example_dag run_this 2021-01-01

# ----------------------------------------------------------
# property: bash_command
# ----------------------------------------------------------
echo "Today is Friday"

# ----------------------------------------------------------
# property: env
# ----------------------------------------------------------
None
```

Для работы команды нужен доступ к метаданным (БД). Чтобы поднять локальную SQLite:

```bash
cd <your-project-directory>
export AIRFLOW_HOME=$(pwd)
airflow db migrate
# создаёт airflow.db, airflow.cfg, webserver_config.py в каталоге проекта
# в версиях до 2.7 используйте airflow db init вместо airflow db migrate

# airflow tasks render [dag_id] [task_id] [execution_date]
```

При использовании Astro CLI база PostgreSQL настраивается автоматически после `astro dev start`. Затем можно выполнить `astro dev run tasks render ...` для проверки шаблонов.

Для большинства шаблонов этого достаточно. Если шаблон обращается к внешней системе (например, к переменной в продакшен-БД Airflow), нужен доступ к ней.

Чтобы посмотреть результат шаблонов после запуска задачи в UI, откройте задачу и нажмите **Rendered** (вкладка с отрендеренными атрибутами).

## Макросы: свои функции и переменные в шаблонах

В шаблонах доступен набор переменных. Окружение Jinja и среда выполнения Airflow — разные вещи: в Jinja нельзя импортировать модули. Например, такой код в шаблоне не сработает:

```python
from datetime import datetime

BashOperator(
    task_id="print_now",
    # jinja2.exceptions.UndefinedError: 'datetime' is undefined
    bash_command="echo It is currently {{ datetime.now() }}",
)
```

Функции можно внедрить в окружение Jinja. В Airflow по умолчанию подключаются стандартные модули Python под именем **macros**. Тот же пример с `macros.datetime`:

```python
BashOperator(
    task_id="print_now",
    bash_command="echo It is currently {{ macros.datetime.now() }}",
)
```

Список доступных макросов: [документация Airflow](https://airflow.apache.org/docs/apache-airflow/stable/templates-ref.html#macros). Загрузка JSON: `{{ macros.json.loads(...) }}`, YAML: `{{ macros.yaml.safe_load(...) }}`.

Кроме встроенных макросов можно использовать свои переменные и функции. Их передают в окружение Jinja через **`user_defined_macros`** в DAG. Пример: функция, возвращающая число дней с 1 мая 2015 года:

```python
def days_to_now(starting_date):
    return (datetime.now() - starting_date).days
```

Чтобы использовать её в шаблоне, в DAG передают словарь в `user_defined_macros`:

```python
def days_to_now(starting_date):
    return (datetime.now() - starting_date).days


@dag(
    start_date=datetime(2021, 1, 1),
    schedule=None,
    user_defined_macros={
        "starting_date": datetime(2015, 5, 1),
        "days_to_now": days_to_now,
    },
)
def demo_template():
    print_days = BashOperator(
        task_id="print_days",
        bash_command="echo Days since {{ starting_date }} is {{ days_to_now(starting_date) }}",
    )


demo_template()
```

Функции можно подключать и как [фильтры Jinja](https://jinja.palletsprojects.com/en/3.0.x/api/#jinja2.Environment.filters) через **`user_defined_filters`**. Фильтры вызываются через pipe `|`. Тот же результат с фильтром:

```python
@dag(
    start_date=datetime(2021, 1, 1),
    schedule=None,
    user_defined_filters={"days_to_now": days_to_now},
    user_defined_macros={"starting_date": datetime(2015, 5, 1)},
)
def bash_script_template():
    print_days = BashOperator(
        task_id="print_days",
        bash_command="echo Days since {{ starting_date }} is {{ starting_date | days_to_now }}",
    )


bash_script_template()
```

И макросы, и фильтры доступны в шаблонах. Результат одинаковый; Astronomer рекомендует фильтры, когда своих функций несколько — цепочка фильтров читается слева направо, вложенные вызовы функций — сложнее:

```python
"{{ name | striptags | title }}"
"{{ title(striptags(name)) }}"
```

Если нужно вызывать функцию на верхнем уровне DAG (например, в `default_args`), макрос регистрируют как [Airflow plugin](https://www.astronomer.io/docs/learn/using-airflow-plugins). Плюс: выражение вычисляется только при выполнении, а не при каждом парсинге DAG (см. [избегание кода на верхнем уровне](dag-best-practices.md)). Пример — код в файле в каталоге `plugins`:

```python
from airflow.plugins_manager import AirflowPlugin


def get_acl():
    return 'helooo!'


class TestPlugin(AirflowPlugin):
    name = 'test_macro'
    macros = [get_acl]
```

После этого макрос `get_acl` можно использовать в `default_args` через Jinja:

```python
default_args = {
    'owner': 'astro',
    'access_control_list': "{{ macros.test_macro.get_acl() }}",
}
```

## Рендер в нативный Python-код

По умолчанию Jinja всегда рендерит шаблоны в строки. Иногда нужен результат в виде нативного типа Python. Если вызываемый код ожидает не строку, возможны ошибки. Пример:

```python
def sum_numbers(*args):
    total = 0
    for val in args:
        total += val
    return total


sum_numbers(1, 2, 3)
# 6
sum_numbers("1", "2", "3")
# TypeError: unsupported operand type(s) for +=: 'int' and 'str'
```

Сценарий: в DAG в `op_args` передаётся список из конфига DAG run (триггер с JSON `{"numbers": [1,2,3]}`):

```python
@dag(
    start_date=datetime.datetime(2021, 1, 1),
    schedule=None,
    catchup=False
)
def failing_template():
    PythonOperator(
        task_id="sumnumbers",
        python_callable=sum_numbers,
        op_args="{{ dag_run.conf['numbers'] }}",
    )


failing_template()
```

Отрендеренное значение — строка. Функция `sum_numbers` распаковывает эту строку и по сути складывает символы:

```python
('[', '1', ',', ' ', '2', ',', ' ', '3', ']')
```

Чтобы получить нативный список, нужно настроить Jinja на рендер в нативные типы. [Стандартное окружение Jinja](https://jinja.palletsprojects.com/en/3.0.x/api/#jinja2.Environment) выдаёт строки; [NativeEnvironment](https://jinja.palletsprojects.com/en/3.0.x/nativetypes/#jinja2.nativetypes.NativeEnvironment) — нативный Python-код. В DAG это включается аргументом **`render_template_as_native_obj=True`**:

```python
def sum_numbers(*args):
    total = 0
    for val in args:
        total += val
    return total


@dag(
    dag_id="native_templating",
    start_date=datetime.datetime(2021, 1, 1),
    schedule=None,
    render_template_as_native_obj=True,
)
def native_templating():
    sumnumbers = PythonOperator(
        task_id="sumnumbers",
        python_callable=sum_numbers,
        op_args="{{ dag_run.conf['numbers'] }}",
    )


native_templating()
```

При той же конфигурации `{"numbers": [1,2,3]}` в шаблон подставится список целых чисел, и `sum_numbers` вернёт 6.

Окружение Jinja задаётся на уровне DAG: все задачи DAG рендерят шаблоны либо обычным окружением (строки), либо NativeEnvironment (нативные объекты).

---

[← Контекст](airflow-context.md) | [К содержанию](README.md) | [Передача данных →](passing-data-between-tasks.md)
