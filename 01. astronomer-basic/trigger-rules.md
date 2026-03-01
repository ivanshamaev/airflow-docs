# Правила срабатывания (Trigger rules) в Airflow

**Trigger rules** задают, когда задача должна запускаться относительно вышестоящих. По умолчанию Airflow запускает задачу только когда все непосредственно вышестоящие задачи успешны. Это поведение можно изменить параметром `trigger_rule` в определении задачи.

> **Инфо.** Trigger rules определяют запуск задачи по состоянию её **непосредственных** вышестоящих зависимостей. Как задавать зависимости между задачами: [Управление зависимостями между задачами и группами задач](task-dependencies.md).

## Задание trigger rule

Правило по умолчанию переопределяется параметром `trigger_rule` в определении задачи.

**С декоратором @task:**

```python
from airflow.sdk import chain, task


@task
def upstream_task():
    return "Hello..."


@task(trigger_rule="all_success")
def downstream_task():
    return " World!"


chain(upstream_task(), downstream_task())
```

**С EmptyOperator:**

```python
from airflow.providers.standard.operators.empty import EmptyOperator
from airflow.sdk import chain

upstream_task = EmptyOperator(task_id="upstream_task")
downstream_task = EmptyOperator(
    task_id="downstream_task",
    trigger_rule="all_success",
)
chain(upstream_task, downstream_task)
```

## Доступные trigger rules в Airflow

Доступны следующие правила:

- **always**: задача запускается в любой момент.
- **none_skipped**: задача запускается только если ни одна вышестоящая не в состоянии `skipped`.
- **none_failed_min_one_success**: задача запускается только если все вышестоящие не в состоянии `failed` или `upstream_failed` и хотя бы одна вышестоящая успешна.
- **none_failed**: задача запускается только если все вышестоящие успешны или пропущены (skipped).
- **one_done**: задача запускается, когда хотя бы одна вышестоящая успешна или завершилась с ошибкой.
- **one_success**: задача запускается, когда хотя бы одна вышестоящая успешна.
- **one_failed**: задача запускается, когда хотя бы одна вышестоящая завершилась с ошибкой.
- **all_done_min_one_success** (Airflow 3.1+): задача запускается, когда все вышестоящие завершили выполнение и хотя бы одна успешна.
- **all_skipped**: задача запускается только когда все вышестоящие пропущены.
- **all_done**: задача запускается, когда все вышестоящие завершили выполнение.
- **all_failed**: задача запускается только когда все вышестоящие в состоянии `failed` или `upstream_failed`.
- **all_success** (по умолчанию): задача запускается только когда все вышестоящие успешны.

> **Инфо.** [Setup and Teardown задачи](../04.%20astronomer-advanced/setup-teardown.md) — особый тип задач для создания и удаления ресурсов; они тоже влияют на срабатывание. На поведение trigger rules влияют и другие возможности Airflow: например, параметр DAG [fail_fast](../02.%20astronomer-dags/dag-parameters.md) при значении `True` останавливает выполнение DAG при первой неудаче, переводит все ещё выполняющиеся задачи в `failed`, а ещё не запущенные — в `skipped`. В DAG с `fail_fast=True` нельзя использовать правило, отличное от `all_success`.

## Ветвление и trigger rules

Trigger rules часто нужны в DAG с условной логикой, например при [ветвлении](../02.%20astronomer-dags/branch-operator.md). В таких случаях полезнее бывают `none_failed_min_one_success` или `none_failed`, а не `all_success`: при ветвлении выполняется только одна ветка, остальные задачи остаются в `skipped`, и при правиле «все успешны» общая нижестоящая задача никогда не запустится.

В примере ниже — простое ветвление и нижестоящая задача `end`, которая должна запускаться при выполнении любой из веток. При правиле `all_success` задача `end` не запустится, потому что все ветки, кроме одной, пропускаются и не имеют состояния success. Если задать правило `none_failed_min_one_success`, задача `end` запустится, когда хотя бы одна ветка успешна и ни одна не завершилась с ошибкой.

**Вариант с декоратором @task.branch:**

```python
import random
from airflow.decorators import dag, task
from airflow.operators.empty import EmptyOperator
from datetime import datetime
from airflow.utils.trigger_rule import TriggerRule


@dag(start_date=datetime(2021, 1, 1), max_active_runs=1, schedule=None, catchup=False)
def branching_dag():
    start = EmptyOperator(task_id="start")
    end = EmptyOperator(task_id="end", trigger_rule=TriggerRule.ONE_SUCCESS)

    @task.branch
    def branching(**kwargs):
        branches = ["branch_0", "branch_1", "branch_2"]
        return random.choice(branches)

    branching_task = branching()

    start >> branching_task

    for i in range(0, 3):
        d = EmptyOperator(task_id="branch_{0}".format(i))
        branching_task >> d >> end


branching_dag()
```

**Вариант с BranchPythonOperator:**

```python
import random
from airflow import DAG
from airflow.operators.empty import EmptyOperator
from airflow.operators.python import BranchPythonOperator
from datetime import datetime
from airflow.utils.trigger_rule import TriggerRule


def return_branch(**kwargs):
    branches = ["branch_0", "branch_1", "branch_2"]
    return random.choice(branches)


with DAG(
    dag_id="branching_dag",
    start_date=datetime(2021, 1, 1),
    max_active_runs=1,
    schedule=None,
    catchup=False,
):
    start = EmptyOperator(task_id="start")
    end = EmptyOperator(task_id="end", trigger_rule=TriggerRule.ONE_SUCCESS)

    branching = BranchPythonOperator(
        task_id="branching", python_callable=return_branch, provide_context=True
    )

    start >> branching

    for i in range(0, 3):
        d = EmptyOperator(task_id="branch_{0}".format(i))
        branching >> d >> end
```

В обоих вариантах задача `end` с `trigger_rule=TriggerRule.ONE_SUCCESS` запускается, как только успешно завершилась одна из веток (branch_0, branch_1 или branch_2).

---

[← Зависимости задач](task-dependencies.md) | [К содержанию](README.md) | [Переменные →](variables.md)
