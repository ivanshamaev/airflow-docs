# Параметры (Params) в Airflow

**Params** — параметры уровня DAG или задачи, которые можно задать по умолчанию и переопределять при запуске (UI, API, CLI). В отличие от [Variables](https://www.astronomer.io/docs/learn/airflow-variables), params привязаны к DAG/DAG run и не шифруются — не используйте их для секретов.

## Использование

- На уровне DAG: `params` в конструкторе DAG или в `@dag(params={"my_param": "default"})`. Доступ в задачах: `context["params"]["my_param"]` или в Jinja: `{{ params.my_param }}`.
- На уровне задачи: параметр `params` у оператора. Task-level params переопределяют DAG-level для этой задачи.
- При ручном запуске или через API можно передать словарь params — он объединяется с дефолтными и доступен в контексте как `context["params"]`.

## Рекомендации

- Задавайте осмысленные значения по умолчанию и типы (через Pydantic или проверку в коде), чтобы DAG был предсказуемым.
- Для больших или чувствительных данных используйте [Variables](https://www.astronomer.io/docs/learn/airflow-variables) или Secrets Backend, а не params.

Подробнее: [Airflow params](https://www.astronomer.io/docs/learn/airflow-params), [Templates reference: params](https://airflow.apache.org/docs/apache-airflow/stable/templates-ref.html).

---

[← К содержанию](README.md) | [Контекст →](airflow-context.md) | [Trigger rules →](../astronomer-basic/trigger-rules.md)
