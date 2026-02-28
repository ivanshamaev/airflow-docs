# DAG Factory

[DAG Factory](https://astronomer.github.io/dag-factory/latest/) — open-source инструмент от Astronomer для **генерации DAG из YAML**. Позволяет создавать пайплайны без Python: аналитики и младшие разработчики могут описывать DAG декларативно.

## Когда использовать

- **Доступность для команды** — YAML проще Python для многих ролей.
- **Стандартизация** — десятки одинаковых по структуре DAG (например, extract-load); один шаблон, параметры в YAML.
- **Разделение логики и структуры** — YAML задаёт структуру и зависимости; бизнес-логика в Python-функциях в `include/`.

## Когда не использовать

- Сложная условная логика, branching, тонкая обработка ошибок — лучше нативный Python.
- YAML сложнее отлаживать (нет пошаговой отладки, меньше логов).
- Asset-aware scheduling и новые фичи Airflow могут работать хуже, чем в чистом Python.

## Установка и структура

1. Добавить в `requirements.txt`: `dag-factory==1.0.1`.
2. В `include/tasks/` — Python-функции, вызываемые операторами.
3. В `dags/` — YAML-файлы с описанием DAG и Python-скрипт, который генерирует DAG из YAML (через `DAGFactory`).

## Пример YAML

![DAG из YAML в Airflow UI](images/dag-factory_basic-yaml.png)

```yaml
basic_example_dag:
  default_args:
    owner: "astronomer"
    start_date: 2025-09-01
  schedule: "@hourly"
  task_groups:
    extract:
      tooltip: "data extraction"
  tasks:
    extract_data_from_a:
      decorator: airflow.sdk.task
      python_callable: include.tasks.basic_example_tasks._extract_data
      task_group_name: extract
    store_data:
      decorator: airflow.sdk.task
      python_callable: include.tasks.basic_example_tasks._store_data
      data_a: +extract_data_from_a
      data_b: +extract_data_from_b
```

Поддерживаются TaskFlow API, task groups, передача данных между задачами (`+task_id`). См. [dag-factory](https://github.com/astronomer/dag-factory), [DAG Factory guide](https://www.astronomer.io/docs/learn/dag-factory).

---

[← Документирование](dag-documentation.md) | [К содержанию](README.md) | [PyCharm →](pycharm-local-dev.md)
