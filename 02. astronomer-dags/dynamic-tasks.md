# Динамические задачи (Dynamic task mapping)

При **dynamic task mapping** (динамическом маппинге задач) можно писать DAG, которые в рантайме создают параллельные задачи. Это меняет подход к проектированию DAG: задачи создаются по текущему окружению без правок кода DAG.

В этом руководстве — что такое динамический маппинг и полный пример для типичного сценария.

## Необходимая база

Полезно понимать:

- XCom в Airflow. См. [Передача данных между задачами](passing-data-between-tasks.md).
- Декораторы для определения задач. См. [Декораторы Airflow](airflow-decorators.md).
- Операторы Airflow. См. [Что такое оператор?](../01.%20astronomer-basic/operators.md).

## Концепции динамического маппинга

Динамический маппинг в Airflow основан на модели [MapReduce](https://en.wikipedia.org/wiki/MapReduce). Для каждого входного элемента создаётся одна задача (map). Опциональная фаза reduce — одна задача обрабатывает собранный вывод маппленных задач. То есть DAG может создать произвольное число параллельных задач в рантайме по входному параметру (map) и при необходимости одну задачу ниже, зависящую от их вывода (reduce).

У задачи Airflow есть две функции для фазы map:

- **`partial()`**: передаёт параметры, одинаковые для всех маппленных задач, создаваемых `expand()`.
- **`expand()`**: передаёт параметры, по которым идёт маппинг; для каждого элемента создаётся отдельная параллельная задача. В части случаев при [маппинге по нескольким параметрам](#mapping-over-multiple-parameters) вместо этого используется **`.expand_kwargs()`**.

В примере ниже задача использует и `.partial()`, и `.expand()` и создаёт три запуска задачи:

```python
from airflow.sdk import task


@task
def add(x: int, y: int):
    return x + y


added_values = add.partial(y=10).expand(x=[1, 2, 3])
```

```python
from airflow.providers.standard.operators.python import PythonOperator


def add_function(x: int, y: int):
    return x + y


added_values = PythonOperator.partial(
    task_id="add",
    python_callable=add_function,
    op_kwargs={"y": 10},
).expand(op_args=[[1], [2], [3]])
```

Функция `expand` создаёт три маппленные задачи `add` — по одной на каждый элемент списка `x`. Функция `partial` задаёт значение `y`, общее для всех задач.

При работе с маппленными задачами учитывайте:

- XCom маппленных экземпляров хранятся в виде списка; доступ по map index. Например, XCom третьего экземпляра (map index 2) задачи `my_mapped_task`: `ti.xcom_pull(task_ids=['my_mapped_task'])[2]` или `ti.xcom_pull(task_ids=['my_mapped_task'], map_indexes=[2])`.
- Ограничить число параллельно выполняемых маппленных экземпляров можно параметрами задачи: **max_active_tis_per_dag** (по всем DAG run) и **max_active_tis_per_dagrun** (в рамках одного DAG run).
- Максимальное число маппленных экземпляров задаётся конфигом [max_map_length](https://airflow.apache.org/docs/apache-airflow/stable/configurations-ref.html#max-map-length) (по умолчанию 1024).
- **expand()** принимает только keyword arguments.
- Некоторые параметры маппить нельзя (например, `task_id`, `pool` и многие аргументы BaseOperator).
- Маппленная задача может не создать ни одного экземпляра (например, если вышестоящая задача вернула пустой список). Тогда маппленная задача помечается skipped; нижестоящие выполняются согласно trigger rules (по умолчанию тоже skipped).
- Результат маппленной задачи можно передать в следующую маппленную задачу.
- Можно маппить по нескольким параметрам.
- Результат вышестоящей задачи можно использовать как вход маппленной задачи. Вышестоящая задача должна вернуть значение в виде `dict` или `list`. При традиционных операторах (не [декорированных задачах](airflow-decorators.md)) значения для маппинга должны быть в XCom.

Дополнительные примеры: [Dynamic Task Mapping](https://airflow.apache.org/docs/apache-airflow/stable/concepts/dynamic-task-mapping.html) в документации Airflow.

В UI маппленные задачи отображаются в Grid View: в графе DAG run у задачи в скобках `[ ]` указано число созданных экземпляров. Логи и XCom каждого экземпляра открываются по клику на соответствующий квадрат в сетке.

## Маппинг по результату другого оператора

Входные данные для маппленной нижестоящей задачи можно брать из вывода вышестоящего оператора.

Ниже — как передавать данные маппинга в каждом из вариантов:

- **Традиционный → традиционный:** обе задачи — традиционные операторы.
- **Традиционный → TaskFlow:** вышестоящая — TaskFlow, нижестоящая — традиционный оператор.
- **TaskFlow → традиционный:** вышестоящая — традиционный оператор, нижестоящая — TaskFlow.
- **TaskFlow → TaskFlow:** обе задачи — TaskFlow API.

**TaskFlow → TaskFlow:** вызов вышестоящей задачи передаётся аргументом в `expand()`:

```python
from airflow.sdk import task


@task
def one_two_three_TF():
    return [1, 2, 3]


@task
def plus_10_TF(x):
    return x + 10


plus_10_TF.partial().expand(x=one_two_three_TF())
```

**Традиционный оператор:** в `expand()` передаётся атрибут **`.output`** объекта задачи:

```python
from airflow.sdk import task
from airflow.providers.standard.operators.python import PythonOperator


def one_two_three_traditional():
    return [1, 2, 3]


@task
def plus_10_TF(x):
    return x + 10


one_two_three_task = PythonOperator(
    task_id="one_two_three_task", python_callable=one_two_three_traditional
)

plus_10_TF.partial().expand(x=one_two_three_task.output)
```

**TaskFlow → традиционный PythonOperator:** вывод нужно привести к формату, который принимает **op_args** у PythonOperator:

```python
from airflow.sdk import task
from airflow.providers.standard.operators.python import PythonOperator


@task
def one_two_three_TF():
    # op_args ожидает каждый аргумент в виде списка
    return [[1], [2], [3]]


def plus_10_traditional(x):
    return x + 10


plus_10_task = PythonOperator.partial(
    task_id="plus_10_task", python_callable=plus_10_traditional
).expand(op_args=one_two_three_TF())
```

**Традиционный → традиционный:** используйте `.output` вышестоящей задачи; формат возвращаемого значения должен соответствовать **op_args**:

```python
from airflow.providers.standard.operators.python import PythonOperator


def one_two_three_traditional():
    return [[1], [2], [3]]


def plus_10_traditional(x):
    return x + 10


one_two_three_task = PythonOperator(
    task_id="one_two_three_task", python_callable=one_two_three_traditional
)

plus_10_task = PythonOperator.partial(
    task_id="plus_10_task", python_callable=plus_10_traditional
).expand(op_args=one_two_three_task.output)

one_two_three_task >> plus_10_task
```

### Маппинг по объединённым результатам вышестоящих задач

Списки результатов вышестоящих задач можно объединить методом **`.concat()`**. Раньше для этого нужна была промежуточная задача.

В примере ниже для задачи `map_me` создаётся 7 маппленных экземпляров:

```python
from airflow.sdk import task
import time


@task
def t1():
    return [1, 2, 3]


t1_obj = t1()


@task
def t2():
    return [4, 5, 6, 7]


t2_obj = t2()


@task
def map_me(input):
    print(f"Sleeping for {input} seconds!")
    time.sleep(input)
    print("Waking up!")


map_me.expand(input=t1_obj.concat(t2_obj))
```

```python
from airflow.providers.standard.operators.python import PythonOperator
import time


def t1_func():
    return [[1], [2], [3]]


def t2_func():
    return [[4], [5], [6], [7]]


def map_me_func(input):
    print(f"Sleeping for {input} seconds!")
    time.sleep(input)
    print("Waking up!")


t1 = PythonOperator(task_id="t1", python_callable=t1_func)
t2 = PythonOperator(task_id="t2", python_callable=t2_func)

map_me = PythonOperator.partial(
    task_id="map_me", python_callable=map_me_func
).expand(op_args=t1.output.concat(t2.output))
```

## Маппинг по нескольким параметрам {#mapping-over-multiple-parameters} {#mapping-over-multiple-parameters}

Доступны три способа маппить по нескольким параметрам:

- **Zip:** маппинг по наборам позиционных аргументов (Python `zip()` или `.zip()` у XComArg) — один маппленный экземпляр на каждый набор. Используется **expand()**.
- **Наборы keyword arguments:** маппинг по двум и более наборам kwargs — один экземпляр на каждый набор, а не на каждую комбинацию. Используется **expand_kwargs()**.
- **Декартово произведение (cross-product):** маппинг по двум и более keyword arguments — один экземпляр на каждую комбинацию входов. Используется **expand()**.

### Декартово произведение (cross-product)

По умолчанию `expand()` создаёт маппленный экземпляр для **каждой комбинации** всех переданных входов. Например, при двух значениях первого аргумента, четырёх второго и пяти третьего получится 2×4×5=40 экземпляров. Типичный сценарий — перебор гиперпараметров модели.

В примере ниже маппинг по трём вариантам `bash_command` и трём вариантам `env` даёт 3×3=9 экземпляров. Каждая команда выполняется с каждым значением переменной окружения `WORD`:

```python
from airflow.providers.standard.operators.bash import BashOperator

cross_product_example = BashOperator.partial(
    task_id="cross_product_example"
).expand(
    bash_command=[
        "echo $WORD",
        "echo `expr length $WORD`",
        "echo \\${WORD//e/X}"
    ],
    env=[
        {"WORD": "hello"},
        {"WORD": "tea"},
        {"WORD": "goodbye"}
    ]
)
```

Девять экземпляров задачи `cross_product_example` дают все комбинации команды и переменной: map index 0 — `hello`, 1 — `tea`, 2 — `goodbye`, 3 — `5`, 4 — `3`, 5 — `7`, 6 — `hXllo`, 7 — `tXa`, 8 — `goodbyX`.

### Наборы keyword arguments

Чтобы маппить по наборам значений для двух и более keyword arguments, используется **expand_kwargs()**. Наборы задаются списком словарей или XComArg. В примере ниже оператор получает 3 набора параметров — 3 маппленных экземпляра:

```python
from airflow.providers.standard.operators.bash import BashOperator

t1 = BashOperator.partial(task_id="t1").expand_kwargs(
    [
        {"bash_command": "echo $WORD", "env": {"WORD": "hello"}},
        {"bash_command": "echo `expr length $WORD`", "env": {"WORD": "tea"}},
        {"bash_command": "echo \\${WORD//e/X}", "env": {"WORD": "goodbye"}}
    ]
)
```

У задачи `t1` три маппленных экземпляра с выводами: map index 0 — `hello`, 1 — `3`, 2 — `goodbyX`.

### Zip

В маппинге можно передавать наборы позиционных аргументов в один и тот же keyword argument (например, `op_args` у PythonOperator). Если данные — итерируемые (кортежи, словари, списки), можно использовать встроенную функцию Python [`zip()`](https://docs.python.org/3/library/functions.html#zip). Если данные из XCom, используется метод **`.zip()`** у XComArg.

#### Встроенный Python zip()

`zip()` принимает произвольное число итерируемых и строит кортежи из их элементов; число кортежей равно длине кратчайшей итерируемой. Примеры:

- `zip(["a", "b"], ["hi", "bye"], (19, 23))` → `('a', 'hi', 19), ('b', 'bye', 23)`.
- `zip(["a", "b"], [1], ["hi", "bye"], [19, 23], ["x", "y", "z"])` → один кортеж `("a", 1, "hi", 19, "x")` (кратчайший список — один элемент).
- `zip(["a", "b", "c"], [1, 2, 3], ["hi", "bye", "tea"])` → `("a", 1, "hi"), ("b", 2, "bye"), ("c", 3, "tea")`.

Пример: список из zip передаётся в `expand()` для маппинга по наборам позиционных аргументов. В TaskFlow каждый набор передаётся в аргумент `zipped_x_y_z`; в PythonOperator каждый набор распаковывается в `op_args` и попадает в `x`, `y`, `z`:

```python
from airflow.sdk import task

zipped_arguments = list(zip([1, 2, 3], [10, 20, 30], [100, 200, 300]))
# zipped_arguments: [(1,10,100), (2,20,200), (3,30,300)]


@task
def add_numbers(zipped_x_y_z):
    return zipped_x_y_z[0] + zipped_x_y_z[1] + zipped_x_y_z[2]


add_numbers.expand(zipped_x_y_z=zipped_arguments)
```

```python
from airflow.providers.standard.operators.python import PythonOperator

zipped_arguments = list(zip([1, 2, 3], [10, 20, 30], [100, 200, 300]))


def add_numbers_function(x, y, z):
    return x + y + z


add_numbers = PythonOperator.partial(
    task_id="add_numbers",
    python_callable=add_numbers_function,
).expand(op_args=zipped_arguments)
```

Три экземпляра задачи `add_numbers`: map index 0 — `111`, 1 — `222`, 2 — `333`.

#### XComArg.zip()

Можно объединять через `.zip()` и XComArg. Для TaskFlow передаётся вызов задачи, для традиционного оператора — `task_object.output` или `XcomArg(task_object)`. Опциональный аргумент **fillvalue** в `.zip()` повторяет поведение [`zip_longest()`](https://docs.python.org/3/library/itertools.html#itertools.zip_longest): число кортежей равно длине самой длинной итерируемой, недостающие элементы заполняются значением по умолчанию. Без `fillvalue` в примере ниже был бы один кортеж `[(1, 10, 100)]`, так как кратчайший список — один элемент.

```python
from airflow.sdk import task


@task
def one_two_three():
    return [1, 2]


@task
def ten_twenty_thirty():
    return [10]


@task
def one_two_three_hundred():
    return [100, 200, 300]


zipped_arguments = one_two_three().zip(
    ten_twenty_thirty(), one_two_three_hundred(), fillvalue=1000
)
# zipped_arguments: [(1, 10, 100), (2, 1000, 200), (1000, 1000, 300)]


@task
def add_nums(zipped_x_y_z):
    return zipped_x_y_z[0] + zipped_x_y_z[1] + zipped_x_y_z[2]


add_nums.expand(zipped_x_y_z=zipped_arguments)
```

```python
from airflow.providers.standard.operators.python import PythonOperator


def one_two_three_function():
    return [1, 2]


def ten_twenty_thirty_function():
    return [10]


def one_two_three_hundred_function():
    return [100, 200, 300]


one_two_three = PythonOperator(
    task_id="one_two_three", python_callable=one_two_three_function
)
ten_twenty_thirty = PythonOperator(
    task_id="ten_twenty_thirty", python_callable=ten_twenty_thirty_function
)
one_two_three_hundred = PythonOperator(
    task_id="one_two_three_hundred", python_callable=one_two_three_hundred_function
)

zipped_arguments = one_two_three.output.zip(
    ten_twenty_thirty.output, one_two_three_hundred.output, fillvalue=1000
)


def add_nums_function(x, y, z):
    return x + y + z


add_nums = PythonOperator.partial(
    task_id="add_nums", python_callable=add_nums_function
).expand(op_args=zipped_arguments)
```

Три экземпляра `add_nums`: map index 0 — `111`, 1 — `1202`, 2 — `2300`.

## Цепочка маппингов (repeated mapping)

Задачу можно маппить по выводу другой маппленной задачи. Тогда создаётся по одному маппленному экземпляру на каждый экземпляр вышестоящей задачи.

В примере ниже три маппленные задачи в цепочке:

```python
from airflow.sdk import task


@task
def multiply_by_2(num):
    return num * 2


@task
def add_10(num):
    return num + 10


@task
def multiply_by_100(num):
    return num * 100


multiplied_value_1 = multiply_by_2.expand(num=[1, 2, 3])
summed_value = add_10.expand(num=multiplied_value_1)
multiply_by_100.expand(num=summed_value)
```

```python
from airflow.providers.standard.operators.python import PythonOperator


def multiply_by_2_func(num):
    return [num * 2]


def add_10_func(num):
    return [num + 10]


def multiply_by_100_func(num):
    return num * 100


multiply_by_2 = PythonOperator.partial(
    task_id="multiply_by_2",
    python_callable=multiply_by_2_func
).expand(op_args=[[1], [2], [3]])

add_10 = PythonOperator.partial(
    task_id="add_10",
    python_callable=add_10_func
).expand(op_args=multiply_by_2.output)

multiply_by_100 = PythonOperator.partial(
    task_id="multiply_by_100",
    python_callable=multiply_by_100_func
).expand(op_args=add_10.output)

multiply_by_2 >> add_10 >> multiply_by_100
```

У `multiply_by_2` три экземпляра: 2, 4, 6. У `add_10` три экземпляра: 12, 14, 16. У `multiply_by_100` три экземпляра: 1200, 1400, 1600. Так можно выстраивать цепочку из любого числа маппленных задач. Экспоненциально увеличивать число экземпляров при этом нельзя.

## Маппинг по task groups

[Task groups](task-groups.md), определённые декоратором `@task_group`, тоже можно маппить. Синтаксис такой же, как для одной задачи:

```python
from airflow.sdk import task, task_group


@task_group(group_id="group1")
def tg1(my_num):
    @task
    def print_num(num):
        return num

    @task
    def add_42(num):
        return num + 42

    print_num(my_num) >> add_42(my_num)


tg1_object = tg1.expand(my_num=[19, 23, 42, 8, 7, 108])
```

Получается 6 маппленных экземпляров группы `group1`. По нескольким параметрам группы можно маппить так же, как для обычных задач (cross-product, zip, expand_kwargs). См. [Маппинг по нескольким параметрам](#mapping-over-multiple-parameters).

## Преобразование вывода с помощью .map() {#transform-outputs-with-map}

Иногда нужно преобразовать вывод вышестоящей задачи до того, как по нему маппится следующая. Например, если традиционный оператор возвращает данные в фиксированном формате или нужно пропустить часть экземпляров по условию.

Метод **`.map()`** принимает Python-функцию и применяет её к итерируемому входу перед маппингом. Вызов: на задаче TaskFlow — `my_upstream_task().map(mapping_function)`, на традиционном операторе — `my_upstream_operator.output.map(mapping_function)`. Нижестоящая задача маппится по результату `.map()` через `.expand()` (один keyword argument) или `.expand_kwargs()` (список словарей с наборами kwargs).

В примере ниже `.map()` используется, чтобы пропускать отдельные элементы по условию (через `AirflowSkipException`):

```python
from airflow.sdk import task
from airflow.exceptions import AirflowSkipException


@task
def list_strings():
    return ["skip_hello", "hi", "skip_hallo", "hola", "hey"]


def skip_strings_starting_with_skip(string):
    if len(string) < 4:
        return string + "!"
    elif string[:4] == "skip":
        raise AirflowSkipException(f"Skipping {string}; as I was told!")
    else:
        return string + "!"


transformed_list = list_strings().map(skip_strings_starting_with_skip)


@task
def mapped_printing_task(string):
    return "Say " + string


mapped_printing_task.partial().expand(string=transformed_list)
```

```python
from airflow.providers.standard.operators.python import PythonOperator
from airflow.exceptions import AirflowSkipException


def list_strings():
    return ["skip_hello", "hi", "skip_hallo", "hola", "hey"]


listed_strings = PythonOperator(
    task_id="list_strings",
    python_callable=list_strings,
)


def skip_strings_starting_with_skip(string):
    if len(string) < 4:
        return [string + "!"]
    elif string[:4] == "skip":
        raise AirflowSkipException(f"Skipping {string}; as I was told!")
    else:
        return [string + "!"]


transformed_list = listed_strings.output.map(skip_strings_starting_with_skip)


def mapped_printing_function(string):
    return "Say " + string


mapped_printing = PythonOperator.partial(
    task_id="mapped_printing",
    python_callable=mapped_printing_function,
).expand(op_args=transformed_list)
```

Функция преобразования не появляется как отдельная задача Airflow. В UI во вкладке Mapped Tasks видно, что экземпляры 0 и 2 помечены как skipped.

## Пример: обработка файлов в S3

Типичный сценарий — обработка файлов в Amazon S3: ELT — извлечение из S3, загрузка в Snowflake, трансформация в Snowflake. Число файлов каждый день неизвестно; динамический маппинг создаёт по задаче на каждый файл в рантайме (атомарность, удобная наблюдаемость и восстановление после сбоев).

Полный код примера: [dynamic-task-mapping-tutorial](https://github.com/astronomer/dynamic-task-mapping-tutorial).

Шаги DAG:

1. Задекорированная задача получает список файлов из S3 (префикс с `ds_nodash` — только за дату DAG run, например `20250412/`).
2. По результату маппится `CopyFromExternalStageToSnowflakeOperator` для каждого файла.
3. Одновременно: копирование обработанных файлов в `processed/`, удаление исходной папки, выполнение SQL-трансформации в Snowflake.

**TaskFlow:**

```python
from airflow.decorators import dag, task
from airflow.providers.snowflake.transfers.copy_into_snowflake import (
    CopyFromExternalStageToSnowflakeOperator,
)
from airflow.providers.snowflake.operators.snowflake import SnowflakeOperator
from airflow.providers.amazon.aws.hooks.s3 import S3Hook
from airflow.providers.amazon.aws.operators.s3 import S3CopyObjectOperator
from airflow.providers.amazon.aws.operators.s3 import S3DeleteObjectsOperator
from pendulum import datetime


@dag(
    start_date=datetime(2024, 4, 2),
    catchup=False,
    template_searchpath="/usr/local/airflow/include",
    schedule="@daily",
)
def mapping_elt():
    @task
    def get_s3_files(current_prefix):
        s3_hook = S3Hook(aws_conn_id="s3")
        current_files = s3_hook.list_keys(
            bucket_name="my-bucket",
            prefix=current_prefix + "/",
            start_after_key=current_prefix + "/",
        )
        return [[file] for file in current_files]

    copy_to_snowflake = CopyFromExternalStageToSnowflakeOperator.partial(
        task_id="load_files_to_snowflake",
        stage="MY_STAGE",
        table="COMBINED_HOMES",
        schema="MYSCHEMA",
        file_format="(type = 'CSV',field_delimiter = ',', skip_header=1)",
        snowflake_conn_id="snowflake",
    ).expand(files=get_s3_files(current_prefix="{{ ds_nodash }}"))

    move_s3 = S3CopyObjectOperator(
        task_id="move_files_to_processed",
        aws_conn_id="s3",
        source_bucket_name="my-bucket",
        source_bucket_key="{{ ds_nodash }}" + "/",
        dest_bucket_name="my-bucket",
        dest_bucket_key="processed/" + "{{ ds_nodash }}" + "/",
    )

    delete_landing_files = S3DeleteObjectsOperator(
        task_id="delete_landing_files",
        aws_conn_id="s3",
        bucket="my-bucket",
        prefix="{{ ds_nodash }}" + "/",
    )

    transform_in_snowflake = SnowflakeOperator(
        task_id="run_transformation_query",
        sql="/transformation_query.sql",
        snowflake_conn_id="snowflake",
    )

    copy_to_snowflake >> [move_s3, transform_in_snowflake]
    move_s3 >> delete_landing_files


mapping_elt()
```

**Традиционный вариант:**

```python
from airflow import DAG
from airflow.providers.snowflake.transfers.copy_into_snowflake import (
    CopyFromExternalStageToSnowflakeOperator,
)
from airflow.operators.python import PythonOperator
from airflow.providers.snowflake.operators.snowflake import SnowflakeOperator
from airflow.providers.amazon.aws.hooks.s3 import S3Hook
from airflow.providers.amazon.aws.operators.s3 import S3CopyObjectOperator
from airflow.providers.amazon.aws.operators.s3 import S3DeleteObjectsOperator
from pendulum import datetime


def get_s3_files(current_prefix):
    s3_hook = S3Hook(aws_conn_id="s3")
    current_files = s3_hook.list_keys(
        bucket_name="my-bucket",
        prefix=current_prefix + "/",
        start_after_key=current_prefix + "/",
    )
    return [[file] for file in current_files]


with DAG(
    "mapping_elt_traditional",
    start_date=datetime(2024, 4, 2),
    catchup=False,
    template_searchpath="/usr/local/airflow/include",
    schedule="@daily",
):
    get_s3_files_task = PythonOperator(
        task_id="get_s3_files",
        python_callable=get_s3_files,
        op_kwargs={"current_prefix": "{{ ds_nodash }}"},
    )

    copy_to_snowflake = CopyFromExternalStageToSnowflakeOperator.partial(
        task_id="load_files_to_snowflake",
        stage="MY_STAGE",
        table="COMBINED_HOMES",
        schema="MYSCHEMA",
        file_format="(type = 'CSV',field_delimiter = ',', skip_header=1)",
        snowflake_conn_id="snowflake",
    ).expand(files=get_s3_files_task.output)

    move_s3 = S3CopyObjectOperator(
        task_id="move_files_to_processed",
        aws_conn_id="s3",
        source_bucket_name="my-bucket",
        source_bucket_key="{{ ds_nodash }}" + "/",
        dest_bucket_name="my-bucket",
        dest_bucket_key="processed/" + "{{ ds_nodash }}" + "/",
    )

    delete_landing_files = S3DeleteObjectsOperator(
        task_id="delete_landing_files",
        aws_conn_id="s3",
        bucket="my-bucket",
        prefix="{{ ds_nodash }}" + "/",
    )

    transform_in_snowflake = SnowflakeOperator(
        task_id="run_transformation_query",
        sql="/transformation_query.sql",
        snowflake_conn_id="snowflake",
    )

    copy_to_snowflake >> [move_s3, transform_in_snowflake]
    move_s3 >> delete_landing_files
```

При маппинге важно учитывать формат параметра: в примере выше своя функция формирует список списков из ключей S3, потому что оператор загрузки в Snowflake ожидает каждый ключ в виде списка, а `list_keys` возвращает один плоский список. Такая функция превращает результат хука в список списков для нижестоящего оператора.

---

[← Отладка](debugging-dags.md) | [К содержанию](README.md) | [Task groups →](task-groups.md) | [Передача данных →](passing-data-between-tasks.md)
