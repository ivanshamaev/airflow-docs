# Airflow и dbt

[dbt Core](https://docs.getdbt.com/) — открытая библиотека для аналитической инженерии: с её помощью строят взаимозависимые SQL-модели для трансформации данных в хранилище, используя эфемерные вычисления хранилища. [Cosmos](https://astronomer.github.io/astronomer-cosmos/) — пакет с открытым исходным кодом от Astronomer для запуска dbt-моделей проекта dbt Core в Airflow.

### dbt в Airflow с Cosmos и Astro CLI

Пакет с открытым исходным кодом [Cosmos](https://astronomer.github.io/astronomer-cosmos/) позволяет встроить задания dbt в Airflow, автоматически создавая задачи Airflow из dbt-моделей. Проект dbt Core можно превратить в DAG или task group Airflow буквально несколькими строками кода.

Подробные инструкции по настройке Cosmos для разных хранилищ, опции конфигурации и оптимизация производительности описаны в [eBook «Orchestrating dbt with Apache Airflow® using Cosmos»](https://www.astronomer.io/ebooks/orchestrating-dbt-with-airflow-using-cosmos/?utm_source=website&utm_medium=learn-guides&utm_campaign=learn-dbt-tutorial-11-25) и в краткой выжимке [Quick Notes: Airflow + dbt with Cosmos](https://www.astronomer.io/ebooks/quick-notes-airflow-dbt-with-cosmos/?utm_source=website&utm_medium=learn-guides&utm_campaign=learn-dbt-tutorial-11-25).

## Зачем использовать Airflow с dbt Core?

dbt Core даёт возможность строить модульные, переиспользуемые SQL-компоненты со встроенным управлением зависимостями и [инкрементальными сборками](https://docs.getdbt.com/docs/build/incremental-models).

С [Cosmos](https://astronomer.github.io/astronomer-cosmos/) задания dbt можно интегрировать в окружение оркестрации на открытом Airflow — как отдельные DAG или как task group внутри DAG.

Преимущества связки Airflow и dbt Core:

- [Генерация](https://astronomer.github.io/astronomer-cosmos/configuration/generating-docs.html) и [хостинг](https://astronomer.github.io/astronomer-cosmos/configuration/hosting-docs.html) dbt docs в Airflow.
- Поддержка установки и запуска dbt в виртуальном окружении, чтобы избежать конфликтов зависимостей с Airflow.
- Запуск проектов dbt через [подключения Airflow](https://www.astronomer.io/docs/learn/connections) вместо dbt profiles. Все подключения можно хранить в одном месте — в Airflow или через [secrets backend](https://airflow.apache.org/docs/apache-airflow/stable/security/secrets/secrets-backend/index.html).
- Запуск `dbt test` по таблицам, созданным отдельными моделями, сразу после завершения модели. Ошибки видны до перехода к следующим шагам; можно добавить [проверки качества данных](https://www.astronomer.io/docs/learn/data-quality) и запускать их вместе с dbt test.
- Каждая dbt-модель становится задачей с возможностями Airflow: [повторы](https://www.astronomer.io/docs/learn/rerunning-dags#automatically-retry-tasks), [уведомления об ошибках](https://www.astronomer.io/docs/learn/error-notifications-in-airflow), полная наблюдаемость прошлых запусков в UI Airflow.
- [Data-aware планирование](https://www.astronomer.io/docs/learn/airflow-datasets) и [сенсоры Airflow](https://www.astronomer.io/docs/learn/what-is-a-sensor) для запуска моделей в зависимости от событий в экосистеме данных.

На Astro вы получаете всё перечисленное и можете разворачивать проект dbt на Astro Deployment отдельно от проекта Airflow с помощью Astro CLI. Подробнее: [Deploy dbt projects to Astro](https://www.astronomer.io/docs/astro/deploy-dbt-project).

## Время прохождения

Туториал рассчитан примерно на 30 минут.

## Необходимые знания

Чтобы получить максимум от туториала, желательно понимать:

- Подключения Airflow: [Manage connections in Apache Airflow](https://www.astronomer.io/docs/learn/connections).
- Task groups: [Airflow task groups](https://www.astronomer.io/docs/learn/task-groups).
- Операторы: [Operators 101](https://www.astronomer.io/docs/learn/what-is-an-operator).
- Соответствие понятий Airflow и dbt: [Similar dbt & Airflow concepts](https://astronomer.github.io/astronomer-cosmos/getting_started/dbt-airflow-concepts.html).
- Основы Airflow: написание DAG и определение задач — [Get started with Apache Airflow](https://www.astronomer.io/docs/learn/get-started-with-airflow).
- Основы dbt Core: [What is dbt?](https://docs.getdbt.com/docs/introduction).

## Предварительные требования

- Доступ к хранилищу данных, поддерживаемому dbt Core. Список: [dbt documentation](https://docs.getdbt.com/docs/supported-data-platforms). В туториале используется Postgres.
- [Astro CLI](https://www.astronomer.io/docs/astro/cli/overview).

Устанавливать dbt Core локально для прохождения туториала не обязательно.

## Шаг 1: Настройка проекта Astro

Чтобы использовать dbt Core с Airflow, установите dbt Core в виртуальном окружении и Cosmos в новом проекте Astro.

1. Создайте новый проект Astro:

```sh
$ mkdir astro-dbt-core-tutorial && cd astro-dbt-core-tutorial
$ astro dev init
```

2. Добавьте [Cosmos](https://github.com/astronomer/astronomer-cosmos), [провайдер Postgres для Airflow](https://registry.astronomer.io/providers/apache-airflow-providers-postgres/versions/latest) и [адаптер dbt Postgres](https://github.com/dbt-labs/dbt-adapters) в `requirements.txt` проекта Astro. Для другого хранилища замените `apache-airflow-providers-postgres` и `dbt-postgres` на соответствующие пакеты из [Astronomer registry](https://registry.astronomer.io/).

```text
astronomer-cosmos==1
apache-airflow-providers-postgres==6
apache-airflow-providers-common-sql==1
dbt-postgres==1
```

3. (Альтернатива) Если из‑за конфликтов пакетов нельзя установить dbt-адаптер в то же окружение, что и Airflow, можно создать исполняемый dbt в виртуальном окружении. В конец `Dockerfile` добавьте:

```text
# замените dbt-postgres на другой поддерживаемый адаптер при ином типе хранилища
RUN python -m venv dbt_venv && source dbt_venv/bin/activate && \
    pip install --no-cache-dir dbt-postgres && deactivate
```

Эта команда при сборке образа создаёт виртуальное окружение `dbt_venv` в контейнере scheduler Astro CLI и устанавливает в него `dbt-postgres` (в него входит и `dbt-core`). Для другого хранилища подставьте нужный адаптер.

Если установка адаптера через `requirements.txt` или виртуальное окружение в Docker недоступна, Cosmos можно запускать и другими способами. Подробнее: [Cosmos documentation on execution modes](https://astronomer.github.io/astronomer-cosmos/getting_started/execution-modes.html).

## Шаг 2: Подготовка проекта dbt

Чтобы подключить проект dbt к Airflow, добавьте папку проекта в окружение Airflow. Можно использовать свой проект или создать простой по шагам ниже (две модели).

1. В папке `my_simple_dbt_project` добавьте `dbt_project.yml`. В конфиге должен быть как минимум имя проекта. В туториале дополнительно показывается передача переменной `my_name` из Airflow в проект dbt.
2. В папке `dbt` создайте папку `my_simple_dbt_project`.
3. В папке `include` создайте папку `dbt`.

```yaml
version: '0.1'
name: 'my_simple_dbt_project'
vars:
    my_name: "No entry"
```

4. Добавьте dbt-модели в подпапку `models` внутри `my_simple_dbt_project`. Моделей может быть любое количество; в туториале используются две.

`model1.sql`:

```sql
SELECT '{{ var("my_name") }}' as name
```

`model2.sql`:

```sql
SELECT * FROM {{ ref('model1') }}
```

В `model1.sql` выбирается переменная `my_name`. В `model2.sql` используется зависимость от `model1.sql` и выбираются все данные из вышестоящей модели.

Итоговая структура в окружении Airflow:

```text
.
└── dags
└── include
    └── dbt
        └── my_simple_dbt_project
           ├── dbt_project.yml
           └── models
               ├── model1.sql
               └── model2.sql
```

Если хранить проект dbt рядом с проектом Airflow нельзя, Cosmos можно использовать и при размещении проекта в другом месте (например, разбор через manifest-файл и контейнерный режим выполнения). Подробнее: [Cosmos documentation](https://astronomer.github.io/astronomer-cosmos/configuration/index.html).

## Шаг 3: Подключение Airflow к хранилищу данных

Cosmos позволяет применять подключения Airflow к проекту dbt.

1. Создайте подключение с именем `db_conn`. Тип и параметры выберите в зависимости от хранилища. Для Postgres укажите:
2. В UI Airflow: Admin → Connections → +.
3. Запустите Airflow: `astro dev start`.

- Port: порт Postgres.
- Password: пароль Postgres.
- Login: имя пользователя Postgres.
- Schema: база данных Postgres.
- Host: адрес хоста Postgres.
- Connection Type: `Postgres`.
- Connection ID: `db_conn`.

Если нужного типа подключения нет, добавьте [соответствующий провайдер](https://registry.astronomer.io/) в `requirements.txt` и выполните `astro dev restart`.

## Шаг 4: Написание DAG Airflow

DAG создаёт задачи из существующих dbt-моделей через Cosmos и использует [SQLExecuteQueryOperator](https://registry.astronomer.io/providers/apache-airflow-providers-common-sql/versions/latest/modules/SQLExecuteQueryOperator) для запроса к созданной таблице. При необходимости можно добавить задачи до и после, встроив dbt в более широкий пайплайн.

1. В папке `dags` создайте файл `my_simple_dbt_dag.py`.
2. Скопируйте в него следующий код:

```python
"""
### Запуск проекта dbt Core как task group с Cosmos

Простой DAG: запуск проекта dbt как task group с использованием
подключения Airflow и передачей переменной в проект dbt.
"""

from airflow.sdk import dag, chain
from airflow.providers.common.sql.operators.sql import SQLExecuteQueryOperator
from cosmos import DbtTaskGroup, ProjectConfig, ProfileConfig, ExecutionConfig

# для других БД подставьте свой профиль
from cosmos.profiles.postgres import PostgresUserPasswordProfileMapping
import os

YOUR_NAME = "YOUR_NAME"
CONNECTION_ID = "db_conn"
DB_NAME = "YOUR_DB_NAME"
SCHEMA_NAME = "YOUR_SCHEMA_NAME"
MODEL_TO_QUERY = "model2"
# путь к проекту dbt
DBT_PROJECT_PATH = f"{os.environ['AIRFLOW_HOME']}/include/dbt/my_simple_dbt_project"

# ОПЦИОНАЛЬНО: путь к исполняемому dbt в venv из Dockerfile,
# если адаптер dbt нельзя ставить в requirements.txt из‑за конфликтов.
# DBT_EXECUTABLE_PATH = f"{os.environ['AIRFLOW_HOME']}/dbt_venv/bin/dbt"

profile_config = ProfileConfig(
    profile_name="default",
    target_name="dev",
    profile_mapping=PostgresUserPasswordProfileMapping(
        conn_id=CONNECTION_ID,
        profile_args={"schema": SCHEMA_NAME},
    ),
)

# ОПЦИОНАЛЬНО: путь к исполняемому dbt
# execution_config = ExecutionConfig(
#     dbt_executable_path=DBT_EXECUTABLE_PATH,
# )

@dag(
    params={"my_name": YOUR_NAME},
)
def my_simple_dbt_dag():
    transform_data = DbtTaskGroup(
        group_id="transform_data",
        project_config=ProjectConfig(DBT_PROJECT_PATH),
        profile_config=profile_config,
        # ОПЦИОНАЛЬНО: execution_config при использовании venv
        # execution_config=execution_config,
        operator_args={
            "vars": '{"my_name": {{ params.my_name }} }',
        },
        default_args={"retries": 2},
    )

    query_table = SQLExecuteQueryOperator(
        task_id="query_table",
        conn_id=CONNECTION_ID,
        sql=f"SELECT * FROM {DB_NAME}.{SCHEMA_NAME}.{MODEL_TO_QUERY}",
    )

    chain(transform_data, query_table)

my_simple_dbt_dag()
```

В DAG класс `DbtTaskGroup` из Cosmos создаёт task group из моделей проекта dbt. Зависимости между моделями dbt автоматически становятся зависимостями между задачами Airflow. Подставьте свои значения для `YOUR_NAME`, `YOUR_DB_NAME` и `YOUR_SCHEMA_NAME`.

Через ключ `vars` в словаре `operator_args` в проект dbt передаются переменные. Здесь передаётся `YOUR_NAME` в переменную `my_name`. Если в проекте есть dbt test, они запускаются сразу после завершения модели. Рекомендуется задавать `retries` не менее 2 для всех задач, запускающих dbt-модели.

При больших проектах dbt иногда возникает ошибка `DagBag import timeout`. Её можно устранить, увеличив значение настройки Airflow [core.dagbag_import_timeout](https://airflow.apache.org/docs/apache-airflow/stable/configurations-ref.html#dagbag-import-timeout).

3. Запустите DAG вручную (кнопка play) и откройте граф. Разверните [task groups](https://www.astronomer.io/docs/learn/task-groups), чтобы увидеть все задачи.

4. Проверьте [XCom](https://www.astronomer.io/docs/learn/airflow-passing-data-between-tasks) задачи `query_table` — в таблице `model2` должно быть ваше имя.

Класс `DbtTaskGroup` заполняет task group Airflow задачами, созданными из dbt-моделей внутри обычного DAG. Чтобы описать целый DAG только из dbt-моделей, используйте класс `DbtDag`, как в [документации Cosmos](https://astronomer.github.io/astronomer-cosmos/getting_started/astro.html).

Готово: вы запустили DAG с Cosmos, который автоматически создал задачи из dbt-моделей. Дальнейшая настройка Cosmos описана в [документации Cosmos](https://astronomer.github.io/astronomer-cosmos/index.html).

Для крупных проектов dbt и ускорения выполнения есть несколько вариантов. Один из недавних — экспериментальный режим watcher, который может сократить время выполнения DAG до 80% и приблизить его к скорости `dbt build` через dbt CLI. Подробнее: [Cosmos documentation](https://astronomer.github.io/astronomer-cosmos/getting_started/watcher-execution-mode.html).

## Другие способы запуска dbt Core с Airflow

Рекомендуется использовать Cosmos, но запускать dbt Core в Airflow можно и иначе.

### BashOperator

С помощью [BashOperator](https://registry.astronomer.io/providers/apache-airflow/modules/bashoperator) можно выполнять отдельные команды dbt. Желательно запускать `dbt-core` и адаптер dbt для вашей БД в виртуальном окружении из‑за частых конфликтов зависимостей с другими пакетами.

Пример DAG с BashOperator: активация venv и выполнение `dbt run` для проекта dbt.

```python
from airflow.sdk import dag
from airflow.providers.standard.operators.bash import BashOperator

PATH_TO_DBT_PROJECT = "<path to your dbt project>"
PATH_TO_DBT_VENV = "<path to your venv activate binary>"

@dag
def simple_dbt_dag():
    dbt_run = BashOperator(
        task_id="dbt_run",
        bash_command="source $PATH_TO_DBT_VENV && dbt run --models .",
        env={"PATH_TO_DBT_VENV": PATH_TO_DBT_VENV},
        cwd=PATH_TO_DBT_PROJECT,
    )

simple_dbt_dag()
```

Запуск `dbt run` и других команд dbt через BashOperator удобен при разработке. Но запуск dbt на уровне всего проекта имеет минусы:

- Любой сбой ведёт к необходимости перезапуска всех моделей проекта, что может быть затратно.
- Низкая наблюдаемость за состоянием выполнения проекта.

### Использование manifest-файла

Использование сгенерированного dbt файла `manifest.json` даёт больше ясности о шагах dbt в каждой задаче. Файл создаётся в каталоге `target` проекта dbt и содержит полное представление проекта. Подробнее: [dbt documentation](https://docs.getdbt.com/reference/dbt-artifacts/).

Cosmos умеет разбирать manifest-файлы: [Cosmos documentation](https://astronomer.github.io/astronomer-cosmos/configuration/parsing-methods.html).
