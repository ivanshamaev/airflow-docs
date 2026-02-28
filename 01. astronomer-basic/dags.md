# Введение в DAG в Apache Airflow®

В [Apache Airflow®](https://airflow.apache.org/) **DAG** — это пайплайн или рабочий процесс. DAG — основная организационная единица в Airflow: набор задач и зависимостей, которые нужно выполнять по расписанию.

DAG задаётся в Python-коде и отображается в Airflow UI. DAG может быть от одной задачи до сотен и тысяч с сложными зависимостями.

На снимке ниже показан [граф сложного DAG run](https://www.astronomer.io/docs/learn/dags#complex-dag-run) в Airflow UI. После прочтения руководства вы сможете разбираться в элементах этого графа, а также знать, как задавать DAG и использовать параметры DAG.

## Необходимая база

Чтобы получить максимум от руководства, нужно понимать:

- Что такое Airflow. См. [Введение в Apache Airflow](https://www.astronomer.io/docs/learn/intro-to-airflow).

## Что такое DAG?

**DAG** (directed acyclic graph — ориентированный ациклический граф) — математическая структура из узлов и рёбер. В Airflow DAG представляет пайплайн или рабочий процесс с началом и концом.

Свойства DAG делают их удобными для построения пайплайнов:

- **Graph (граф)**: DAG — это граф, структура из узлов и рёбер. В Airflow узлы — задачи, рёбра — зависимости между задачами. Задание workflow как графа помогает визуализировать весь процесс и легко в нём ориентироваться.
- **Acyclic (ациклический)**: в DAG нет циклических зависимостей. Задача не может зависеть от себя и не может зависеть от задачи, которая в итоге зависит от неё.
- **Directed (направленный)**: между задачами есть явное направление потока. Задача может быть выше по потоку (upstream), ниже по потоку (downstream) или параллельна другой задаче.

Помимо этих требований, DAG может быть сколь угодно простым или сложным: задачи можно выполнять параллельно или последовательно, добавлять условные ветвления и визуально группировать задачи в [группы задач (task groups)](https://www.astronomer.io/docs/learn/task-groups).

Каждая задача в DAG выполняет одну единицу работы — от простой Python-функции до сложного преобразования или вызова внешнего сервиса. Задачи задаются [операторами Airflow](operators.md) или [декораторами Airflow](../02.%20astronomer-dags/airflow-decorators.md). Зависимости между задачами задаются разными способами (см. [Управление зависимостями](task-dependencies.md)).

На снимке ниже — простой граф DAG с тремя последовательными задачами.

## Что такое DAG run?

**DAG run** — экземпляр DAG в конкретный момент времени. **Task instance** — экземпляр задачи в конкретный момент времени. У каждого DAG run уникальный `run_id`; в него входит одна или несколько task instances. История прошлых DAG run хранится в [метаданных Airflow](../03.%20astronomer-infra/airflow-database.md).

В Airflow UI прошлые запуски DAG отображаются в виде **Grid**; по клику на полосу длительности открывается конкретный DAG run.

![Вид Grid для DAG с 3 задачами.](https://files.buildwithfern.com/astronomer.docs.buildwithfern.com/docs/6b8eaf260e6e432185783e305cbae176cef493cbcbeae4fe5bd66421f384c37d/docs/assets/img/guides/3-0-dags_grid_view.png)

Граф DAG run похож на граф DAG, но дополнен информацией о состоянии каждой task instance в этом запуске.

![Снимок графа DAG run с 3 задачами: get_astronauts, print_astronaut_craft (динамически маппленная задача с 12 экземплярами) и print_reaction.](https://files.buildwithfern.com/astronomer.docs.buildwithfern.com/docs/8b5f9b6498960a29b94b7a2ce6b1a7e9407ff28f131fc07168485fe11e84bb6c/docs/assets/img/guides/3-0-dags_simple_dag_run_graph.png)

Подробнее о навигации в Airflow UI: [Введение в интерфейс Airflow](airflow-ui.md).

Каждый DAG run связан с **версией DAG**. При каждом структурном изменении DAG и создании нового DAG run версия увеличивается. Так можно отслеживать изменения DAG во времени и лучше понимать прошлые запуски. Подробнее: [Версионирование DAG и DAG Bundles](https://www.astronomer.io/docs/learn/airflow-dag-versioning).

### Свойства DAG run

Граф DAG run в UI содержит информацию о DAG run и состоянии каждой task instance. На снимке ниже те же элементы графа с поясняющими подписями.

![Снимок Airflow UI: DAG run с 3 задачами, подписи к dag_id, logical date, task_id, task state, оператору/декоратору, числу динамически маппленных экземпляров в [], зависимостям.](https://files.buildwithfern.com/astronomer.docs.buildwithfern.com/docs/06356cbfa0ea2ec3316585eb846f72e9b061e1072eab32500dc964e434180fb1/docs/assets/img/guides/3-0-dags_simple_dag_run_graph_annotated.png)

- **dag_id**: уникальный идентификатор DAG.
- **logical date**: момент времени, после которого этот DAG run может быть выполнен. Эта дата и время не обязательно совпадают с фактическим моментом выполнения. Подробнее: [Планирование](scheduling.md).
- **task_id**: уникальный идентификатор задачи.
- **task state**: состояние task instance в DAG run. Возможные состояния: `running`, `success`, `failed`, `skipped`, `restarting`, `up_for_retry`, `upstream_failed`, `queued`, `scheduled`, `none`, `removed`, `deferred`, `up_for_reschedule`; каждое отображается своим цветом рамки узла. Подробнее: [task instances](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/tasks.html#task-instances) в документации Airflow.

Запустить DAG run можно четырьмя способами:

- **Backfill**: [бэкфилл](../02.%20astronomer-dags/rerunning-dags.md) — создание нескольких DAG run за прошлые даты через UI, API или CLI. У бэкфилл-запусков на полосе длительности отображается изогнутая стрелка.
- **Scheduled**: DAG run по расписанию DAG (например `@daily`, `@hourly`) создаёт планировщик Airflow. На полосе длительности нет дополнительной иконки.
- **Manual**: ручной запуск из UI, CLI или API. У ручных запусков на полосе — иконка воспроизведения.
- **Asset triggered**: DAG можно планировать по данным (data-aware scheduling): запуск при обновлении одного или нескольких [ассетов Airflow](assets.md). Обновления могут приходить из задач того же экземпляра Airflow, из вызова [REST API](https://airflow.apache.org/docs/apache-airflow/stable/stable-rest-api-ref.html), вручную из UI или по [сообщениям в очереди](https://www.astronomer.io/docs/learn/airflow-event-driven-scheduling). На полосе — иконка ассета.

У DAG run могут быть следующие статусы:

- **Queued**: время, после которого DAG run может быть создан, наступило, но планировщик ещё не создал для него task instances.
- **Running**: DAG run допущен к планированию task instances.
- **Success**: все task instances в терминальном состоянии (`success`, `skipped`, `failed`, `upstream_failed`), и все листовые задачи (без downstream) в состоянии `success` или `skipped`. Полоса успешного DAG run — зелёная.
- **Failed**: все task instances в терминальном состоянии и хотя бы одна листовая задача в `failed` или `upstream_failed`. Полоса неуспешного DAG run — красная.

### Сложные DAG run

В более сложных DAG в графе DAG run отображаются дополнительные возможности Airflow. На снимке ниже тот же сложный DAG с подписями к элементам графа. Не страшно, если не всё знакомо — вы узнаете об этом по мере освоения Airflow.

![Снимок сложного DAG run с подписями: динамически маппленная задача, ветвящаяся задача, подпись на ребре, динамически маппленная группа задач, обычные группы задач, setup/teardown задачи, Dataset.](https://files.buildwithfern.com/astronomer.docs.buildwithfern.com/docs/417770b39097a8a7375a9bd79d45ffbbaccc9d737468705e25636a459208fa0c/docs/assets/img/guides/3-0-dags_complex_DAG_annotated.png)

В таком графе могут быть:

- **Динамически маппленные задачи**: задача [создаётся динамически](../02.%20astronomer-dags/dynamic-tasks.md) в runtime по заданному входу. Число экземпляров показывается в квадратных скобках `[]` после task ID.
- **Ветвящиеся задачи**: задача создаёт условную ветвь в DAG. См. [BranchOperator в Airflow](../02.%20astronomer-dags/branch-operator.md).
- **Подписи на рёбрах (edge labels)**: подписи на ребре между двумя задачами, часто для пояснения решений ветвления.
- **Группы задач (task groups)**: логическая и визуальная группировка задач в DAG. См. [Группы задач](../02.%20astronomer-dags/task-groups.md).
- **Setup/teardown задачи**: при управлении инфраструктурой через Airflow задачи можно объявлять setup и teardown для особого поведения зависимостей. В графе они отображаются диагональными стрелками и пунктирной линией. См. [Setup и teardown задачи](https://www.astronomer.io/docs/learn/airflow-setup-teardown).
- **Ассеты**: ассеты отображаются в графе DAG. Если DAG запланирован по ассету, он показан выше первой задачи. Если задача обновляет ассет, он показан после этой задачи. См. [Ассеты Airflow](assets.md).

Подробнее о сложных зависимостях между задачами и группами: [Управление зависимостями](task-dependencies.md).

**Пример кода сложного DAG** (как на снимке выше):

```python
"""
Пример сложной структуры DAG: ветвления с подписями, группы задач,
динамически маппленные задачи и группы, setup/teardown.
"""

from airflow.decorators import dag, task_group, task
from airflow.operators.empty import EmptyOperator
from airflow.operators.bash import BashOperator
from airflow.models.baseoperator import chain, chain_linear
from airflow.utils.edgemodifier import Label
from airflow.datasets import Dataset
from pendulum import datetime


@dag(
    start_date=datetime(2024, 1, 1),
    schedule=None,
    catchup=False,
)
def complex_dag_structure():

    start = EmptyOperator(task_id="start")

    sales_data_extract = BashOperator.partial(task_id="sales_data_extract").expand(
        bash_command=["echo 1", "echo 2", "echo 3", "echo 4"]
    )
    internal_api_extract = BashOperator.partial(task_id="internal_api_extract").expand(
        bash_command=["echo 1", "echo 2", "echo 3", "echo 4"]
    )

    @task.branch
    def determine_load_type() -> str:
        import random
        if random.choice([True, False]):
            return "internal_api_load_full"
        return "internal_api_load_incremental"

    sales_data_transform = EmptyOperator(task_id="sales_data_transform")
    determine_load_type_obj = determine_load_type()

    sales_data_load = EmptyOperator(task_id="sales_data_load")
    internal_api_load_full = EmptyOperator(task_id="internal_api_load_full")
    internal_api_load_incremental = EmptyOperator(
        task_id="internal_api_load_incremental"
    )

    @task_group
    def sales_data_reporting(a):
        prepare_report = EmptyOperator(
            task_id="prepare_report", trigger_rule="all_done"
        )
        publish_report = EmptyOperator(task_id="publish_report")
        chain(prepare_report, publish_report)

    sales_data_reporting_obj = sales_data_reporting.expand(a=[1, 2, 3, 4, 5, 6])

    @task_group
    def cre_integration():
        cre_extract = EmptyOperator(task_id="cre_extract", trigger_rule="all_done")
        cre_transform = EmptyOperator(task_id="cre_transform")
        cre_load = EmptyOperator(task_id="cre_load")
        chain(cre_extract, cre_transform, cre_load)

    cre_integration_obj = cre_integration()

    @task_group
    def mlops():
        set_up_cluster = EmptyOperator(
            task_id="set_up_cluster", trigger_rule="all_done"
        )
        train_model = EmptyOperator(
            task_id="train_model", outlets=[Dataset("model_trained")]
        )
        tear_down_cluster = EmptyOperator(task_id="tear_down_cluster")
        chain(set_up_cluster, train_model, tear_down_cluster)
        tear_down_cluster.as_teardown(setups=set_up_cluster)

    mlops_obj = mlops()
    end = EmptyOperator(task_id="end")

    chain(
        start,
        sales_data_extract,
        sales_data_transform,
        sales_data_load,
        [sales_data_reporting_obj, cre_integration_obj],
        end,
    )
    chain(
        start,
        internal_api_extract,
        determine_load_type_obj,
        [internal_api_load_full, internal_api_load_incremental],
        mlops_obj,
        end,
    )
    chain_linear(
        [sales_data_load, internal_api_load_full],
        [sales_data_reporting_obj, cre_integration_obj],
    )
    chain(
        determine_load_type_obj, Label("additional data"), internal_api_load_incremental
    )
    chain(
        determine_load_type_obj, Label("changed existing data"), internal_api_load_full
    )


complex_dag_structure()
```

## Написание DAG

DAG задаётся Python-файлом в [DAG bundle](https://www.astronomer.io/docs/learn/airflow-dag-versioning) проекта Airflow. При использовании [Astro CLI](https://www.astronomer.io/docs/astro/cli/get-started-cli) с настройками по умолчанию это папка `dags`. Airflow автоматически разбирает файлы в этой папке каждые 5 минут на предмет новых DAG и каждые 30 секунд — существующие DAG на изменения кода. Принудительный разбор: [`airflow dags reserialize`](https://airflow.apache.org/docs/apache-airflow/stable/cli-and-env-variables-ref.html#reserialize) или `astro dev run dags reserialize` в Astro CLI.

Структурировать DAG можно двумя синтаксисами:

- **TaskFlow API**: декоратор `@dag`. Функция с `@dag` задаёт DAG. В конце скрипта нужно **вызвать** функцию, чтобы Airflow зарегистрировал DAG. Все задачи задаются внутри функции DAG.
- **Традиционный синтаксис**: контекст `with DAG(...)` или экземпляр класса `DAG`, задачи внутри контекста.

TaskFlow API и традиционный синтаксис можно смешивать. См. [Смешивание декораторов TaskFlow с традиционными операторами](../02.%20astronomer-dags/airflow-decorators.md). Однозадачные DAG можно создавать декоратором `@asset`, см. [Ассеты](assets.md).

Ниже — один и тот же DAG в обоих вариантах синтаксиса.

**TaskFlow:**

```python
from airflow.sdk import dag, task, chain
from pendulum import datetime


@dag(
    start_date=datetime(2025, 4, 1),
    schedule="@daily",
)
def taskflow_dag():
    @task
    def my_task_1():
        import time

        time.sleep(5)
        print(1)

    @task
    def my_task_2():
        print(2)

    chain(my_task_1(), my_task_2())


taskflow_dag()
```

**Традиционный синтаксис:**

```python
from airflow.sdk import DAG
from airflow.providers.standard.operators.python import PythonOperator
from pendulum import datetime


def my_task_1_func():
    import time

    time.sleep(5)
    print(1)


with DAG(
    dag_id="traditional_syntax_dag",
    start_date=datetime(2025, 4, 1),
    schedule="@daily",
):
    my_task_1 = PythonOperator(
        task_id="my_task_1",
        python_callable=my_task_1_func,
    )

    my_task_2 = PythonOperator(
        task_id="my_task_2",
        python_callable=lambda: print(2),
    )

    my_task_1 >> my_task_2
```

> **Совет.** Astronomer рекомендует создавать один Python-файл на DAG и называть его по `dag_id` — это удобная практика организации проекта. В продвинутых сценариях DAG можно генерировать динамически, см. [Динамическая генерация DAG в Airflow](https://www.astronomer.io/docs/learn/dynamically-generating-dags).

### Параметры уровня DAG

В Airflow момент и способ запуска DAG настраиваются параметрами объекта DAG. Параметры уровня DAG влияют на поведение всего DAG в отличие от параметров уровня задачи.

В примерах выше заданы такие базовые параметры:

- **schedule**: расписание DAG. Варианты задания расписания см. в [Планирование в Airflow](scheduling.md). По умолчанию `None`.
- **start_date**: дата и время, после которых DAG начинает планироваться. По умолчанию `None`.
- **dag_id**: имя DAG, уникальное в окружении Airflow. При использовании декоратора `@dag` без явного параметра `dag_id` в качестве `dag_id` берётся имя функции.

Есть и другие параметры уровня DAG — от использования ресурсов до отображения в UI. Полный список: [Параметры уровня DAG](https://www.astronomer.io/docs/learn/airflow-dag-parameters).

## См. также

- [Операторы Airflow](operators.md) и [Введение в TaskFlow API и декораторы Airflow](../02.%20astronomer-dags/airflow-decorators.md) — как задавать задачи в DAG.
- [Get started with Apache Airflow](https://www.astronomer.io/docs/learn/get-started-with-airflow) — практическое введение в написание первого простого DAG.

---

[← Ассеты](assets.md) | [К содержанию](README.md) | [Планирование →](scheduling.md)
