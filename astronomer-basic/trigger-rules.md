# Правила срабатывания (Trigger rules) в Airflow

**Trigger rules** задают, при каких состояниях вышестоящих задач должна запускаться текущая. По умолчанию задача запускается только когда **все** непосредственно вышестоящие задачи успешны (`all_success`). Это переопределяется параметром **trigger_rule** в определении задачи.

Зависимости между задачами задаются отдельно; см. [Управление зависимостями](task-dependencies.md).

## Задание trigger rule

В задаче указывается параметр `trigger_rule`:

```python
@task(trigger_rule="all_success")
def downstream_task():
    return " World!"

# или для оператора
downstream_task = EmptyOperator(
    task_id="downstream_task",
    trigger_rule="all_success"
)
```

## Доступные правила

- **all_success** (по умолчанию) — все вышестоящие успешны.
- **all_failed** — все вышестоящие в состоянии failed или upstream_failed.
- **all_done** — все вышестоящие завершили выполнение (любой исход).
- **all_skipped** — все вышестоящие skipped.
- **all_done_min_one_success** (Airflow 3.1+) — все вышестоящие завершены и хотя бы один успешен.
- **one_failed** — хотя бы один вышестоящий failed.
- **one_success** — хотя бы один вышестоящий успешен.
- **one_done** — хотя бы один вышестоящий успешен или failed.
- **none_failed** — ни одна вышестоящая не failed (успех или skipped).
- **none_failed_min_one_success** — нет failed/upstream_failed и хотя бы один успешен.
- **none_skipped** — ни одна вышестоящая не skipped.
- **always** — задача запускается в любом случае.

Параметр DAG **fail_fast=True** останавливает DAG при первой неудаче и несовместим с правилами, отличными от all_success. [Setup/Teardown задачи](https://www.astronomer.io/docs/learn/airflow-setup-teardown) тоже влияют на срабатывание.

## Ветвление (branching) и trigger rules

При [ветвлении](https://www.astronomer.io/docs/learn/airflow-branch-operator) часть веток пропускается, поэтому нижестоящая «общая» задача может никогда не запуститься с правилом `all_success` (одна из веток всегда skipped). В таких случаях обычно используют **none_failed_min_one_success** или **none_failed**: задача запустится, если хотя бы одна ветка выполнена и ни одна не упала.

Пример: после branch задача `end` с `trigger_rule=TriggerRule.NONE_FAILED_MIN_ONE_SUCCESS` (или ONE_SUCCESS в старых версиях) запустится, когда выполнится одна из веток и ни одна не упадёт.

---

[← Зависимости задач](task-dependencies.md) | [К содержанию](README.md)
