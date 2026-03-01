# Ветвление в Airflow (@task.branch и BranchPythonOperator)

При проектировании пайплайнов бывают сценарии сложнее цепочки «Задача A → Задача B → Задача C». Например, нужно выбрать одну из нескольких задач по результату вышестоящей или выполнять часть пайплайна только при определённых внешних условиях. В Airflow для этого есть несколько механизмов условной логики и ветвления.

В этом руководстве: использование `@task.branch` (BranchPythonOperator) и `@task.short_circuit` (ShortCircuitOperator), другие операторы ветвления и дополнительные материалы.

## Необходимая база

Полезно понимать:

- Декораторы Airflow. См. [TaskFlow API и декораторы](airflow-decorators.md).
- Зависимости в Airflow. См. [Управление зависимостями](../01.%20astronomer-basic/task-dependencies.md).
- Операторы Airflow. См. [Что такое оператор?](../01.%20astronomer-basic/operators.md).

## @task.branch (BranchPythonOperator)

Один из простых способов задать ветвление — декоратор `@task.branch`, вариант [BranchPythonOperator](https://registry.astronomer.io/providers/apache-airflow/modules/branchpythonoperator). В `@task.branch` передаётся Python-функция, которая возвращает список допустимых **task_id** задач, которые DAG должен выполнить после этой функции.

В примере ниже функция `choose_branch` возвращает один набор task_id, если результат больше 0.5, и другой — если меньше или равен 0.5:

```python
from airflow.sdk import task

result = 1


@task.branch
def choose_branch(result):
    if result > 0.5:
        return ["task_a", "task_b"]
    return ["task_c"]


choose_branch(result)
```

```python
from airflow.providers.standard.operators.python import BranchPythonOperator

result = 1


def choose_branch(result):
    if result > 0.5:
        return ["task_a", "task_b"]
    return ["task_c"]


branching = BranchPythonOperator(
    task_id="branching",
    python_callable=choose_branch,
    op_args=[result],
)
```

В общем случае `@task.branch` удобен, когда логику ветвления легко описать одной Python-функцией. Выбор между декоратором и традиционным оператором — вопрос стиля.

Полный пример DAG с `@task.branch`:

```python
"""Пример DAG с декоратором @task.branch (TaskFlow API)."""

from airflow.sdk import dag, Label, task
from airflow.providers.standard.operators.empty import EmptyOperator

import random


@dag
def branch_python_operator_decorator_example():
    run_this_first = EmptyOperator(task_id="run_this_first")
    options = ["branch_a", "branch_b", "branch_c", "branch_d"]

    @task.branch(task_id="branching")
    def random_choice(choices):
        return random.choice(choices)

    random_choice_instance = random_choice(choices=options)
    run_this_first >> random_choice_instance

    join = EmptyOperator(
        task_id="join",
        trigger_rule="none_failed_min_one_success",
    )

    for option in options:
        t = EmptyOperator(task_id=option)
        empty_follow = EmptyOperator(task_id="follow_" + option)
        # Label необязателен, но помогает в сложных ветвлениях
        random_choice_instance >> Label(option) >> t >> empty_follow >> join


branch_python_operator_decorator_example()
```

```python
"""Пример DAG с BranchPythonOperator."""

from airflow.sdk import DAG, Label
from airflow.providers.standard.operators.empty import EmptyOperator
from airflow.providers.standard.operators.python import BranchPythonOperator
import random


with DAG(dag_id="branch_python_operator_example") as dag:
    run_this_first = EmptyOperator(task_id="run_this_first")
    options = ["branch_a", "branch_b", "branch_c", "branch_d"]

    branching = BranchPythonOperator(
        task_id="branching",
        python_callable=lambda: random.choice(options),
    )
    run_this_first >> branching

    join = EmptyOperator(
        task_id="join",
        trigger_rule="none_failed_min_one_success",
    )

    for option in options:
        t = EmptyOperator(task_id=option)
        empty_follow = EmptyOperator(task_id="follow_" + option)
        # Label необязателен, но помогает в сложных ветвлениях
        branching >> Label(option) >> t >> empty_follow >> join
```

В этом DAG `random.choice()` выбирает одну из четырёх веток. На скриншоте ниже выбрана ветка `branch_b` — две задачи в ней выполнены, остальные пропущены.

![Граф DAG с ветвлением в UI](https://files.buildwithfern.com/astronomer.docs.buildwithfern.com/docs/304bad39761a8cf62a0b6b24f452eb024046af5de74ec9d56f63bbff750ef584/docs/assets/img/guides/3-0_airflow_task_group.png)

Если после ветвления есть задача, которая должна выполняться при любой выбранной ветке (как `join` в примере), нужно изменить [trigger rule](../01.%20astronomer-basic/trigger-rules.md). По умолчанию в Airflow используется `all_success` — при пропуске вышестоящих задач нижестоящая не запустится. В примере для `join` задано правило `none_failed_min_one_success`: задача выполняется, если хотя бы одна вышестоящая успешна и ни одна не упала.

В качестве непосредственной нижестоящей ветки можно указать [task group](task-groups.md): функция ветвления возвращает `task_group_id` вместо `task_id`. Тогда выполняются все корневые задачи этой группы.

```python
from airflow.sdk import dag, task, task_group, chain
from pendulum import datetime


@dag(
    dag_display_name="Task Group Branching",
    start_date=datetime(2024, 8, 1),
    schedule=None,
    catchup=False,
    tags=["Branching"],
)
def task_group_branching():
    @task.branch
    def upstream_task():
        return "my_task_group"

    @task_group
    def my_task_group():
        @task
        def t1():
            return "hi"

        @task
        def t2():
            return "hi"

        t1()
        t2()

    @task
    def outside_task():
        return "hi"

    chain(upstream_task(), [my_task_group(), outside_task()])


task_group_branching()
```

Важно: функция с декоратором `@task.branch` **должна** возвращать хотя бы один task_id для выбранной ветки (не может ничего не вернуть). Если по одной из веток ничего выполнять не нужно, в этой ветке можно использовать EmptyOperator.

## @task.short_circuit (ShortCircuitOperator)

Ещё один вариант условной логики — декоратор `@task.short_circuit`, вариант [ShortCircuitOperator](https://registry.astronomer.io/providers/apache-airflow/modules/shortcircuitoperator). Он принимает Python-функцию, возвращающую `True` или `False`. При `True` выполнение DAG продолжается, при `False` все нижестоящие задачи пропускаются.

`@task.short_circuit` удобен, когда часть задач должна запускаться не всегда. Например, DAG запускается ежедневно, но некоторые задачи — только по воскресеньям. Или DAG оркестрирует ML-модель, и задачи публикации модели должны выполняться только при достижении нужной точности после обучения. То же можно реализовать через `@task.branch`, но тогда нужно возвращать task_id. Для логики «выполнять или не выполнять» (а не «это или то») `@task.short_circuit` часто проще.

Пример DAG с `@task.short_circuit`:

**TaskFlow:**

```python
"""Пример DAG с декоратором @task.short_circuit."""

from airflow.sdk import dag, task, chain
from airflow.providers.standard.operators.empty import EmptyOperator


@dag
def short_circuit_operator_decorator_example():
    @task.short_circuit
    def condition_is_True():
        return True

    @task.short_circuit
    def condition_is_False():
        return False

    ds_true = [EmptyOperator(task_id="true_" + str(i)) for i in [1, 2]]
    ds_false = [EmptyOperator(task_id="false_" + str(i)) for i in [1, 2]]

    chain(condition_is_True(), *ds_true)
    chain(condition_is_False(), *ds_false)


short_circuit_operator_decorator_example()
```

**Традиционный вариант:**

```python
"""Пример DAG с ShortCircuitOperator."""

from airflow.sdk import DAG, chain
from airflow.providers.standard.operators.empty import EmptyOperator
from airflow.providers.standard.operators.python import ShortCircuitOperator


with DAG(dag_id="short_circuit_operator_example") as dag:
    cond_true = ShortCircuitOperator(
        task_id="condition_is_True",
        python_callable=lambda: True,
    )
    cond_false = ShortCircuitOperator(
        task_id="condition_is_False",
        python_callable=lambda: False,
    )

    ds_true = [EmptyOperator(task_id="true_" + str(i)) for i in [1, 2]]
    ds_false = [EmptyOperator(task_id="false_" + str(i)) for i in [1, 2]]

    chain(cond_true, *ds_true)
    chain(cond_false, *ds_false)
```

В этом DAG два short circuit: один всегда возвращает `True`, другой — `False`. При запуске задачи ниже условия `True` выполняются, ниже условия `False` — пропускаются.

## Другие операторы ветвления

В Airflow есть и другие операторы ветвления, по идее похожие на BranchPythonOperator, но заточенные под конкретные сценарии:

- [**BranchPythonVirtualenvOperator**](https://registry.astronomer.io/providers/apache-airflow/versions/latest/modules/BranchPythonVirtualenvOperator): ветвление по Python-функции как у BranchPythonOperator (раздел выше), но выполнение в новом виртуальном окружении (как у [PythonVirtualenvOperator](https://registry.astronomer.io/providers/apache-airflow/versions/latest/modules/PythonVirtualenvOperator)). Окружение можно кэшировать через `venv_cache_path`.
- [**BranchExternalPythonOperator**](https://registry.astronomer.io/providers/apache-airflow/versions/latest/modules/BranchExternalPythonOperator): ветвление по Python-функции, выполнение в уже существующем виртуальном окружении (как у [ExternalPythonOperator](../04.%20astronomer-advanced/isolated-environments.md)).
- [**BranchDateTimeOperator**](https://registry.astronomer.io/providers/apache-airflow/versions/latest/modules/branchdatetimeoperator): ветвление в зависимости от того, попадает ли текущее время в интервал между `target_lower` и `target_upper`.
- [**BranchDayOfWeekOperator**](https://registry.astronomer.io/providers/apache-airflow/versions/latest/modules/branchdayofweekoperator): ветвление по дню недели (параметр `week_day`).
- [**BranchSQLOperator**](https://registry.astronomer.io/providers/apache-airflow-providers-common-sql/versions/latest/modules/branchsqloperator): ветвление по результату SQL-запроса (`true` или `false`).

У этих операторов есть параметры `follow_task_ids_if_true` и `follow_task_ids_if_false` — списки задач для веток «условие истинно» и «условие ложно».

---

[← К содержанию](README.md) | [Trigger rules →](../01.%20astronomer-basic/trigger-rules.md) | [Task groups →](task-groups.md)
