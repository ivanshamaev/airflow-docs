# Управление зависимостями между задачами

[Зависимости](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/tasks.html#relationships) — одна из ключевых возможностей Airflow. Пайплайны задаются в виде ориентированных ациклических графов (DAG): каждая задача — узел, зависимости — направленные рёбра, по которым определяется порядок обхода графа. Поэтому зависимости важны для гибких пайплайнов с атомарными задачами.

В руководстве используются термины:

- **Downstream task** (нижестоящая задача): зависимая задача, которая не может запуститься, пока вышестоящая не перейдёт в заданное состояние.
- **Upstream task** (вышестоящая задача): задача, которая должна достичь заданного состояния, прежде чем сможет выполниться зависимая от неё задача.

Вы узнаете, как задавать зависимости в Airflow разными способами:

- Trigger rules (правила срабатывания).
- Зависимости в TaskFlow API.
- Зависимости с task groups.
- Динамические зависимости.
- Функции зависимостей.
- Базовые зависимости между задачами.

Видео по теме: [Manage Dependencies Between Airflow Deployments, DAGs, and Tasks](https://www.astronomer.io/events/webinars/manage-dependencies-between-airflow-deployments-dags-tasks/).

Здесь рассматриваются зависимости между задачами **внутри одного DAG**. Зависимости между разными DAG: [Cross-DAG dependencies](https://www.astronomer.io/docs/learn/cross-dag-dependencies).

## Необходимая база

Нужно понимать основы Airflow. См. [Введение в Apache Airflow](https://www.astronomer.io/docs/learn/intro-to-airflow).

## Базовые зависимости

Базовые зависимости между задачами можно задать так:

- методами `set_upstream` и `set_downstream`;
- операторами сдвига `<<` и `>>`.

Например, для DAG из четырёх последовательных задач зависимости можно задать четырьмя способами:

- Через `<<`: `t3 << t2 << t1 << t0`
- Через `>>`: `t0 >> t1 >> t2 >> t3`
- Через `set_upstream()`:
  ```text
  t3.set_upstream(t2)
  t2.set_upstream(t1)
  t1.set_upstream(t0)
  ```
- Через `set_downstream()`:
  ```text
  t0.set_downstream(t1)
  t1.set_downstream(t2)
  t2.set_downstream(t3)
  ```

Все варианты эквивалентны и дают один и тот же граф (цепочка t0 → t1 → t2 → t3).

Рекомендуется придерживаться одного способа. Смешение операторов сдвига и `set_upstream`/`set_downstream` усложняет код.

Чтобы две нижестоящие задачи зависели от одной вышестоящей, используйте списки или кортежи:

```python
# Зависимости со списками
t0 >> t1 >> [t2, t3]

# Зависимости с кортежами
t0 >> t1 >> (t2, t3)
```

Оба варианта эквивалентны: t0 → t1 → t2 и t0 → t1 → t3.

Операторами сдвига и методами `.set_upstream`/`.set_downstream` нельзя задать зависимости между двумя списками: выражение `[t0, t1] >> [t2, t3]` вызовет ошибку. Для зависимостей между списками используйте функции зависимостей (см. следующий раздел).

## Функции зависимостей

Функции зависимостей позволяют задавать зависимости между несколькими задачами или списками задач. Их часто используют для задач, созданных в цикле и собранных в список.

```python
from airflow.sdk import chain

list_of_tasks = []
for i in range(5):
    if i % 3 == 0:
        ta = EmptyOperator(task_id=f"ta_{i}")
        list_of_tasks.append(ta)
    else:
        ta = EmptyOperator(task_id=f"ta_{i}")
        tb = EmptyOperator(task_id=f"tb_{i}")
        tc = EmptyOperator(task_id=f"tc_{i}")
        list_of_tasks.append([ta, tb, tc])

chain(*list_of_tasks)
```

Так строится цепочка задач с ветвлениями.

### Функция `chain()`

Чтобы задать параллельные зависимости между задачами и списками задач **одинаковой длины**, используйте функцию `chain()`:

```python
from airflow.sdk import chain

chain(t0, t1, [t2, t3, t4], [t5, t6, t7], t8)
```

В результате: t0 → t1 → (параллельно t2, t3, t4) → (параллельно t5, t6, t7) → t8.

При использовании `chain()` любые списки или кортежи, зависящие друг от друга напрямую, должны иметь одинаковую длину.

```python
chain([t0, t1], [t2, t3])       # корректно
chain([t0, t1], [t2, t3, t4])   # ошибка — разная длина
chain([t0, t1], t2, [t3, t4, t5])  # корректно
```

### Функция `chain_linear()`

Чтобы задать перекрёстные зависимости (каждый элемент нижестоящего списка зависит от каждого элемента вышестоящего), используйте `chain_linear()`.

Если в предыдущем примере заменить `chain` на `chain_linear`, получится: каждый из t2, t3, t4 зависит от t0 и t1; каждый из t5, t6, t7 — от t2, t3, t4 и т.д.

```python
from airflow.sdk import chain_linear

chain_linear(t0, t1, [t2, t3, t4], [t5, t6, t7], t8)
```

`chain_linear()` допускает списки любой длины в любом порядке:

```python
chain_linear([t0, t1], [t2, t3, t4])
```

## Зависимости при динамическом маппинге задач

Зависимости для [динамически маппленных задач](https://www.astronomer.io/docs/learn/dynamic-tasks) задаются так же, как для обычных. При [trigger rule](trigger-rules.md) по умолчанию `all_success` все экземпляры маппленной задачи должны завершиться успешно, чтобы запустилась нижестоящая задача (с точки зрения trigger rules они ведут себя как набор параллельных вышестоящих задач).

**Вариант с TaskFlow API:**

```python
from airflow.providers.standard.operators.empty import EmptyOperator
from airflow.sdk import task

start = EmptyOperator(task_id="start")


@task
def multiply(x, y):
    return x * y


multiply_obj = multiply.partial(x=2).expand(y=[1, 2, 3])

# end запустится только если все экземпляры задачи multiply успешно завершились
end = EmptyOperator(task_id="end")

start >> multiply_obj >> end

# Допустимы и такие варианты:
# multiply_obj.set_downstream(end)
# end.set_upstream(multiply_obj)
# chain(start, multiply_obj, end)
```

**Вариант с PythonOperator:**

```python
from airflow.providers.standard.operators.empty import EmptyOperator
from airflow.providers.standard.operators.python import PythonOperator

start = EmptyOperator(task_id="start")


def multiply_func(x, y):
    return x * y


multiply_obj = PythonOperator.partial(
    task_id="multiply",
    python_callable=multiply_func,
    op_args=[2],
).expand(op_kwargs=[{"y": 1}, {"y": 2}, {"y": 3}])

# end запустится только если все экземпляры задачи multiply успешно завершились
end = EmptyOperator(task_id="end")

start >> multiply_obj >> end

# Допустимы и такие варианты:
# multiply_obj.set_downstream(end)
# end.set_upstream(multiply_obj)
# chain(start, multiply_obj, end)
```

## Зависимости с task groups

[Task groups](https://www.astronomer.io/docs/learn/task-groups) логически группируют задачи в UI и могут [создаваться динамически](https://www.astronomer.io/docs/learn/task-groups#generate-task-groups-dynamically-at-runtime). Ниже — как задавать зависимости между группами и внутри них.

Зависимости задаются и внутри группы, и снаружи. В примере: стартовая задача, группа из двух зависимых задач и конечная задача идут последовательно. Зависимость между двумя задачами внутри группы задаётся в контексте группы (`t1 >> t2`). Зависимости группы со стартом и концом — в контексте DAG (`t0 >> tg1() >> t3`).

**Вариант с декоратором @task_group:**

```python
from airflow.providers.standard.operators.empty import EmptyOperator
from airflow.sdk import task_group

t0 = EmptyOperator(task_id="start")


@task_group(group_id="group1")
def tg1():
    t1 = EmptyOperator(task_id="task1")
    t2 = EmptyOperator(task_id="task2")
    t1 >> t2


t3 = EmptyOperator(task_id="end")

# Зависимости группы tg1
t0 >> tg1() >> t3
```

**Вариант с контекстным менеджером TaskGroup:**

```python
from airflow.providers.standard.operators.empty import EmptyOperator
from airflow.sdk import TaskGroup

t0 = EmptyOperator(task_id="start")

with TaskGroup(group_id="group1") as tg1:
    t1 = EmptyOperator(task_id="task1")
    t2 = EmptyOperator(task_id="task2")
    t1 >> t2

t3 = EmptyOperator(task_id="end")

# Зависимости группы tg1
t0 >> tg1 >> t3
```

Можно задавать зависимости между группами, между задачами внутри и снаружи групп, а также между задачами из разных (вложенных) групп.

Примеры более сложных зависимостей с task groups: [TaskFlow API](https://github.com/astronomer/webinar-task-groups/blob/main/dags/example_complex_dependencies_1.py) и [традиционный вариант](https://github.com/astronomer/webinar-task-groups/blob/main/dags/example_complex_dependencies_2.py) в репозитории Astronomer.

## Зависимости в TaskFlow API

Декоратор [TaskFlow API](https://www.astronomer.io/docs/learn/airflow-decorators) `@task` превращает Python-функции в задачи Airflow.

Если в DAG несколько задач с `@task` используют выход друг друга, зависимости можно задать неявно: передать вызов вышестоящей задачи аргументом в нижестоящую. В примере две задачи — `get_a_cat_fact` и `print_the_cat_fact`. Зависимость задаётся так: `print_the_cat_fact(get_a_cat_fact())`.

```python
from airflow.decorators import dag, task
from datetime import datetime

import requests
import json

url = "http://catfact.ninja/fact"
default_args = {"start_date": datetime(2021, 1, 1)}


@dag(schedule="@daily", default_args=default_args, catchup=False)
def xcom_taskflow_dag():
    @task
    def get_a_cat_fact():
        """Получает факт о котах из CatFacts API."""
        res = requests.get(url)
        return {"cat_fact": json.loads(res.text)["fact"]}

    @task
    def print_the_cat_fact(cat_fact: str):
        """Печатает факт о коте."""
        print("Cat fact for today:", cat_fact)

    # Вызов функций создаёт задачи и задаёт зависимости
    print_the_cat_fact(get_a_cat_fact())


xcom_taskflow_dag()
```

Можно сохранить вызов вышестоящей задачи в переменную и передать её в нижестоящую. Так код читается проще, и одну задачу легко сделать вышестоящей для нескольких других:

```python
from airflow.sdk import task


@task
def get_num():
    return 42


@task
def add_one(num):
    return num + 1


@task
def add_two(num):
    return num + 2


num = get_num()
add_one(num)
add_two(num)
```

Если в DAG есть и задачи с декораторами, и задачи с обычными операторами, сохраните вызов декорированной задачи в переменную и задайте зависимости как обычно. В примере ниже `my_task_1` объявлена через `@task` и вызывается как `_my_task_1 = my_task_1()`; переменная `_my_task_1` используется в цепочке зависимостей.

```python
from airflow.sdk import dag, task
from airflow.providers.standard.operators.empty import EmptyOperator
from airflow.sdk import chain


@dag
def my_dag():
    @task
    def my_task_1():
        pass

    _my_task_1 = my_task_1()
    my_task_2 = EmptyOperator(task_id="my_task_2")

    chain(_my_task_1, my_task_2)


my_dag()
```

Передачу данных между декораторами TaskFlow и традиционными операторами см. в [Mixing TaskFlow decorators with traditional operators](https://www.astronomer.io/docs/learn/airflow-decorators#mixing-taskflow-decorators-with-traditional-operators).

## Trigger rules (правила срабатывания)

При заданных зависимостях по умолчанию задача запускается только когда **все** вышестоящие задачи завершились успешно. Поведение можно изменить с помощью [trigger rules](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/dags.html#trigger-rules).

Доступные варианты:

- **always**: задача запускается в любой момент.
- **none_skipped**: задача запускается только если ни одна вышестоящая не в состоянии skipped.
- **none_failed_min_one_success**: задача запускается только если все вышестоящие не в failed/upstream_failed и хотя бы одна вышестоящая успешна.
- **none_failed**: задача запускается только если все вышестоящие успешны или пропущены (skipped).
- **one_done**: задача запускается, когда хотя бы одна вышестоящая успешна или завершилась с ошибкой.
- **one_success**: задача запускается, когда хотя бы одна вышестоящая успешна.
- **one_failed**: задача запускается, когда хотя бы одна вышестоящая завершилась с ошибкой.
- **all_skipped**: задача запускается только когда все вышестоящие пропущены.
- **all_done**: задача запускается, когда все вышестоящие завершили выполнение.
- **all_failed**: задача запускается только когда все вышестоящие в состоянии failed или upstream_failed.
- **all_success** (по умолчанию): задача запускается только когда все вышестоящие успешны.

Подробнее: [Trigger rules](trigger-rules.md).

---

[← Сенсоры](sensors.md) | [К содержанию](README.md) | [Trigger rules →](trigger-rules.md)
