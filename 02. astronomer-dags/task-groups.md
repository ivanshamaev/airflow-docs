# Группы задач (Task groups)

[Группы задач](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/dags.html#taskgroups) в Airflow позволяют организовывать задачи в группы внутри DAG. С их помощью можно:

- Превращать повторяющиеся паттерны задач в переиспользуемые модули для разных DAG или инстансов Airflow.
- [Динамически маппить](dynamic-tasks.md) группы задач и строить сложные динамические сценарии.
- Задавать `default_args` для набора задач вместо уровня всего DAG, см. [параметры DAG](https://www.astronomer.io/docs/learn/dags#dag-parameters).
- Упорядочивать сложные DAG: в представлении Grid в Airflow UI задачи визуально объединяются в группы.

В этом руководстве — как создавать и использовать группы задач в DAG. Примеры DAG с группами: [Astronomer GitHub](https://github.com/astronomer/webinar-task-groups).

## Необходимая база

Чтобы получить максимум от руководства, нужно понимать:

- Операторы Airflow. См. [Операторы 101](https://www.astronomer.io/docs/learn/what-is-an-operator).

## Когда использовать группы задач

Группы задач чаще всего используют для визуальной организации сложных DAG. Например:

- **Вход неизвестной длины** — например, неизвестное число файлов в каталоге. Группы задач позволяют [динамически маппить](task-groups.md#generate-task-groups-dynamically-at-runtime) по входу и создавать группу задач, выполняющую набор действий для каждого файла. Это единственный способ динамически маппить последовательные задачи в Airflow.
- **Один и тот же паттерн в нескольких DAG** — группа как переиспользуемый модуль.
- **DAG с несколькими командами** — группы визуально разделяют задачи по командам. В таком случае иногда лучше разнести логику по разным DAG и связать их через [Assets](https://www.astronomer.io/docs/learn/airflow-datasets).
- **MLOps DAG** — отдельная группа на каждую обучаемую модель.
- **Крупные ELT/ETL DAG** — группа на каждую таблицу или схему.

## Определение групп задач

Группы можно задавать двумя способами:

- Декоратор **`@task_group`** на Python-функции.
- Класс **`TaskGroup`** как контекстный менеджер.

В большинстве случаев выбор — вопрос предпочтений. Исключение — [динамический маппинг](dynamic-tasks.md) по группе: он возможен только с **`@task_group`**.

Ниже — простая группа из двух последовательных задач. Операторы зависимостей (`<<` и `>>`) работают внутри групп и между группами так же, как для отдельных задач.

```python
# from airflow.decorators import task_group

t0 = EmptyOperator(task_id='start')

# Начало определения группы задач
@task_group(group_id='my_task_group')
def tg1():
    t1 = EmptyOperator(task_id='task_1')
    t2 = EmptyOperator(task_id='task_2')

    t1 >> t2
# Конец определения группы задач

t3 = EmptyOperator(task_id='end')

# Задание зависимостей группы (tg1)
t0 >> tg1() >> t3
```

```python
# from airflow.utils.task_group import TaskGroup

t0 = EmptyOperator(task_id='start')

# Начало определения группы задач
with TaskGroup(group_id='my_task_group') as tg1:
    t1 = EmptyOperator(task_id='task_1')
    t2 = EmptyOperator(task_id='task_2')

    t1 >> t2
# Конец определения группы задач

t3 = EmptyOperator(task_id='end')

# Задание зависимостей группы (tg1)
t0 >> tg1 >> t3
```

Группы задач отображаются и в Grid, и в Graph DAG:

![Группа задач в представлении Grid](images/3_0_taskgroups_simple_grid.png)

![Группа задач в представлении Graph](images/3_0_taskgroups_simple_graph.png)

## Параметры группы задач

Группу можно настроить параметрами. Важнее всего **`group_id`** (имя группы) и **`default_args`** (передаются всем задачам в группе). Ниже — примеры с часто используемыми параметрами:

```python
@task_group(
    group_id="task_group_1",
    default_args={"conn_id": "postgres_default"},
    tooltip="This task group is very important!",
    prefix_group_id=True,
    # parent_group=None,
    # dag=None,
)
def tg1():
    t1 = EmptyOperator(task_id="t1")

tg1()
```

```python
with TaskGroup(
    group_id="task_group_2",
    default_args={"conn_id": "postgres_default"},
    tooltip="This task group is also very important!",
    prefix_group_id=True,
    # parent_group=None,
    # dag=None,
    # add_suffix_on_collision=True, # разрешает коллизии group_id добавлением суффикса
) as tg2:
    t1 = EmptyOperator(task_id="t1")
```

## task_id в группах задач

Если задача входит в группу, её фактический **task_id** имеет вид **`group_id.task_id`**. Так сохраняется уникальность task_id в рамках DAG. Этот формат нужно использовать при работе с [XCom](passing-data-between-tasks.md) и [ветвлением](branch-operator.md). Отключить префикс можно параметром группы [**`prefix_group_id=False`**](task-groups.md#task-group-parameters).

Например, у задачи `task_1` в следующем DAG task_id — `my_outer_task_group.my_inner_task_group.task_1`.

```python
@task_group(group_id="my_outer_task_group")
def my_outer_task_group():
    @task_group(group_id="my_inner_task_group")
    def my_inner_task_group():
        EmptyOperator(task_id="task_1")

    my_inner_task_group()

my_outer_task_group()
```

```python
with TaskGroup(group_id="my_outer_task_group") as tg1:
    with TaskGroup(group_id="my_inner_task_group") as tg2:
        EmptyOperator(task_id="task_1")
```

## Передача данных через группы задач

С декоратором **`@task_group`** данные через группу передаются так же, как с обычными декораторами **`@task`**:

```python
from airflow.decorators import dag, task, task_group
from pendulum import datetime
import json


@dag(start_date=datetime(2023, 8, 1), schedule=None, catchup=False)
def task_group_example():
    @task
    def extract_data():
        data_string = '{"1001": 301.27, "1002": 433.21, "1003": 502.22}'
        order_data_dict = json.loads(data_string)
        return order_data_dict

    @task
    def transform_sum(order_data_dict: dict):
        total_order_value = 0
        for value in order_data_dict.values():
            total_order_value += value

        return {"total_order_value": total_order_value}

    @task
    def transform_avg(order_data_dict: dict):
        total_order_value = 0
        for value in order_data_dict.values():
            total_order_value += value
            avg_order_value = total_order_value / len(order_data_dict)

        return {"avg_order_value": avg_order_value}

    @task_group
    def transform_values(order_data_dict):
        return {
            "avg": transform_avg(order_data_dict),
            "total": transform_sum(order_data_dict),
        }

    @task
    def load(order_values: dict):
        print(
            f"""Total order value is: {order_values['total']['total_order_value']:.2f} 
            and average order value is: {order_values['avg']['avg_order_value']:.2f}"""
        )

    load(transform_values(extract_data()))


task_group_example()
```

Получившийся DAG показан на рисунке ниже:

![Передача данных через группу задач](images/3_0_taskgroups_passing_data.png)

При передаче данных в группу и из группы важно:

- Если функция группы возвращает результат, который принимает другая задача на вход, TaskFlow API сам выводит зависимости между группой и задачами. Если выход группы никуда не передаётся, зависимости нужно задать явно операторами `<<` или `>>`.
- Если нижестоящим задачам нужен вывод задач из группы, функция группы должна возвращать результат. В примере выше возвращается словарь с двумя значениями (по одному от каждой задачи группы), он передаётся в задачу `load()`.

## Динамическое создание групп задач в рантайме

[Динамический маппинг задач](dynamic-tasks.md) можно сочетать с декоратором **`@task_group`**: группы создаются динамически по разным входам для заданного параметра. В следующем DAG группа маппится по разным значениям параметра:

```python
from airflow.decorators import dag, task_group, task
from pendulum import datetime


@dag(
    start_date=datetime(2022, 12, 1),
    schedule=None,
    catchup=False,
)
def task_group_mapping_example():
    # группа задач с динамическим входом my_num
    @task_group(group_id="group1")
    def tg1(my_num):
        @task
        def print_num(num):
            return num

        @task
        def add_42(num):
            return num + 42

        print_num(my_num) >> add_42(my_num)

    # нижестоящая задача для вывода XCom
    @task
    def pull_xcom(**context):
        pulled_xcom = context["ti"].xcom_pull(
            # ссылка на задачу в группе: task_group_id.task_id
            task_ids=["group1.add_42"],
            # выбор XCom только из указанных маппленных экземпляров группы (функция 2.5)
            map_indexes=[2, 3],
            key="return_value",
        )

        # выведет список результатов для map index 2 и 3 задачи add_42
        print(pulled_xcom)

    # создание 6 маппленных экземпляров группы group1 (функция 2.5)
    tg1_object = tg1.expand(my_num=[19, 23, 42, 8, 7, 108])

    # задание зависимостей
    tg1_object >> pull_xcom()


task_group_mapping_example()
```

В этом DAG группа `group1` маппится по разным значениям параметра `my_num`. Создаётся 6 экземпляров группы, по одному на каждое значение. В каждом экземпляре две задачи выполняются с соответствующим значением `my_num`. Задача `pull_xcom()` ниже по потоку показывает, как получить нужные значения [XCom](passing-data-between-tasks.md) из списка маппленных экземпляров группы (`map_indexes`).

Подробнее о динамическом маппинге, в том числе по нескольким параметрам: [Динамические задачи](dynamic-tasks.md).

## Порядок групп задач

При создании групп в цикле по умолчанию они выполняются параллельно. Если одна группа зависит от результатов другой, их нужно выполнять последовательно. Например, при загрузке таблиц с внешними ключами записи в основной таблице должны существовать до загрузки зависимых таблиц.

В примере ниже третья группа (третья итерация цикла) имеет внешний ключ к двум предыдущим группам, поэтому её нужно выполнять последней. Для этого создаётся пустой список, в него по мере создания добавляются объекты групп; по этому списку задаются зависимости между группами:

```python
groups = []
for g_id in range(1,4):
    tg_id = f"group{g_id}"

    @task_group(group_id=tg_id)
    def tg1():
        t1 = EmptyOperator(task_id="task1")
        t2 = EmptyOperator(task_id="task2")

        t1 >> t2

        if tg_id == "group1":
            t3 = EmptyOperator(task_id="task3")
            t1 >> t3

    groups.append(tg1())

[groups[0] , groups[1]] >> groups[2]
```

```python
groups = []
for g_id in range(1,4):
    tg_id = f"group{g_id}"
    with TaskGroup(group_id=tg_id) as tg1:
        t1 = EmptyOperator(task_id="task1")
        t2 = EmptyOperator(task_id="task2")

        t1 >> t2

        if tg_id == "group1":
            t3 = EmptyOperator(task_id="task3")
            t1 >> t3

        groups.append(tg1)

[groups[0] , groups[1]] >> groups[2]
```

На следующем рисунке — как эти группы отображаются в Airflow UI:

![Порядок выполнения групп задач](images/3_0_taskgroups_order.png)

В примере также показано, как добавить дополнительную задачу в `group1` в зависимости от `group_id`: даже при создании групп в цикле по одному паттерну можно вносить отличия без дублирования кода.

## Вложенные группы задач

Группы можно вкладывать друг в друга: группа объявляется внутри другой. Уровней вложенности может быть любое количество.

```python
groups = []
for g_id in range(1,3):
    @task_group(group_id=f"group{g_id}")
    def tg1():
        t1 = EmptyOperator(task_id="task1")
        t2 = EmptyOperator(task_id="task2")

        sub_groups = []
        for s_id in range(1,3):
            @task_group(group_id=f"sub_group{s_id}")
            def tg2():
                st1 = EmptyOperator(task_id="task1")
                st2 = EmptyOperator(task_id="task2")

                st1 >> st2
            sub_groups.append(tg2())

        t1 >> sub_groups >> t2
    groups.append(tg1())

groups[0] >> groups[1]
```

```python
groups = []
for g_id in range(1,3):
    with TaskGroup(group_id=f"group{g_id}") as tg1:
        t1 = EmptyOperator(task_id="task1")
        t2 = EmptyOperator(task_id="task2")

        sub_groups = []
        for s_id in range(1,3):
            with TaskGroup(group_id=f"sub_group{s_id}") as tg2:
                st1 = EmptyOperator(task_id="task1")
                st2 = EmptyOperator(task_id="task2")

                st1 >> st2
                sub_groups.append(tg2)

        t1 >> sub_groups >> t2
        groups.append(tg1)

groups[0] >> groups[1]
```

На следующем рисунке — развёрнутый вид вложенных групп в Airflow UI:

![Вложенные группы задач](images/3_0_taskgroups_nested.png)

## Кастомные классы групп задач

Если один и тот же паттерн задач повторяется в нескольких DAG или инстансах Airflow, удобно вынести его в отдельный класс-модуль. Нужно унаследовать **`TaskGroup`** и описать задачи внутри класса. Задачи привязываются к группе через **`self`**. В остальном задачи задаются так же, как в файле DAG.

```python
from airflow.utils.task_group import TaskGroup
from airflow.decorators import task


class MyCustomMathTaskGroup(TaskGroup):
    """Группа задач: сумма двух чисел и умножение результата на 23."""

    # значения по умолчанию для аргументов num1 и num2
    def __init__(self, group_id, num1=0, num2=0, tooltip="Math!", **kwargs):
        """Создание экземпляра MyCustomMathTaskGroup."""
        super().__init__(group_id=group_id, tooltip=tooltip, **kwargs)

        # привязка задачи к группе через self
        @task(task_group=self)
        def task_1(num1, num2):
            """Складывает два числа."""
            return num1 + num2

        @task(task_group=self)
        def task_2(num):
            """Умножает число на 23."""
            return num * 23

        # задание зависимостей
        task_2(task_1(num1, num2))
```

В DAG импортируется класс и создаётся экземпляр с нужными аргументами:

```python
from airflow.decorators import dag, task
from pendulum import datetime
from include.custom_task_group import MyCustomMathTaskGroup


@dag(
    start_date=datetime(2023, 8, 1),
    schedule=None,
    catchup=False,
    tags=["@task_group", "task_group"],
)
def custom_tg():
    @task
    def get_num_1():
        return 5

    tg1 = MyCustomMathTaskGroup(group_id="my_task_group", num1=get_num_1(), num2=19)

    @task
    def downstream_task():
        return "hello"

    tg1 >> downstream_task()


custom_tg()
```

На рисунке ниже — получившаяся кастомная группа, которую можно переиспользовать в других DAG с разными значениями `num1` и `num2`.

![Кастомный класс группы задач](images/3_0_taskgroups_custom_task_group.png)

---

[← Динамические задачи](dynamic-tasks.md) | [К содержанию](README.md) | [Передача данных →](passing-data-between-tasks.md)
