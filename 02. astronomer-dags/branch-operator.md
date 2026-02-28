# BranchOperator и ветвление в Airflow

**Ветвление (branching)** — выбор одной из нескольких веток выполнения DAG в зависимости от условия. Задача-«ветка» возвращает `task_id` следующей задачи, которую нужно выполнить; остальные ветки помечаются как **skipped**.

## Способы реализации

- **@task.branch** — декоратор для функции, возвращающей `task_id` (или список task_id в части версий). Удобно для сложной логики.
- **BranchPythonOperator** — традиционный оператор с `python_callable`, возвращающей `task_id`.
- **BranchDateTimeOperator**, **BranchTimestampOperator** — ветвление по времени.
- **@task.short_circuit** — условие возвращает True/False; при False все нижестоящие задачи пропускаются (short circuit).
- **Other branch operators:** `@task.branch_virtualenv`, `@task.branch_external_python` — ветвление в отдельном виртуальном окружении.

## Важно для нижестоящих задач

После ветвления только одна ветка выполняется, остальные — skipped. Задача, объединяющая ветки (например, `end`), по умолчанию ждёт **all_success** и не запустится, так как в пропущенных ветках нет success. Нужно задать **trigger_rule**: `none_failed_min_one_success` или `none_failed`, чтобы задача запустилась после одной успешной ветки. См. [Trigger rules](../01. astronomer-basic/trigger-rules.md).

## Пример (@task.branch)

```python
@task.branch
def branching(**kwargs):
    branches = ["branch_0", "branch_1", "branch_2"]
    return random.choice(branches)

branching_task = branching()
start >> branching_task
for i in range(0, 3):
    d = EmptyOperator(task_id=f"branch_{i}")
    branching_task >> d >> end  # end с trigger_rule="none_failed_min_one_success"
```

Подробнее: [BranchOperator](https://www.astronomer.io/docs/learn/airflow-branch-operator), [Trigger rules](../01. astronomer-basic/trigger-rules.md).

---

[← К содержанию](README.md) | [Зависимости задач →](../01. astronomer-basic/task-dependencies.md)
