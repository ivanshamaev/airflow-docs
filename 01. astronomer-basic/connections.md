# Подключения (Connections) в Airflow

**Подключения (Connections)** в Airflow — наборы конфигураций для связи с другими инструментами в экосистеме данных. Поскольку большинство хуков и операторов используют подключения для отправки и получения данных из внешних систем, умение создавать и настраивать их необходимо для эксплуатации Airflow в production.

В этом руководстве вы:

- Добавите примеры подключений Snowflake и Slack Webhook в DAG.
- Узнаете, как задавать подключения через переменные окружения.
- Узнаете, как задавать подключения через Airflow UI.
- Познакомитесь с основами подключений в Airflow.

> **Инфо.** Клиентам Astro Astronomer рекомендует использовать [Astro Environment Manager](https://www.astronomer.io/docs/astro/manage-connections-variables#astro-cloud-ui-environment-manager) для хранения подключений в управляемом Astro secrets backend. Такие подключения можно использовать в нескольких развёрнутых и локальных окружениях Airflow. См. [Create Airflow connections in the Astro UI](https://www.astronomer.io/docs/astro/create-and-link-connections).

## Необходимая база

Чтобы получить максимум от руководства, нужно понимать:

- Хуки Airflow. См. [Хуки (Hooks)](hooks.md).
- Операторы Airflow. См. [Операторы](operators.md).
- Основные концепции Airflow. См. [Введение в Apache Airflow](https://www.astronomer.io/docs/learn/intro-to-airflow).

## Основы подключений в Airflow

Подключение в Airflow — это набор конфигураций для отправки запросов к API внешнего инструмента. В большинстве случаев для аутентификации Airflow во внешней системе нужны учётные данные (логин/пароль) или приватный ключ.

Подключения в Airflow можно создавать одним из способов:

- Файл [`airflow_settings.yaml`](https://www.astronomer.io/docs/astro/cli/develop-project#configure-airflow_settingsyaml-local-development-only) для пользователей Astro CLI.
- [Airflow CLI](https://airflow.apache.org/docs/apache-airflow/stable/howto/connection.html#connection-cli).
- [Secrets backend](https://airflow.apache.org/docs/apache-airflow/stable/security/secrets/secrets-backend/index.html) (внешняя система управления секретами).
- [Airflow REST API](https://airflow.apache.org/docs/apache-airflow/stable/stable-rest-api-ref.html#tag/Connection).
- [Переменные окружения](https://airflow.apache.org/docs/apache-airflow/stable/cli-and-env-variables-ref.html#environment-variables).
- [Airflow UI](airflow-ui.md).
- [Astro Environment Manager](https://www.astronomer.io/docs/astro/manage-connections-variables#astro-cloud-ui-environment-manager) — рекомендуемый способ для клиентов Astro.

В этом руководстве рассматриваются добавление подключений через Airflow UI и переменные окружения. Подробнее о других способах: [REST API reference](https://airflow.apache.org/docs/apache-airflow/stable/stable-rest-api-ref.html#tag/Connection), [Managing Connections](https://airflow.apache.org/docs/apache-airflow/stable/howto/connection.html) и [Secrets Backend](https://airflow.apache.org/docs/apache-airflow/stable/security/secrets/secrets-backend/index.html).

У каждого подключения уникальный **conn_id**, который передаётся в операторы и [хуки](hooks.md), требующие подключения.

Для унификации подключений в Airflow есть множество типов. Есть общие типы для облаков (`aws_default`, `gcp_default`) и типы для конкретных сервисов (например, `azure_service_bus_default`).

Каждый тип подключения требует своих полей и значений в зависимости от сервиса. Узнать, что нужно указать, можно так:

- Посмотреть исходный код хука, который использует ваш оператор.
- Проверить документацию внешнего инструмента по способам аутентификации.
- Открыть страницу провайдера в [Astronomer Registry](https://registry.astronomer.io/providers/) и по первой ссылке в Helpful Links перейти к документации Apache Airflow по этому провайдеру. У популярных провайдеров обычно описаны типы подключений. Например, настройка подключений к Azure — в [документации Azure provider](https://registry.astronomer.io/providers/microsoft-azure).

> **Совет.** Подробнее о том, как Airflow ищет подключения: [How Airflow finds connections](https://www.astronomer.io/docs/astro/manage-connections-variables#how-airflow-finds-connections). Если одно и то же подключение задано несколькими способами, Airflow использует следующий порядок приоритета: 1) метаданные Airflow (UI), 2) переменные окружения, 3) Astro Environment Manager, 4) Secrets Backend.

## Задание подключений в Airflow UI

Самый распространённый способ — Airflow UI. Перейдите в Admin → Connections и нажмите + Add Connection.

В форме подключения выберите **Connection Type** из выпадающего списка. От типа зависят доступные поля; разным типам нужны разные данные. При установке провайдеров в списке появляются новые типы. Например, после установки [Snowflake provider](https://registry.astronomer.io/providers/snowflake) появляется тип `Snowflake`. Если подходящего типа нет, можно использовать тип **generic**.

Для большинства подключений не обязательно заполнять все поля. Но пометки «обязательное» в UI не всегда полные. Например, для PostgreSQL нужно смотреть [документацию PostgreSQL provider](https://airflow.apache.org/docs/apache-airflow-providers-postgres/stable/connections/postgres.html): обычно нужны Host, логин (login), пароль (password) и чаще всего Port.

Параметры, для которых нет отдельных полей в форме, можно задать в поле **Extra** в виде JSON-словаря. Например, для PostgreSQL в Extra можно добавить `sslmode` или `sslkey` клиента.

## Задание подключений через переменные окружения

Подключения можно задавать переменными окружения. При использовании Astro CLI можно использовать файл `.env` для локальной разработки или задать переменные в Dockerfile проекта.

> **Примечание.** При синхронизации проекта с удалённым репозиторием не храните чувствительные данные в Dockerfile. Предпочтительнее secrets backend, подключения в UI или локальный `.env`, чтобы не держать секреты в открытом виде.

Переменная окружения для подключения должна иметь вид **AIRFLOW_CONN_<CONNID>** и содержать либо [URI](https://en.wikipedia.org/wiki/Uniform_Resource_Identifier), либо JSON.

URI — строка, в которой собрана вся информация о подключении: тип, логин, пароль, хост; часто добавляются порт, схема и доп. параметры.

```dockerfile
# общий формат URI-подключения в Dockerfile
ENV AIRFLOW_CONN_MYCONNID='my-conn-type://login:password@host:port/schema?param1=val1&param2=val2'

# пример подключения к Snowflake в виде URI
ENV AIRFLOW_CONN_SNOWFLAKE_CONN='snowflake://LOGIN:PASSWORD@/?account=xy12345&region=eu-central-1'
```

Подключение можно также задать переменной окружения в виде JSON-словаря:

```json
# пример подключения в виде JSON в файле .env
AIRFLOW_CONN_MYCONNID='{
    "conn_type": "my-conn-type",
    "login": "my-login",
    "password": "my-password",
    "host": "my-host",
    "port": 1234,
    "schema": "my-schema",
    "extra": {
        "param1": "val1",
        "param2": "val2"
    }
}'
```

Подключения, заданные переменными окружения, **не отображаются** в списке подключений в Airflow UI.

> **Инфо.** Чтобы хранить подключение в JSON как переменную окружения Astro, уберите все переносы строк в JSON — значение должно быть одной строкой. См. [Add Airflow connections and variables using environment variables](https://www.astronomer.io/docs/astro/environment-variables#add-airflow-connections-and-variables-using-environment-variables).

## Маскирование чувствительных данных

Подключения часто содержат учётные данные. По умолчанию Airflow скрывает поле **password** в UI и в логах. Если задано `AIRFLOW__CORE__HIDE_SENSITIVE_VAR_CONN_FIELDS=True`, также маскируются значения из поля **Extra**, ключи которых содержат слова из списка `AIRFLOW__CORE__SENSITIVE_VAR_CONN_NAMES`. Подробнее о маскировании и значениях по умолчанию: [Masking sensitive data](https://airflow.apache.org/docs/apache-airflow/stable/security/secrets/mask-sensitive-values.html).

## Проверка подключений в Airflow UI

Проверку подключений в UI можно включить, установив переменную окружения `AIRFLOW__CORE__TEST_CONNECTION` в значение `Enabled`.

Подробнее: [Testing connections](https://airflow.apache.org/docs/apache-airflow/stable/howto/connection.html#testing-connections) в документации Airflow.

## Пример: настройка SqlToSlackWebhookOperator (Snowflake и Slack)

В этом примере настраивается оператор, которому нужны подключения к SQL-базе (используем Snowflake) и к Slack. Подключения задаются через Airflow UI.

Перед запуском Airflow нужно установить провайдеры [Snowflake](https://registry.astronomer.io/providers/snowflake) и [Slack](https://registry.astronomer.io/providers/slack). При использовании Astro CLI добавьте в `requirements.txt` проекта Astro:

```text
apache-airflow-providers-snowflake>=6.2.0
apache-airflow-providers-slack>=9.0.3
```

Откройте Airflow UI и создайте новое подключение. Укажите Connection Type: **Snowflake**. Варианты настройки подключения к Snowflake см. в [Create a Snowflake Connection in Airflow](https://www.astronomer.io/docs/learn/connections/snowflake).

Затем настройте подключение к Slack. Чтобы отправлять сообщения в канал Slack, нужно создать Slack-приложение для вашего workspace и настроить Incoming Webhooks. Шаги описаны в [документации Slack](https://api.slack.com/messaging/webhooks).

Для подключения к Slack из Airflow укажите:

- **Password**: вторая часть URL вебхука в формате `T00000000/B00000000/XXXXXXXXXXXXXXXXXXXXXXXX`
- **Host**: `https://hooks.slack.com.services` — первая часть URL вебхука
- **Connection Type**: `Slack Webhook`
- **Connection Id**: `slack_conn` (или любая строка, ещё не использованная для другого подключения)

Последний шаг — написать DAG с оператором, который выполняет SQL-запрос к таблице Snowflake и отправляет результат в канал Slack. Оператору нужны conn_id подключения к Snowflake (`sql_conn_id`) и conn_id подключения к Slack (`slack_webhook_conn_id`). В примере ниже используется `SnowflakeToSlackOperator` (перенос данных из Snowflake в Slack).

```python
from airflow.decorators import dag
from pendulum import datetime
from airflow.providers.snowflake.transfers.snowflake_to_slack import (
    SnowflakeToSlackOperator,
)


@dag(start_date=datetime(2022, 7, 1), schedule=None, catchup=False)
def snowflake_to_slack_dag():
    transfer_task = SnowflakeToSlackOperator(
        task_id="transfer_task",
        # оба подключения передаются оператору здесь:
        snowflake_conn_id="snowflake_conn",
        slack_conn_id="slack_conn",
        params={"table_name": "ORDERS", "col_to_sum": "O_TOTALPRICE"},
        sql="""
            SELECT
              COUNT(*) AS row_count,
              SUM({{ params.col_to_sum }}) AS sum_price
            FROM {{ params.table_name }}
        """,
        slack_message="""The table {{ params.table_name }} has
            => {{ results_df.ROW_COUNT[0] }} entries
            => with a total price of {{results_df.SUM_PRICE[0]}}""",
    )

    transfer_task


snowflake_to_slack_dag()
```

---

[← BashOperator](bashoperator.md) | [К содержанию](README.md) | [Хуки →](hooks.md)
