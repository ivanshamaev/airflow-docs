# Выполнение SQL в Airflow

Выполнение SQL — один из типичных сценариев в пайплайнах: загрузка данных, вызов хранимых процедур, отчёты. Ниже — практики и операторы для выполнения SQL из DAG.

## Рекомендации

- **Хуки и операторы.** Универсальный вариант — [Common SQL provider](https://registry.astronomer.io/providers/apache-airflow-providers-common-sql/versions/latest) и [SQLExecuteQueryOperator](https://registry.astronomer.io/providers/apache-airflow-providers-common-sql/modules/SQLExecuteQueryOperator). Для конкретных БД есть специализированные операторы (например, SnowflakeSqlApiOperator). Хуки для БД обычно в пакете провайдера (например, PostgresHook в Postgres provider).
- **Длинный SQL — в отдельный файл.** Не держите большие запросы в DAG. Храните их в каталоге `include/` (при использовании Astro CLI — `include/query1.sql`, `include/query2.sql`). Исключение — короткие однострочные запросы.

## Пример 1: выполнение запроса

Установите провайдеры (например, Snowflake и Common SQL), задайте `template_searchpath` в DAG и передайте в оператор путь к `.sql` файлу:

```python
from airflow.sdk import chain, dag
from airflow.providers.common.sql.operators.sql import SQLExecuteQueryOperator

@dag(template_searchpath="/usr/local/airflow/include")
def execute_snowflake_queries():
    run_query1 = SQLExecuteQueryOperator(
        task_id="run_query1", conn_id="my_snowflake_conn", sql="query1.sql"
    )
    run_query2 = SQLExecuteQueryOperator(
        task_id="run_query2", conn_id="my_snowflake_conn", sql="query2.sql"
    )
    chain(run_query1, run_query2)

execute_snowflake_queries()
```

В `include/` положите `query1.sql` и `query2.sql`. Подключение к Snowflake с `conn_id="my_snowflake_conn"` настраивается в [Connections](connections.md).

## Пример 2: параметризованный запрос

Можно использовать контекст Airflow (например, логическую дату) в SQL через шаблоны и макросы или передавать параметры через **parameters** в операторе (значения из вышестоящих задач):

```python
opr_param_query = SQLExecuteQueryOperator(
    task_id="param_query",
    conn_id="my_snowflake_conn",
    sql="""
        SELECT * FROM DEMO_DB.DEMO_SCHEMA.MY_NEW_TABLE
        WHERE grocery = %(my_grocery)s;
    """,
    parameters={"my_grocery": _get_grocery},
)
```

Для проверок качества данных см. [SQL check operators](https://www.astronomer.io/docs/learn/airflow-sql-data-quality).

---

[← К содержанию](README.md) | [Операторы →](operators.md) | [Connections →](connections.md)
