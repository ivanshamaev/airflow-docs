# Управление подключениями в Apache Airflow

**Подключения (connections)** в Airflow — наборы настроек для связи с внешними системами. Большинство хуков и операторов используют подключения для отправки и получения данных, поэтому их настройка важна для продакшена.

## Основы

Подключение содержит данные для запросов к API внешнего сервиса (учётные данные, ключи и т.д.). Создавать подключения можно:

- через [Airflow UI](airflow-ui.md) (Admin → Connections);
- через [переменные окружения](https://airflow.apache.org/docs/apache-airflow/stable/cli-and-env-variables-ref.html#environment-variables);
- через [REST API](https://airflow.apache.org/docs/apache-airflow/stable/stable-rest-api-ref.html#tag/Connection);
- через [secrets backend](https://airflow.apache.org/docs/apache-airflow/stable/security/secrets/secrets-backend/index.html);
- через [CLI](https://airflow.apache.org/docs/apache-airflow/stable/howto/connection.html#connection-cli).

У каждого подключения уникальный **conn_id**, который передаётся в операторы и [хуки](hooks.md). В Airflow есть типы подключений под облака и сервисы (например, Snowflake, Slack); при установке провайдеров появляются новые типы. Если подходящего типа нет, используется тип **generic**.

## Создание в UI

Admin → Connections → + Add Connection. Выберите **Connection Type** — от него зависят поля формы. Обязательные поля лучше уточнять в документации провайдера (например, [PostgreSQL](https://airflow.apache.org/docs/apache-airflow-providers-postgres/stable/connections/postgres.html)). Дополнительные параметры можно задать в поле **Extra** в виде JSON.

## Подключения через переменные окружения

Переменная должна называться **AIRFLOW_CONN_<CONNID>** и содержать либо URI, либо JSON.

**URI:**

```
my-conn-type://login:password@host:port/schema?param1=val1&param2=val2
```

Пример для Snowflake:

```
AIRFLOW_CONN_SNOWFLAKE_CONN='snowflake://LOGIN:PASSWORD@/?account=xy12345&region=eu-central-1'
```

**JSON:**

```json
AIRFLOW_CONN_MYCONNID='{"conn_type": "my-conn-type", "login": "my-login", "password": "my-password", "host": "my-host", "port": 1234, "schema": "my-schema", "extra": {"param1": "val1"}}'
```

Подключения, заданные переменными окружения, не отображаются в списке Connections в UI.

## Маскирование секретов

Поле `password` по умолчанию скрыто в UI и логах. При `AIRFLOW__CORE__HIDE_SENSITIVE_VAR_CONN_FIELDS=True` маскируются и ключи из **Extra**, перечисленные в `AIRFLOW__CORE__SENSITIVE_VAR_CONN_NAMES`. Подробнее: [Masking sensitive data](https://airflow.apache.org/docs/apache-airflow/stable/security/secrets/mask-sensitive-values.html).

## Проверка подключений в UI

Проверку подключений в UI можно включить параметром `AIRFLOW__CORE__TEST_CONNECTION=Enabled`. В форме подключения появится кнопка теста.

## Пример: SqlToSlackWebhookOperator

Оператор требует подключение к БД (например, Snowflake) и к Slack Webhook. Подключение к Slack Webhook: тип **Slack Webhook**, Host: `https://hooks.slack.com.services`, Password: вторая часть URL вебхука (`T00000000/B00000000/...`). В оператор передаются `sql_conn_id` (Snowflake) и `slack_webhook_conn_id` (Slack).

---

[← К содержанию](README.md) | [Хуки →](hooks.md)
