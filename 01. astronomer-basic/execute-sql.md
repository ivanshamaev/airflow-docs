# Выполнение SQL в Airflow

Выполнение SQL-запросов — один из самых частых сценариев в пайплайнах данных. Извлечение и загрузка данных, вызов хранимой процедуры или сложный запрос для отчёта — Airflow помогает оркестрировать эти процессы.

В этом руководстве вы узнаете о лучших практиках выполнения SQL из DAG, разберёте наиболее используемые SQL-операторы Airflow и на примерах реализуете несколько типичных сценариев.

## Необходимая база

Чтобы получить максимум от руководства, нужно понимать:

- Основы Snowflake. См. [Introduction to Snowflake](https://docs.snowflake.com/en/user-guide-intro.html).
- Операторы Airflow. См. [Операторы](operators.md).

## Рекомендации по выполнению SQL из DAG

Независимо от базы данных и варианта SQL, выполнять запросы через Airflow можно разными способами. После того как вы выбрали способ, следующие советы помогут держать DAG чистыми, читаемыми и эффективными.

### Используйте хуки и операторы

[Common SQL provider](https://registry.astronomer.io/providers/apache-airflow-providers-common-sql/versions/latest) — хорошая отправная точка для SQL-операторов. В него входит [SQLExecuteQueryOperator](https://registry.astronomer.io/providers/apache-airflow-providers-common-sql/modules/SQLExecuteQueryOperator) — универсальный оператор для разных БД, в том числе Snowflake и Postgres.

Для некоторых БД есть более специализированные операторы в пакете провайдера сервиса. Например, [SnowflakeSqlApiOperator](https://registry.astronomer.io/providers/apache-airflow-providers-snowflake/versions/latest/modules/SnowflakeSqlApiOperator) позволяет отправлять несколько SQL-выражений в одном запросе.

Хуки для работы с БД в функции с декоратором `@task` или в PythonOperator обычно входят в пакет провайдера этой БД. Например, [PostgresHook](https://registry.astronomer.io/providers/apache-airflow-providers-postgres/versions/latest/modules/PostgresHook) — часть Postgres provider.

> **Совет.** Для проверок качества данных используйте SQL check operators, см. [Run data quality checks using SQL check operators](https://www.astronomer.io/docs/learn/airflow-sql-data-quality).

### Не держите длинный SQL в DAG

Astronomer рекомендует не помещать длинные SQL-выражения в файл DAG. Запрос лучше хранить в отдельном `.sql` файле.

При использовании Astro CLI вспомогательный код (в том числе SQL-скрипты) можно класть в каталог `include/`:

```text
├─ dags/
|    └─ example-dag.py
├─ plugins/
├─ include/
|    ├─ query1.sql
|    └─ query2.sql
├─ Dockerfile
├─ packages.txt
└─ requirements.txt
```

Исключение — очень короткие запросы (например, `SELECT * FROM table`). Однострочные запросы можно оставлять прямо в DAG, если так код читается лучше.

## Примеры

Ниже несколько примеров выполнения SQL через Airflow. Примеры на Snowflake, но подход применим к большинству реляционных БД.

### Пример 1: выполнение запроса

В первом примере DAG выполняет два простых зависимых запроса с помощью [SQLExecuteQueryOperator](https://registry.astronomer.io/providers/apache-airflow-providers-common-sql/versions/latest/modules/SQLExecuteQueryOperator).

Сначала установите Common SQL и Snowflake провайдеры. При использовании Astro CLI добавьте в `requirements.txt`:

```text
apache-airflow-providers-snowflake
apache-airflow-providers-common-sql
```

Провайдер Snowflake нужен для подключения к Snowflake, Common SQL — для `SQLExecuteQueryOperator`.

Затем задайте DAG:

```python
from airflow.sdk import chain, dag
from airflow.providers.common.sql.operators.sql import SQLExecuteQueryOperator


@dag(template_searchpath="/usr/local/airflow/include")  # путь к SQL-файлам
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

Параметр `template_searchpath` в определении DAG задаёт каталог, в котором DAG ищет скрипты. Добавьте в проект два SQL-скрипта. В примере это `query1.sql` и `query2.sql` с таким содержимым:

**query1.sql:**

```sql
CREATE OR REPLACE TABLE MY_DATABASE.MY_SCHEMA.MY_NEW_TABLE (
    id INT,
    grocery STRING
);
```

**query2.sql:**

```sql
INSERT INTO MY_DATABASE.MY_SCHEMA.MY_NEW_TABLE (id, grocery)
VALUES
    (1, 'Chocolate'),
    (2, 'Eggs'),
    (3, 'Cake');
```

В этих файлах может быть любой нужный вам тип запроса.

Осталось настроить подключение к БД — в данном случае к Snowflake с connection ID `my_snowflake_conn`. Варианты управления подключениями см. в [руководстве по подключениям](connections.md) и в [настройке подключения Snowflake](https://www.astronomer.io/docs/learn/connections/snowflake).

### Пример 2: выполнение запроса с параметрами

В Airflow можно параметризовать SQL-запросы данными из контекста Airflow. Например, запрос выбирает данные из таблицы за дату, которую нужно задавать динамически. Используйте ту же схему, что в примере 1, с небольшими изменениями.

DAG может выглядеть так:

```python
from airflow.decorators import dag
from airflow.providers.common.sql.operators.sql import SQLExecuteQueryOperator
from pendulum import datetime, duration

default_args = {
    "owner": "airflow",
    "depends_on_past": False,
    "email_on_failure": False,
    "email_on_retry": False,
    "retries": 1,
    "retry_delay": duration(minutes=1),
}


@dag(
    start_date=datetime(2020, 6, 1),
    max_active_runs=3,
    schedule="@daily",
    default_args=default_args,
    template_searchpath="/usr/local/airflow/include",
    catchup=False,
)
def parameterized_query():
    opr_param_query = SQLExecuteQueryOperator(
        task_id="param_query", conn_id="snowflake", sql="param-query.sql"
    )

    opr_param_query


parameterized_query()
```

В этом примере запрос параметризован так, чтобы динамически выбирать данные за день до логической даты DAG.

Astronomer рекомендует по возможности использовать данные контекста Airflow или макросы — это повышает гибкость и помогает делать workflow [идемпотентными](https://en.wikipedia.org/wiki/Idempotence). Пример выше работает с любыми данными [контекста Airflow](../02.%20astronomer-dags/airflow-context.md). Например, [параметр уровня DAG](../02.%20astronomer-dags/airflow-params.md) доступен через словарь `params`:

```sql
SELECT *
FROM STATE_DATA
WHERE state = {{ params['my_state'] }}
```

Передавать данные в SQL-файл можно и через аргумент **parameters** в `SQLExecuteQueryOperator`. Это удобно, когда значение приходит из другой задачи DAG.

```python
from airflow.sdk import chain, dag, task
from airflow.providers.common.sql.operators.sql import SQLExecuteQueryOperator

@dag
def using_parameters_argument():

    @task
    def get_grocery():
        return "Chocolate"

    _get_grocery = get_grocery()

    opr_param_query = SQLExecuteQueryOperator(
        task_id="param_query",
        conn_id="my_snowflake_conn",
        sql="""
            SELECT *
            FROM DEMO_DB.DEMO_SCHEMA.MY_NEW_TABLE
            WHERE grocery = %(my_grocery)s;
        """,
        parameters={"my_grocery": _get_grocery},
    )

    chain(
        _get_grocery,
        opr_param_query,
    )

using_parameters_argument()
```

---

[← DAG](dags.md) | [К содержанию](README.md) | [Хуки →](hooks.md)
