# Хуки (Hooks) в Airflow

**Хук (hook)** — абстракция над конкретным API для взаимодействия Airflow с внешней системой. Хуки встроены во многие операторы, но их можно использовать и напрямую в коде DAG.

Список хуков: [Astronomer Registry — Hooks](https://registry.astronomer.io/modules/?typeName=Hooks) (более 300). При необходимости можно реализовать [свой хук](https://www.astronomer.io/docs/learn/airflow-importing-custom-hooks-operators).

## Основы

Хуки оборачивают API и предоставляют методы для работы с внешней системой. Обычно достаточно **connection ID** из [подключений](connections.md). Все хуки наследуются от [BaseHook](https://github.com/apache/airflow/blob/main/airflow-core/src/airflow/hooks/base.py), который по connection ID устанавливает соединение. Конкретные хуки добавляют методы для действий в системе (например, S3Hook использует boto3 и содержит методы вроде `check_for_bucket`, `list_keys`, `load_file`, `download_file`).

**Пример в задаче:**

```python
@task
def my_task():
    from airflow.providers.amazon.aws.hooks.s3 import S3Hook
    s3_hook = S3Hook(aws_conn_id="my_aws_conn")
    # использование методов хука
```

## Когда использовать хуки напрямую

- Всегда предпочтительнее хук, а не ручные вызовы API.
- Хуки удобны в функциях с декоратором `@task` и в DAG с `@asset`.
- Кастомный оператор для внешней системы должен использовать хук.
- Если есть готовый оператор под задачу — используйте оператор, а не только хук.
- Если подходящего хука нет — можно написать свой и выложить в сообщество.

## Пример: S3Hook и SlackHook

В одном DAG можно прочитать несколько файлов из S3 через `S3Hook.read_key`, выполнить проверку по данным, отправить результат в Slack через `SlackHook.call(api_method="chat.postMessage", json={...})` и вернуть ответ API для логирования. Для этого нужны подключения к AWS и Slack; провайдеры: `apache-airflow-providers-amazon`, `apache-airflow-providers-slack`. Подключения создаются в Admin → Connections (для AWS — тип aws, Login/Password — ключи; для Slack — тип slack, Password — Bot User OAuth Token).

---

[← К содержанию](README.md) | [Операторы →](operators.md) | [Connections →](connections.md)
