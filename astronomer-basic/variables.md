# Переменные (Variables) в Airflow

**Переменная Airflow** — пара ключ–значение для хранения информации в окружении. Часто используются для редко меняющихся настроек уровня инстанса: API-ключи, пути к конфигурации и т.п.

Бывают обычные значения и JSON-сериализованные (отображаются в UI по-разному).

## Рекомендации

- Переменные — для данных, зависящих от окружения и не меняющихся слишком часто.
- Избегайте обращения к переменным на верхнем уровне DAG (вне задач): при каждом парсинге DAG это даёт обращение к метаданным и может замедлять загрузку. Лучше читать переменные внутри задач или через [Jinja-шаблоны](https://www.astronomer.io/docs/learn/templating) (вычисляются при выполнении задачи).
- Переменные в метаданных шифруются (Fernet). Для маскирования в UI и логах в имени переменной должна быть подстрока из списка чувствительных (password, secret, api_key и т.д.) или настройка `sensitive_var_conn_names`. См. [Hiding sensitive information](#masking).

Другие способы хранения: **environment variables** (через `os.getenv`), **Params** (на уровне DAG/DAG run, не для секретов), **XCom** (обмен между задачами, не для долгоживущих секретов).

## Создание переменных

- **UI:** Admin → Variables → + , ввести ключ, значение, опционально описание; можно импорт из файла.
- **CLI:** `airflow variables set my_var my_value`, для JSON: `airflow variables set -j my_json_var '{"key": "value"}'`. В Astro: `astro dev run variables set ...`.
- **Переменные окружения:** префикс `AIRFLOW_VAR_`, имя переменной в верхнем регистре: `AIRFLOW_VAR_MYREGULARVAR='my_value'`, `AIRFLOW_VAR_MYJSONVAR='{"hello":"world"}'`.
- **Из задачи:** `Variable.set(key="my_regular_var", value="Hello!")`, для JSON `Variable.set(..., serialize_json=True)`. Обновление — метод `.update()`.

## Получение переменных

- В коде: `Variable.get("my_regular_var", default=None)`, для JSON `Variable.get("my_json_var", deserialize_json=True, default=None)`.
- Из контекста: `context["var"]["value"].get("my_regular_var")`, для JSON `context["var"]["json"].get("my_json_var")`.
- В шаблонах Jinja (операторы): `{{ var.value.my_regular_var }}`, `{{ var.json.my_json_var.num2 }}`.

Чтение в верхнем уровне DAG создаёт запрос к БД при каждом парсинге; предпочтительнее Jinja в шаблонируемых полях или чтение внутри задачи.

## Маскирование чувствительных данных (masking)

При включённом `hide_sensitive_var_conn_fields` переменные, в имени которых есть подстроки из списка (access_token, api_key, password, secret, token и др.), маскируются в UI и логах. Список расширяется через конфиг `sensitive_var_conn_names`. Для секретов в нескольких инстансах удобен [Secrets Backend](https://airflow.apache.org/docs/apache-airflow/stable/security/secrets/secrets-backend/index.html).

---

[← К содержанию](README.md) | [Connections →](connections.md)
