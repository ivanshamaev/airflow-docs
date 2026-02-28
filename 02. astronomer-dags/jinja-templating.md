# Jinja-шаблонирование в Airflow

Многие параметры операторов поддерживают **Jinja2**-шаблоны. В момент выполнения задачи шаблон подставляется с учётом [контекста](airflow-context.md) (даты, DAG run, task instance, variables, params и т.д.).

## Шаблонируемые поля

У каждого оператора есть атрибут **template_fields** — список параметров, в которых разрешены шаблоны. По умолчанию это часто `bash_command` у BashOperator, `sql` у SQL-операторов, `python_callable` не шаблонируется (используйте контекст внутри функции). Список по операторам: [Astronomer Registry](https://registry.astronomer.io/); расширение списка: [Templating additional fields](https://www.astronomer.io/docs/learn/templating#templating-additional-fields).

## Часто используемые переменные

- **ds** — логическая дата в формате YYYY-MM-DD.
- **ds_nodash** — то же без дефисов (YYYYMMDD).
- **ts** — полная метка времени (логическая дата + время).
- **ts_nodash** — ts без разделителей.
- **dag**, **task**, **ti** (task_instance), **params**, **var** (variables), **conn** (connections) и др.

Полный список: [Templates reference](https://airflow.apache.org/docs/apache-airflow/stable/templates-ref.html).

## Примеры

В BashOperator:

```python
BashOperator(
    task_id="print_date",
    bash_command="echo {{ ds }}",
)
```

Получение XCom в шаблоне:

```python
bash_command="echo '{{ ti.xcom_pull(task_ids=\"upstream_task\", key=\"return_value\") }}'"
```

Variables и params:

```python
bash_command="echo {{ var.value.my_var }} {{ params.my_param }}"
```

## Рекомендации

- В шаблоны не передавайте недоверенный ввод без санитизации — возможны инъекции.
- Учитывайте, что при asset-triggered или manual run с `logical_date=None` части ключей (ds, ts, data_interval_*) в контексте нет — используйте их в шаблонах только когда они гарантированно заданы.

Подробнее: [Templating](https://www.astronomer.io/docs/learn/templating), [Templates reference](https://airflow.apache.org/docs/apache-airflow/stable/templates-ref.html).

---

[← Контекст](airflow-context.md) | [К содержанию](README.md) | [Передача данных →](passing-data-between-tasks.md)
