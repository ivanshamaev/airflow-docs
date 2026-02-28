# Контекст Airflow (Airflow context)

**Контекст Airflow** — словарь с информацией о выполняющемся DAG и окружении, доступный внутри задачи. Чаще всего используется ключ **ti** / **task_instance** — объект [TaskInstance](https://airflow.apache.org/docs/apache-airflow/stable/_api/airflow/models/taskinstance/index.html), через который доступны `.xcom_pull()` и `.xcom_push()`.

Типичные причины обращаться к контексту:

- Явная отправка и получение значений в [XCom](passing-data-between-tasks.md) с произвольным ключом.
- Использование [логической даты](https://www.astronomer.io/docs/learn/scheduling-in-airflow) DAG run в задаче (например, в имени файла).
- Использование [параметров DAG/задачи](airflow-params.md) (params) в коде задачи.

## Доступ к контексту

Контекст доступен во всех задачах. Способы доступа:

- В методе `.execute` традиционного или кастомного оператора — аргумент **context**.
- В [Jinja-шаблонах](jinja-templating.md) в шаблонируемых полях операторов.
- В функции с декоратором `@asset` — аргумент **context** (см. [Ассеты](https://www.astronomer.io/docs/learn/airflow-datasets)).
- В функции с декоратором **@task** или в **PythonOperator** — аргумент **\*\*context** в сигнатуре функции.

Вне задачи контекст недоступен.

### @task и PythonOperator

Добавьте в функцию аргумент **\*\*context** — контекст будет передан как словарь:

```python
@task
def print_context(**context):
    from pprint import pprint
    pprint(context)
```

### Jinja-шаблоны

Многие поля операторов поддерживают шаблоны. Список полей — атрибут `.template_fields` оператора. Пример логической даты в формате YYYY-MM-DD в BashOperator:

```python
print_logical_date = BashOperator(
    task_id="print_logical_date",
    bash_command="echo {{ ds }}",
)
```

Получение XCom в шаблоне: `{{ ti.xcom_pull(task_ids='return_greeting') }}`. Актуальный список шаблонов: [Templates reference](https://airflow.apache.org/docs/apache-airflow/stable/templates-ref.html).

### Кастомный оператор

В кастомном операторе контекст передаётся в `.execute(context)`:

```python
def execute(self, context):
    print(context["dag"].dag_id)
```

## Основные ключи контекста

- **ti / task_instance** — объект TaskInstance: `.xcom_pull()`, `.xcom_push()` и другие атрибуты.
- **Ключи планирования** — `ts`, `ds`, `ds_nodash`, `data_interval_start`, `data_interval_end` и др. (см. [Templates reference](https://airflow.apache.org/docs/apache-airflow/stable/templates-ref.html)). Используются, например, для имён файлов: `context["ts"]` в формате `YYYY-MM-DDTHH:MM:SS+00:00`.
- **dag_run** — объект DAG run; полезен атрибут `run_type` (как был запущен DAG: scheduled, manual, dataset triggered и т.д.).
- **params** — словарь DAG- и task-level параметров для данного экземпляра задачи.
- **var** — доступ к [переменным Airflow](https://www.astronomer.io/docs/learn/airflow-variables): `context["var"]["value"]`, `context["var"]["json"]`.

Полный список ключей: [airflow/utils/context.py](https://github.com/apache/airflow/blob/main/airflow-core/src/airflow/utils/context.py).

## Метки времени и тип запуска

При запуске по **ассету** или при **ручном/API** запуске с явным `logical_date=None` в контексте **отсутствуют** ключи: `logical_date`, `ds`, `ds_nodash`, `ts`, `ts_nodash`, `data_interval_start`, `data_interval_end`, `previous_data_interval_start_success`, `previous_data_interval_end_success`. Обращение к ним вызовет `KeyError`. Проверяйте наличие ключа или тип запуска перед использованием.

---

[← К содержанию](README.md) | [Декораторы →](airflow-decorators.md) | [XCom →](passing-data-between-tasks.md)
