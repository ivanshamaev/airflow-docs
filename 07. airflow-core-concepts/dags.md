# DAG

**DAG** — это модель, которая описывает всё необходимое для выполнения рабочего процесса (workflow). Среди атрибутов DAG:

- **Schedule (расписание)**: когда должен запускаться workflow.
- **Tasks (задачи)**: [задачи](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/tasks.html) — отдельные единицы работы, выполняемые на воркерах.
- **Task Dependencies (зависимости задач)**: порядок и условия выполнения [задач](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/tasks.html).
- **Callbacks**: действия при завершении всего workflow.
- **Дополнительные параметры**: и другие эксплуатационные детали.

Простой пример DAG:

![Простой пример DAG с четырьмя задачами A, B, C, D.](images/dags.png)

В нём четыре задачи — A, B, C и D — задан порядок выполнения и зависимости между ними. Указано и то, как часто запускать DAG: например, «каждые 5 минут начиная с завтра» или «каждый день с 1 января 2020».

Сам DAG не зависит от того, что происходит внутри задач; он задаёт только способ их выполнения: порядок, число повторов, таймауты и т.п.

> **Примечание.** Термин «DAG» восходит к математическому понятию «ориентированный ациклический граф», но в Airflow его смысл давно шире этой структуры. Поэтому в Airflow принято написание **Dag**.

## Объявление DAG

Объявить DAG можно тремя способами.

**1. Контекстный менеджер `with`** — всё, что внутри блока, неявно попадает в DAG:

```python
import datetime

from airflow.sdk import DAG
from airflow.providers.standard.operators.empty import EmptyOperator

with DAG(
    dag_id="my_dag_name",
    start_date=datetime.datetime(2021, 1, 1),
    schedule="@daily",
):
    EmptyOperator(task_id="task")
```

**2. Обычный конструктор** — DAG передаётся в каждый оператор явно:

```python
import datetime

from airflow.sdk import DAG
from airflow.providers.standard.operators.empty import EmptyOperator

my_dag = DAG(
    dag_id="my_dag_name",
    start_date=datetime.datetime(2021, 1, 1),
    schedule="@daily",
)
EmptyOperator(task_id="task", dag=my_dag)
```

**3. Декоратор `@dag`** — функция превращается в фабрику DAG:

```python
import datetime

from airflow.sdk import dag
from airflow.providers.standard.operators.empty import EmptyOperator


@dag(start_date=datetime.datetime(2021, 1, 1), schedule="@daily")
def generate_dag():
    EmptyOperator(task_id="task")


generate_dag()
```

DAG без [задач](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/tasks.html) бесполезен; задачи обычно задаются [операторами](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/operators.html), [сенсорами](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/sensors.html) или [TaskFlow](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/taskflow.html).

### Зависимости между задачами

У задачи обычно есть зависимости от других задач (выше по потоку — upstream) и задачи, зависящие от неё (ниже по потоку — downstream). Описание этих связей и есть структура DAG.

Зависимости между двумя задачами задают двумя основными способами. Рекомендуемый — операторы `>>` и `<<`:

```python
first_task >> [second_task, third_task]
third_task << fourth_task
```

Либо явными методами `set_upstream` и `set_downstream`:

```python
first_task.set_downstream([second_task, third_task])
third_task.set_upstream(fourth_task)
```

Для более сложных зависимостей есть сокращения. Чтобы сделать список задач зависимым от другого списка, одних `>>`/`<<` недостаточно — используется `cross_downstream`:

```python
from airflow.sdk import cross_downstream

# Заменяет
# [op1, op2] >> op3
# [op1, op2] >> op4
cross_downstream([op1, op2], [op3, op4])
```

Цепочку зависимостей можно задать через `chain`:

```python
from airflow.sdk import chain

# Заменяет op1 >> op2 >> op3 >> op4
chain(op1, op2, op3, op4)

# Можно задать и динамически
chain(*[EmptyOperator(task_id=f"op{i}") for i in range(1, 6)])
```

`chain` может задавать попарные зависимости для списков одинаковой длины (это не то же самое, что кросс-зависимости от `cross_downstream`):

```python
from airflow.sdk import chain

# Заменяет
# op1 >> op2 >> op4 >> op6
# op1 >> op3 >> op5 >> op6
chain(op1, [op2, op3], [op4, op5], op6)
```

## Загрузка DAG

Airflow загружает DAG из Python-файлов в Dag bundle: каждый файл выполняется, после чего из него подхватываются все объекты DAG.

В одном файле можно определить несколько DAG или разнести один сложный DAG по нескольким файлам через импорты.

При загрузке DAG Airflow подхватывает только объекты **верхнего уровня**, являющиеся экземплярами DAG. Пример:

```python
dag_1 = DAG('this_dag_will_be_discovered')

def my_function():
    dag_2 = DAG('but_this_dag_will_not')

my_function()
```

Оба конструктора DAG вызываются при выполнении файла, но в `globals()` верхнего уровня оказывается только `dag_1`, поэтому в Airflow попадёт только он. `dag_2` загружен не будет.

> **Примечание.** При поиске DAG в Dag bundle Airflow по умолчанию обрабатывает только те Python-файлы, в которых (без учёта регистра) встречаются подстроки `airflow` и `dag` — это оптимизация.
>
> Чтобы учитывать все Python-файлы, отключите конфигурационный флаг `DAG_DISCOVERY_SAFE_MODE`.

В Dag bundle (или в любом его подкаталоге) можно положить файл `.airflowignore` с шаблонами файлов, которые загрузчик должен игнорировать. Он действует на свой каталог и все вложенные. Подробнее о синтаксисе — в разделе «.airflowignore» ниже.

Если `.airflowignore` недостаточно и нужна своя логика «содержит ли файл DAG», можно задать callable в конфиге в параметре `might_contain_dag_callable`. Этот callable **заменяет** стандартную эвристику Airflow (проверку на наличие подстрок `airflow` и `dag`):

```python
def might_contain_dag(file_path: str, zip_file: zipfile.ZipFile | None = None) -> bool:
    # Ваша логика: есть ли в file_path определения DAG
    # True — файл нужно разбирать, иначе False
```

## Запуск DAG

DAG запускается одним из двух способов:

1. Вручную или через API.
2. По расписанию, заданному в DAG.

Расписание не обязательно, но задаётся часто — через аргумент `schedule`:

```python
with DAG("my_daily_dag", schedule="@daily"):
    ...
```

Допустимые значения `schedule`:

```python
with DAG("my_daily_dag", schedule="0 0 * * *"):
    ...

with DAG("my_one_time_dag", schedule="@once"):
    ...

with DAG("my_continuous_dag", schedule="@continuous"):
    ...
```

> **Совет.** Подробнее о типах расписаний: [Authoring and Scheduling](https://airflow.apache.org/docs/apache-airflow/stable/authoring-and-scheduling/index.html).

Каждый запуск DAG создаёт новый экземпляр — [Dag Run](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/dag-run.html). У одного DAG может быть несколько параллельных Dag Run; у каждого есть свой **data interval** — период данных, с которыми работают задачи.

Пример: DAG обрабатывает дневной набор экспериментальных данных. Код обновили, нужно прогнать его за последние 3 месяца — можно сделать backfill: Airflow запустит копии DAG за каждый день этого периода.

Все эти Dag Run будут созданы в один и тот же день, но у каждого свой data interval (один день из трёх месяцев), и именно его видят задачи при выполнении.

Аналогично тому, как при каждом запуске DAG создаётся Dag Run, задачи, описанные в DAG, при этом превращаются в [Task Instances](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/tasks.html#concepts-task-instances).

У Dag Run есть дата начала (start date) и дата окончания (end date) — период, когда DAG реально выполнялся. Кроме них есть **logical date** (ранее execution date) — момент, на который запланирован или запущен Dag Run. Название «логическая» связано с тем, что в разных контекстах эта дата трактуется по-разному.

Например, при ручном запуске logical date совпадает с датой и временем запуска и с start date Dag Run. При запуске по расписанию logical date — начало data interval, а start date Dag Run = logical date + интервал расписания.

> **Совет.** Подробнее о logical date: [Data Interval](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/dag-run.html#data-interval) и [What does execution_date mean?](https://airflow.apache.org/docs/apache-airflow/stable/faq.html#faq-what-does-execution-date-mean).

## Назначение DAG задаче

Каждая задача/оператор должна быть привязана к DAG. Airflow может определить DAG без явной передачи в нескольких случаях:

- Оператор объявлен внутри блока `with DAG`.
- Оператор объявлен внутри функции, задекорированной `@dag`.
- Оператор указан как upstream или downstream по отношению к задаче, у которой уже есть DAG.

Во всех остальных случаях DAG нужно передавать в каждый оператор аргументом `dag=`.

## Аргументы по умолчанию (default_args)

Часто многим операторам в DAG нужны одни и те же аргументы по умолчанию (например, `retries`). Вместо того чтобы задавать их у каждого оператора, можно передать `default_args` в DAG при создании — они подставятся во все связанные с ним операторы:

```python
import pendulum

with DAG(
    dag_id="my_dag",
    start_date=pendulum.datetime(2016, 1, 1),
    schedule="@daily",
    default_args={"retries": 2},
):
    op = BashOperator(task_id="hello_world", bash_command="Hello World!")
    print(op.retries)  # 2
```

## Декоратор @dag

*Добавлено в версии 2.0.*

Кроме контекстного менеджера и конструктора `DAG()` можно задекорировать функцию `@dag`, превратив её в фабрику DAG:

*Источник: [airflow/example_dags/example_dag_decorator.py](https://airflow.apache.org/docs/apache-airflow/stable/_modules/airflow/example_dags/example_dag_decorator.html)*

```python
from typing import TYPE_CHECKING, Any

import httpx
import pendulum

from airflow.providers.standard.operators.bash import BashOperator
from airflow.sdk import BaseOperator, dag, task

if TYPE_CHECKING:
    from airflow.sdk import Context


class GetRequestOperator(BaseOperator):
    """Кастомный оператор для GET-запроса по указанному url"""

    template_fields = ("url",)

    def __init__(self, *, url: str, **kwargs):
        super().__init__(**kwargs)
        self.url = url

    def execute(self, context: Context):
        return httpx.get(self.url).json()


@dag(
    schedule=None,
    start_date=pendulum.datetime(2021, 1, 1, tz="UTC"),
    catchup=False,
    tags=["example"],
)
def example_dag_decorator(url: str = "https://httpbingo.org/get"):
    """
    DAG для получения IP и вывода через BashOperator.

    :param url: URL для получения IP. По умолчанию "https://httpbingo.org/get".
    """
    get_ip = GetRequestOperator(task_id="get_ip", url=url)

    @task(multiple_outputs=True)
    def prepare_command(raw_json: dict[str, Any]) -> dict[str, str]:
        external_ip = raw_json["origin"]
        try:
            ipaddress.ip_address(external_ip)
            return {
                "command": f"echo 'Seems like today your server executing Airflow is connected from IP {external_ip}'",
            }
        except ValueError:
            raise ValueError(f"Invalid IP address: '{external_ip}'.")

    command_info = prepare_command(get_ip.output)

    BashOperator(task_id="echo_ip_info", bash_command=command_info["command"])


example_dag = example_dag_decorator()
```

Декоратор не только упрощает объявление DAG, но и превращает параметры функции в параметры DAG, которые можно [задавать при запуске DAG](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/dag-run.html#dagrun-parameters). Доступ к ним — из Python или из шаблона Jinja `{{ context.params }}` (см. [Jinja-шаблонирование](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/operators.html#concepts-jinja-templating)).

> **Примечание.** Airflow загружает только те DAG, которые находятся на верхнем уровне файла. Функцию с `@dag` недостаточно объявить — её нужно хотя бы один раз вызвать и присвоить результат переменной верхнего уровня, как в примере выше.

## Управление потоком (Control Flow)

По умолчанию задача запускается только когда все её зависимости успешно завершились. Это можно менять:

- **Branching** — выбор следующей задачи по условию.
- **Trigger Rules** — условия запуска задачи.
- [**Setup and Teardown**](https://airflow.apache.org/docs/apache-airflow/stable/howto/setup-and-teardown.html) — связи setup/teardown.
- **Latest Only** — особая форма ветвления: выполнение только для «текущего» Dag Run.
- **Depends On Past** — задача может зависеть от своего предыдущего запуска.

### Ветвление (Branching)

Ветвление позволяет не запускать все зависимые задачи, а выбрать одну или несколько веток. Для этого используется декоратор `@task.branch`.

Он похож на `@task`, но задекорированная функция должна вернуть **task_id** (или список task_id). Будут выполнены только указанные задачи, остальные ветки пропускаются. Можно вернуть `None` — тогда все downstream-задачи будут пропущены.

Возвращённый task_id должен относиться к задаче, которая является **прямым** downstream от задачи с `@task.branch`.

> **Примечание.** Если задача является downstream и от ветвящего оператора, и от одной или нескольких выбранных веток, она **не** будет пропущена:
>
> Ветки задачи с ветвлением: `branch_a`, `join` и `branch_b`. Так как `join` — downstream от `branch_a`, она всё равно выполнится, даже если не была возвращена в решении ветвления.

`@task.branch` можно использовать с XCom, чтобы выбирать ветку по результатам вышестоящих задач:

```python
@task.branch(task_id="branch_task")
def branch_func(ti=None):
    xcom_value = int(ti.xcom_pull(task_ids="start_task"))
    if xcom_value >= 5:
        return "continue_task"
    elif xcom_value >= 3:
        return "stop_task"
    else:
        return None


start_op = BashOperator(
    task_id="start_task",
    bash_command="echo 5",
    do_xcom_push=True,
    dag=dag,
)

branch_op = branch_func()

continue_op = EmptyOperator(task_id="continue_task", dag=dag)
stop_op = EmptyOperator(task_id="stop_task", dag=dag)

start_op >> branch_op >> [continue_op, stop_op]
```

Собственный оператор с ветвлением можно реализовать, наследуясь от `BaseBranchOperator`: он ведёт себя аналогично `@task.branch`, но требует реализации метода `choose_branch`.

> **Примечание.** Рекомендуется использовать декоратор `@task.branch`, а не создавать [BranchPythonOperator](https://airflow.apache.org/docs/apache-airflow-providers-standard/stable/_api/airflow/providers/standard/operators/python/index.html#airflow.providers.standard.operators.python.BranchPythonOperator) напрямую. Класс оператора обычно переопределяют только для кастомной реализации.

Метод `choose_branch` может возвращать task_id одной downstream-задачи или список task_id; остальные ветки пропускаются. Возврат `None` пропускает все downstream-задачи:

```python
class MyBranchOperator(BaseBranchOperator):
    def choose_branch(self, context):
        """В первый день месяца — дополнительная ветка."""
        if context['data_interval_start'].day == 1:
            return ['daily_task_id', 'monthly_task_id']
        elif context['data_interval_start'].day == 2:
            return 'daily_task_id'
        else:
            return None
```

Аналогично `@task.branch` для обычного Python есть декораторы с виртуальным окружением `@task.branch_virtualenv` и внешним Python `@task.branch_external_python`.

### Latest Only

Dag Run часто запускают за дату, отличную от текущей — например, по одному запуску за каждый день прошлого месяца (backfill).

Иногда нужно, чтобы часть (или все) задачи не выполнялись за прошлые даты. Для этого служит `LatestOnlyOperator`.

Он пропускает все задачи downstream от себя, если текущий Dag Run не «последний» (т.е. текущее время не между execution_time этого запуска и следующим запланированным execution_time, и запуск не был внешним).

Пример:

*Источник: [airflow/example_dags/example_latest_only_with_trigger.py](https://airflow.apache.org/docs/apache-airflow/stable/_modules/airflow/example_dags/example_latest_only_with_trigger.html)*

```python
import datetime

import pendulum

from airflow.providers.standard.operators.empty import EmptyOperator
from airflow.providers.standard.operators.latest_only import LatestOnlyOperator
from airflow.sdk import DAG, TriggerRule

with DAG(
    dag_id="latest_only_with_trigger",
    schedule=datetime.timedelta(hours=4),
    start_date=pendulum.datetime(2021, 1, 1, tz="UTC"),
    catchup=False,
    tags=["example3"],
) as dag:
    latest_only = LatestOnlyOperator(task_id="latest_only")
    task1 = EmptyOperator(task_id="task1")
    task2 = EmptyOperator(task_id="task2")
    task3 = EmptyOperator(task_id="task3")
    task4 = EmptyOperator(task_id="task4", trigger_rule=TriggerRule.ALL_DONE)

    latest_only >> task1 >> [task3, task4]
    task2 >> [task3, task4]
```

В этом DAG:

- `task1` — прямой downstream от `latest_only`, пропускается во всех запусках, кроме последнего.
- `task2` не зависит от `latest_only` и выполняется во всех запланированных периодах.
- `task3` — downstream от `task1` и `task2`; при правиле по умолчанию `all_success` получает каскадный skip от `task1`.
- `task4` — downstream от `task1` и `task2`, но **не** пропускается, так как у неё `trigger_rule=TriggerRule.ALL_DONE`.

### Depends On Past

Можно указать, что задача выполняется только если её **предыдущий** запуск в **предыдущем** Dag Run завершился успешно. Для этого у задачи задаётся аргумент `depends_on_past=True`.

При самом первом автоматическом запуске DAG задачи с `depends_on_past` всё равно выполнятся — предыдущего запуска ещё нет.

### Trigger Rules

По умолчанию Airflow ждёт [успешного](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/tasks.html#concepts-task-states) завершения всех upstream (прямых родителей) задачи перед её запуском.

Поведение задаётся аргументом `trigger_rule` задачи. Варианты:

| Правило | Описание |
|--------|----------|
| `all_success` (по умолчанию) | Все upstream-задачи успешны |
| `all_failed` | Все upstream в состоянии `failed` или `upstream_failed` |
| `all_done` | Все upstream завершили выполнение |
| `all_done_min_one_success` | Все не пропущенные upstream завершены и хотя бы одна успешна |
| `all_skipped` | Все upstream в состоянии `skipped` |
| `one_failed` | Хотя бы одна upstream завершилась с ошибкой (не ждёт завершения всех) |
| `one_success` | Хотя бы одна upstream успешна (не ждёт завершения всех) |
| `one_done` | Хотя бы одна upstream успешна или упала |
| `none_failed` | Ни одна upstream не в `failed`/`upstream_failed` (все успешны или пропущены) |
| `none_failed_min_one_success` | Как выше, и хотя бы одна upstream успешна |
| `none_skipped` | Нет пропущенных upstream — все в `success`, `failed` или `upstream_failed` |
| `always` | Зависимостей нет, задача может запускаться в любой момент |

Их можно комбинировать с Depends On Past.

> **Примечание.** Важно учитывать взаимодействие trigger rules и пропущенных (skipped) задач, особенно при ветвлении. Почти никогда не стоит использовать `all_success` или `all_failed` downstream от ветвящейся задачи.
>
> Пропуск распространяется по цепочке при правилах `all_success` и `all_failed`. Пример DAG:

```python
# dags/branch_without_trigger.py
import pendulum

from airflow.sdk import task
from airflow.sdk import DAG
from airflow.providers.standard.operators.empty import EmptyOperator

dag = DAG(
    dag_id="branch_without_trigger",
    schedule="@once",
    start_date=pendulum.datetime(2019, 2, 28, tz="UTC"),
)

run_this_first = EmptyOperator(task_id="run_this_first", dag=dag)


@task.branch(task_id="branching")
def do_branching():
    return "branch_a"


branching = do_branching()

branch_a = EmptyOperator(task_id="branch_a", dag=dag)
follow_branch_a = EmptyOperator(task_id="follow_branch_a", dag=dag)

branch_false = EmptyOperator(task_id="branch_false", dag=dag)

join = EmptyOperator(task_id="join", dag=dag)

run_this_first >> branching
branching >> branch_a >> follow_branch_a >> join
branching >> branch_false >> join
```

`join` — downstream от `follow_branch_a` и `branch_false`. Задача `join` будет помечена как skipped: по умолчанию у неё `trigger_rule=all_success`, а пропуск из-за ветвления передаётся по цепочке.

Если задать для `join` `trigger_rule=none_failed_min_one_success`, получится нужное поведение.

### Setup и teardown

В рабочих процессах часто создают ресурс (например, вычислительный), используют его и затем освобождают. В Airflow для этого есть задачи setup и teardown.

Подробности: [Setup and Teardown](https://airflow.apache.org/docs/apache-airflow/stable/howto/setup-and-teardown.html).

## Динамические DAG

DAG задаётся Python-кодом, поэтому он не обязан быть чисто декларативным: можно использовать циклы, функции и т.д.

Пример DAG с циклом `for`:

```python
with DAG("loop_example", ...):
    first = EmptyOperator(task_id="first")
    last = EmptyOperator(task_id="last")

    options = ["branch_a", "branch_b", "branch_c", "branch_d"]
    for option in options:
        t = EmptyOperator(task_id=option)
        first >> t >> last
```

Рекомендуется по возможности сохранять топологию (расположение) задач стабильной; динамику лучше использовать для загрузки конфигурации или изменения параметров операторов.

## Визуализация DAG

Варианты посмотреть DAG в виде графа:

1. Открыть UI Airflow, перейти к DAG и выбрать вид «Graph».
2. Выполнить `airflow dags show` — будет сгенерирован файл-картинка.

Предпочтительнее вид Graph в UI: в нём видно состояние всех [Task Instances](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/tasks.html#concepts-task-instances) выбранного Dag Run.

Для усложняющихся DAG есть способы упростить отображение.

### TaskGroups

**TaskGroup** позволяет сгруппировать задачи иерархически в виде Graph. Это уменьшает визуальный шум и удобно для повторяющихся паттернов.

Задачи внутри TaskGroup остаются в том же DAG и подчиняются его настройкам и пулам.

См. также: [TaskGroup](https://airflow.apache.org/docs/task-sdk/stable/api.html#airflow.sdk.TaskGroup) и `task_group` в API.

Зависимости между задачами в TaskGroup задаются теми же `>>` и `<<`:

```python
from airflow.sdk import task_group


@task_group()
def group1():
    task1 = EmptyOperator(task_id="task1")
    task2 = EmptyOperator(task_id="task2")


task3 = EmptyOperator(task_id="task3")

group1() >> task3
```

У TaskGroup тоже есть `default_args`; они переопределяют `default_args` уровня DAG:

```python
import datetime

from airflow.sdk import DAG
from airflow.sdk import task_group
from airflow.providers.standard.operators.bash import BashOperator
from airflow.providers.standard.operators.empty import EmptyOperator

with DAG(
    dag_id="dag1",
    start_date=datetime.datetime(2016, 1, 1),
    schedule="@daily",
    default_args={"retries": 1},
):

    @task_group(default_args={"retries": 3})
    def group1():
        """Docstring станет подсказкой (tooltip) для TaskGroup."""
        task1 = EmptyOperator(task_id="task1")
        task2 = BashOperator(task_id="task2", bash_command="echo Hello World!", retries=2)
        print(task1.retries)  # 3
        print(task2.retries)  # 2
```

Более сложный пример — `example_task_group_decorator.py` в поставке Airflow.

> **Примечание.** По умолчанию дочерние задачи и TaskGroup получают префикс `group_id` родительской группы. Так сохраняется уникальность group_id и task_id в DAG.
>
> Чтобы отключить префикс, при создании TaskGroup укажите `prefix_group_id=False`; тогда вы сами должны обеспечить уникальность всех task_id и group_id.

> **Примечание.** При использовании `@task_group` docstring функции используется как подсказка TaskGroup в UI, если не задано явное значение `tooltip`.

### Подписи на рёбрах (Edge Labels)

Кроме группировки можно подписывать рёбра зависимостей в виде Graph — особенно полезно в местах ветвления, чтобы обозначить условия перехода по веткам.

Подпись можно задать прямо в цепочке с `>>` и `<<`:

```python
from airflow.sdk import Label

my_task >> Label("When empty") >> other_task
```

Или передать `Label` в `set_upstream`/`set_downstream`:

```python
from airflow.sdk import Label

my_task.set_downstream(other_task, Label("When empty"))
```

Пример DAG с подписями веток:

*Источник: [airflow/example_dags/example_branch_labels.py](https://airflow.apache.org/docs/apache-airflow/stable/_modules/airflow/example_dags/example_branch_labels.html)*

![Пример DAG с подписями на рёбрах в виде Graph.](images/edge_label_example.png)

```python
with DAG(
    "example_branch_labels",
    schedule="@daily",
    start_date=pendulum.datetime(2021, 1, 1, tz="UTC"),
    catchup=False,
) as dag:
    ingest = EmptyOperator(task_id="ingest")
    analyse = EmptyOperator(task_id="analyze")
    check = EmptyOperator(task_id="check_integrity")
    describe = EmptyOperator(task_id="describe_integrity")
    error = EmptyOperator(task_id="email_error")
    save = EmptyOperator(task_id="save")
    report = EmptyOperator(task_id="report")

    ingest >> analyse >> check
    check >> Label("No errors") >> save >> report
    check >> Label("Errors found") >> describe >> error >> report
```

## Документация DAG и задач

К DAG и задачам можно добавлять описание и заметки, которые отображаются в веб-интерфейсе (вкладки «Graph» и «Tree» для DAG, «Task Instance Details» для задач).

У задач есть специальные атрибуты, которые рендерятся как форматированный контент:

| Атрибут   | Отображение      |
|----------|-------------------|
| doc      | моноширинный текст |
| doc_json | JSON              |
| doc_yaml | YAML              |
| doc_md   | Markdown          |
| doc_rst  | reStructuredText  |

У DAG интерпретируется только атрибут `doc_md`. Он может быть строкой или путём к Markdown-файлу (строка, оканчивающаяся на `.md`). Относительный путь разрешается от каталога, из которого запущен Scheduler или парсер DAG. Если файл не найден, переданное имя будет показано как текст, без исключения. Содержимое файла загружается при разборе DAG; изменения в файле подхватятся за один цикл разбора.

Это удобно, когда задачи строятся динамически из конфигурации — можно показать в Airflow конфиг, из которого получились задачи:

```python
"""
### My great Dag
"""

import pendulum

dag = DAG(
    "my_dag",
    start_date=pendulum.datetime(2021, 1, 1, tz="UTC"),
    schedule="@daily",
    catchup=False,
)
dag.doc_md = __doc__

t = BashOperator("foo", dag=dag)
t.doc_md = """\
#Title"
Here's a [url](www.airbnb.com)
"""
```

## Упаковка DAG

Простые DAG часто помещаются в один файл; сложные могут быть разнесены по нескольким файлам и иметь зависимости («vendored»).

Варианты:

1. Всё внутри Dag bundle с обычной файловой структурой.
2. DAG и все нужные Python-файлы упакованы в один zip. Пример содержимого:

```
my_dag1.py
my_dag2.py
package1/__init__.py
package1/functions.py
```

Ограничения упакованных DAG:

- Нельзя использовать при включённой сериализации через pickle.
- Внутри только чистый Python, без скомпилированных библиотек (например, `libz.so`).
- Содержимое попадает в `sys.path` и может импортироваться любым кодом в процессе Airflow — имена пакетов не должны конфликтовать с уже установленными.

При сложных скомпилированных зависимостях обычно лучше использовать виртуальное окружение Python и устанавливать пакеты через `pip` на целевых системах.

## .airflowignore

Файл `.airflowignore` задаёт каталоги и файлы в Dag bundle (или в `PLUGINS_FOLDER`), которые Airflow должен игнорировать. Поддерживаются два синтаксиса шаблонов (параметр конфигурации `DAG_IGNORE_FILE_SYNTAX`, с версии 2.3): `regexp` и `glob`.

> **Примечание.** По умолчанию в Airflow 3 и новее используется `glob` (ранее был `regexp`).

**Синтаксис glob (по умолчанию)** — как в `.gitignore`:

- `*` — любое число символов, кроме `/`.
- `?` — один любой символ, кроме `/`.
- Диапазоны, например `[a-zA-Z]`.
- Отрицание через префикс `!`; порядок важен, более позднее правило может отменить предыдущее (в том числе из родительского каталога).
- `**` — совпадение по каталогам на любую глубину (например, `**/__pycache__/`).

Если в шаблоне есть `/` в начале или в середине, он считается относительно каталога, в котором лежит данный `.airflowignore`. Иначе шаблон может совпадать на любом уровне ниже.

**Синтаксис regexp**: каждая строка — регулярное выражение; каталоги и файлы, чьи имена (не dag_id) совпадают с любым шаблоном, игнорируются (используется `Pattern.search()`). Строки, начинающиеся с `#`, считаются комментариями.

`.airflowignore` кладут в Dag bundle. Пример с синтаксисом glob:

```
**/*project_a*
tenant_[0-9]*
```

Тогда файлы вроде `project_a_dag_1.py`, `TESTING_project_a.py`, `tenant_1.py`, `project_a/dag_1.py`, `tenant_1/dag_1.py` будут проигнорированы. Если имя каталога совпадает с шаблоном, каталог и все подкаталоги не сканируются — это ускоряет поиск DAG.

Действие `.airflowignore` распространяется на свой каталог и все подкаталоги. Отдельный `.airflowignore` в подкаталоге действует только для него.

## Зависимости между DAG

*Добавлено в Airflow 2.1.*

Зависимости между задачами внутри DAG задаются явно (upstream/downstream). Зависимости между разными DAG устроены сложнее. Основные способы:

- **Запуск (triggering)** — [TriggerDagRunOperator](https://airflow.apache.org/docs/apache-airflow-providers-standard/stable/_api/airflow/providers/standard/operators/trigger_dagrun/index.html#airflow.providers.standard.operators.trigger_dagrun.TriggerDagRunOperator).
- **Ожидание (waiting)** — `ExternalTaskSensor`.

Дополнительно один DAG может ждать или запускать несколько запусков другого DAG с разными data interval. Зависимости между DAG отображаются в **Dag Dependencies** (`Menu → Browse → Dag Dependencies`). Они вычисляются планировщиком при сериализации DAG; веб-сервер строит по ним граф.

Детектор зависимостей настраивается — можно реализовать свою логику вместо стандартной в `DependencyDetector`.

## Пауза, деактивация и удаление DAG

У DAG есть несколько состояний «не запущен»: пауза, деактивация и полное удаление метаданных.

**Пауза.** DAG можно приостановить через UI, если он есть в `DAGS_FOLDER` и планировщик сохранил его в БД, но пользователь отключил его в UI. Действия «pause» и «unpause» доступны в UI и API. Приостановленные DAG не планируются, но их можно запускать вручную из UI. В UI приостановленные DAG отображаются во вкладке «Paused», активные — в «Active». При паузе уже выполняющиеся задачи дорабатываются, все downstream переходят в состояние «Scheduled». При снятии паузы задачи в «Scheduled» начнут выполняться по логике DAG; если таких нет, DAG будет запускаться по расписанию.

**Деактивация** (не путать с вкладкой «Active» в UI) происходит при удалении DAG из `DAGS_FOLDER`. Когда планировщик при разборе каталога перестаёт видеть DAG, который раньше был в БД, он помечает его как деактивированный. Метаданные и история деактивированного DAG сохраняются; при возврате файла в `DAGS_FOLDER` DAG снова станет активным и история будет видна. Деактивировать/активировать DAG через UI или API нельзя — только удалением или добавлением файлов в `DAGS_FOLDER`. Данные по прошлым запускам при деактивации не теряются. Вкладка «Active» в UI показывает DAG, которые и активированы, и не на паузе — это может поначалу сбивать с толку.

Деактивированные DAG в UI не отображаются; иногда видны их прошлые запуски, но при переходе к ним появляется ошибка об отсутствии DAG.

**Удаление метаданных** из БД через UI или API не всегда приводит к исчезновению DAG из UI. Если файл DAG по-прежнему в `DAGS_FOLDER`, планировщик при разборе снова его подхватит; удалится только информация о прошлых запусках.

Чтобы полностью удалить DAG и всю его историю, нужно:

1. Поставить DAG на паузу.
2. Удалить метаданные из БД через UI или API.
3. Удалить файл DAG из `DAGS_FOLDER` и дождаться деактивации.

## Автоматическая пауза DAG (экспериментально)

DAG можно настроить на автоматическую паузу. В конфигурации Airflow есть параметр, отключающий DAG после N подряд неуспешных запусков:

[max_consecutive_failed_dag_runs_per_dag](https://airflow.apache.org/docs/apache-airflow/stable/configurations-ref.html#config-core-max-consecutive-failed-dag-runs-per-dag)

Переопределить значение можно аргументом DAG:

- **`max_consecutive_failed_dag_runs`** — переопределяет [max_consecutive_failed_dag_runs_per_dag](https://airflow.apache.org/docs/apache-airflow/stable/configurations-ref.html#config-core-max-consecutive-failed-dag-runs-per-dag).

## Deadline Alerts (оповещения о дедлайнах)

*Добавлено в версии 3.1.*

**Deadline Alerts** позволяют задать временные пороги для запусков DAG и автоматически реагировать при их превышении. Дедлайн можно задать относительно фиксированной даты/времени, одного из встроенных моментов (например, время постановки в очередь или начала Dag Run) или своей реализации. При превышении дедлайна вызывается callback (уведомление или другое действие).

Пример с email-нотификатором:

```python
from datetime import timedelta
from airflow import DAG
from airflow.providers.smtp.notifications.smtp import SmtpNotifier
from airflow.sdk.definitions.deadline import DeadlineAlert, DeadlineReference

with DAG(
    dag_id="email_deadline",
    deadline=DeadlineAlert(
        reference=DeadlineReference.DAGRUN_QUEUED_AT,
        interval=timedelta(minutes=30),
        callback=SmtpNotifier(
            to="team@example.com",
            subject="🚨 Dag {{ dag_run.dag_id }} missed deadline at {{ deadline.deadline_time }}",
            html_content="The Dag Run {{ dag_run.dag_run_id }} has been running for more than 30 minutes since being queued.",
        ),
    ),
):
    EmptyOperator(task_id="task1")
```

В этом примере письмо отправляется, если DAG не завершился в течение 30 минут после постановки в очередь.

Подробнее: [Deadline Alerts](https://airflow.apache.org/docs/apache-airflow/stable/howto/deadline-alerts.html).

---

*Источник: [Airflow 3.1.7 — Dags](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/dags.html). Перевод неофициальный.*
