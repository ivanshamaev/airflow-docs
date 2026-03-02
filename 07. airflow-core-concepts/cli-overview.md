# CLI и переменные окружения — краткий обзор

Краткая выжимка по [официальному справочнику CLI и переменных окружения Airflow](https://airflow.apache.org/docs/apache-airflow/stable/cli-and-env-variables-ref.html). Подробное использование: [Using the Command Line Interface](https://airflow.apache.org/docs/apache-airflow/stable/howto/usage-cli.html).

Использование: `airflow [-h] GROUP_OR_COMMAND ...`

## Основные группы команд

| Группа | Описание |
|--------|----------|
| `api-server` | Запуск API-сервера Airflow |
| `assets` | Управление ассетами (details, list, materialize) |
| `backfill` | Backfill (create — прогон DAG за диапазон дат) |
| `cheat-sheet` | Вывод шпаргалки |
| `config` | Конфигурация (get-value, lint, list, update) |
| `connections` | Подключения (add, delete, export, get, import, list, test) |
| `dag-processor` | Запуск экземпляра DAG processor |
| `dags` | DAG: list, trigger, pause, unpause, test, list-runs, state и др. |
| `db` | Операции с БД (check, migrate, clean, reset, shell и др.) |
| `db-manager` | Внешние менеджеры БД (downgrade, migrate, reset) |
| `info` | Информация о Airflow и окружении |
| `jobs` | Управление джобами (check) |
| `plugins` | Информация о загруженных плагинах |
| `pools` | Пуллы (delete, export, get, import, list, set) |
| `providers` | Провайдеры (list, get, hooks, triggers и др.) |
| `scheduler` | Запуск планировщика |
| `standalone` | Запуск в режиме standalone |
| `tasks` | Задачи: list, test, state, render и др. |
| `triggerer` | Запуск triggerer |
| `variables` | Переменные (get, set, list, delete, import, export) |
| `version` | Версия Airflow |

Дополнительные команды от провайдеров: [Celery](https://airflow.apache.org/docs/apache-airflow-providers-celery/stable/cli-ref.html), [Kubernetes](https://airflow.apache.org/docs/apache-airflow-providers-cncf-kubernetes/stable/cli-ref.html), [AWS](https://airflow.apache.org/docs/apache-airflow-providers-amazon/stable/cli-ref.html); пользователи и роли — [FAB provider](https://airflow.apache.org/docs/apache-airflow-providers-fab/stable/cli-ref.html).

---

## dags

Управление DAG.

### airflow dags list

| Аргумент | Описание |
|----------|----------|
| `dag_id` | (позиционный) ID DAG |
| `-B, --bundle-name` | Имя DAG bundle (можно указать несколько раз) |
| `--columns` | Список колонок (по умолчанию: dag_id, fileloc, owners, is_paused, bundle_name, bundle_version) |
| `-l, --local` | Показать локально разобранные DAG и ошибки импорта, без учёта сериализованного в БД |
| `-o, --output` | Формат: table, json, yaml, plain |

### airflow dags trigger

| Аргумент | Описание |
|----------|----------|
| `dag_id` | (позиционный) ID DAG |
| `-c, --conf` | JSON-конфиг DagRun (conf) |
| `-l, --logical-date` | Логическая дата DAG |
| `-r, --run-id` | Идентификатор запуска |
| `--no-replace-microseconds` | Не обнулять микросекунды (по умолчанию True — обнулять) |
| `-o, --output` | Формат вывода |

### airflow dags pause / unpause

| Аргумент | Описание |
|----------|----------|
| `dag_id` | (позиционный) ID DAG или regex при `--treat-dag-id-as-regex` |
| `--treat-dag-id-as-regex` | Считать dag_id регулярным выражением |
| `-y, --yes` | Не спрашивать подтверждение |
| `-o, --output` | Формат вывода |

### airflow dags list-runs

| Аргумент | Описание |
|----------|----------|
| `dag_id` | (позиционный) ID DAG |
| `-s, --start-date` | Начальная дата (фильтр) |
| `-e, --end-date` | Конечная дата (фильтр) |
| `--state` | queued, running, success, failed |
| `--no-backfill` | Исключить backfill-запуски |
| `-o, --output` | Формат вывода |

### airflow dags test

| Аргумент | Описание |
|----------|----------|
| `dag_id` | (позиционный) ID DAG |
| `logical_date` | (позиционный, опционально) Логическая дата |
| `-B, --bundle-name` | Имя DAG bundle |
| `-f, --dagfile-path` | Путь к файлу DAG |
| `-c, --conf` | JSON conf для DagRun |
| `--mark-success-pattern` | Regex: не запускать задачи, помечать успешными (например, сенсоры при локальном тесте) |
| `--use-executor` | Использовать executor (по умолчанию DAG запускается без executor) |
| `--save-dagrun` | Сохранить диаграмму DAG Run в файл (формат по расширению) |
| `--show-dagrun` | Показать диаграмму в формате DOT |

### airflow dags delete / details / state / next-execution / show / reserialize / report / list-import-errors / list-jobs

- **delete** — удалить все записи DAG из БД: `dag_id`, `-y`, `-v`
- **details** — детали DAG: `dag_id`, `-o`
- **state** — статус Dag Run: `dag_id`, `logical_date_or_run_id`
- **next-execution** — следующие логические даты: `dag_id`, `-n NUM_EXECUTIONS`
- **show** — граф DAG: `dag_id`, `--imgcat`, `-s SAVE`
- **reserialize** — пересериализовать DAG в БД (после смены версии Airflow)
- **report** — отчёт загрузки DagBag
- **list-import-errors** — DAG с ошибками импорта
- **list-jobs** — джобы: `-d DAG_ID`, `--state`, `--limit`

---

## backfill create

Прогон части DAG за диапазон дат.

| Аргумент | Описание |
|----------|----------|
| `--dag-id` | ID DAG |
| `--from-date` | Начальная логическая дата |
| `--to-date` | Конечная логическая дата |
| `--dag-run-conf` | JSON conf DagRun |
| `--dry-run` | Только симуляция |
| `--max-active-runs` | Макс. число активных запусков |
| `--reprocess-behavior` | none, completed, failed — при уже существующем run |
| `--run-backwards` | От новой даты к старой (не поддерживается при depend_on_past) |
| `--run-on-latest-version` | (эксп.) Использовать последнюю версию bundle (по умолчанию True) |

---

## tasks

Работа с задачами.

### airflow tasks list

| Аргумент | Описание |
|----------|----------|
| `dag_id` | (позиционный) ID DAG |
| `-B, --bundle-name` | Имя DAG bundle |

### airflow tasks test

Запуск одной задачи без учёта зависимостей и без записи состояния в БД.

| Аргумент | Описание |
|----------|----------|
| `dag_id`, `task_id` | (позиционные) ID DAG и задачи |
| `logical_date_or_run_id` | (позиционный, опционально) Логическая дата или run_id |
| `-B, --bundle-name` | Имя DAG bundle |
| `--map-index` | Индекс для mapped task |
| `-t, --task-params` | JSON params для задачи |
| `--env-vars` | JSON env vars (parsing + runtime) |
| `-m, --post-mortem` | Открыть отладчик при необработанном исключении |

### airflow tasks state / states-for-dag-run / render

- **state** — статус одного Task Instance: `dag_id`, `task_id`, `logical_date_or_run_id`, `--map-index`
- **states-for-dag-run** — статусы всех TI в Dag Run: `dag_id`, `logical_date_or_run_id`, `-o`
- **render** — отрендерить шаблоны задачи: `dag_id`, `task_id`, `logical_date_or_run_id`, `--map-index`

---

## db

Операции с БД метаданных.

### airflow db check / check-migrations

| Команда | Аргументы | Описание |
|---------|-----------|----------|
| `db check` | `--retry`, `--retry-delay` | Проверка доступности БД |
| `db check-migrations` | `-t, --migration-wait-timeout` | Ожидание завершения миграций (или до таймаута) |

### airflow db migrate / downgrade

| Аргумент | Описание |
|----------|----------|
| `-r, --to-revision` | Ревизия Alembic (migrate — до какой, downgrade — до какой откатиться) |
| `-n, --to-version` | Версия Airflow (альтернатива to-revision) |
| `--from-revision` / `--from-version` | Только при `-s` (генерация SQL) |
| `-s, --show-sql-only` | Только вывести SQL, не выполнять |

### airflow db clean

Очистка старых записей в таблицах метаданных.

| Аргумент | Описание |
|----------|----------|
| `--clean-before-timestamp` | Удалить/архивировать данные до этой даты/времени |
| `-t, --tables` | Таблицы (через запятую), например: dag_run, task_instance, log, job, xcom и др. |
| `--batch-size` | Размер батча при удалении/архиве |
| `--dry-run` | Только симуляция |
| `--skip-archive` | Не сохранять в архивную таблицу |
| `-y, --yes` | Без подтверждения |

### airflow db reset / shell / drop-archived / export-archived

- **reset** — пересоздать БД: `-s` (только удалить таблицы, без init), `-y`
- **shell** — интерактивная оболочка к БД
- **drop-archived** — удалить архивные таблицы (созданные через db clean): `-t`, `-y`
- **export-archived** — экспорт из архивных таблиц: `--output-path`, `--export-format csv`, `-t`, `--drop-archives`, `-y`

---

## variables

Управление переменными.

| Команда | Аргументы | Описание |
|---------|-----------|----------|
| `variables get key` | `-d, --default`, `-j, --json` | Получить значение (default — если нет ключа; json — десериализовать JSON) |
| `variables set key VALUE` | `--description`, `-j, --json` | Установить переменную |
| `variables list` | `-o, --output` | Список переменных |
| `variables delete key` | — | Удалить переменную |
| `variables export file` | — | Экспорт в JSON-файл |
| `variables import file` | `-a, --action-on-existing-key`: overwrite, fail, skip | Импорт из .env, .json, .yaml, .yml |

---

## connections

Управление подключениями.

| Команда | Основные аргументы | Описание |
|---------|--------------------|----------|
| `connections add conn_id` | `--conn-uri` или `--conn-type` + host/login/port/schema/password/extra; `--conn-json` | Добавить подключение |
| `connections get conn_id` | `-o` (table, json, yaml, plain) | Получить подключение |
| `connections list` | `--conn-id`, `-o` | Список подключений |
| `connections delete conn_id` | — | Удалить подключение |
| `connections test conn_id` | — | Проверить подключение |
| `connections export file` | `--file-format` (json, yaml, env), `--serialization-format` (json, uri) | Экспорт |
| `connections import file` | `--overwrite` | Импорт |

---

## config

Просмотр и обновление конфигурации.

| Подкоманда | Описание |
|------------|----------|
| `config get-value section option` | Вывести значение опции |
| `config list` | Список опций: `--section`, `-d` (описания), `-V` (env vars), `-e` (примеры), `-s` (источники), `-a` (только defaults), `-p` (без провайдеров), `-c` (закомментировать всё) |
| `config lint` | Проверка конфига при миграции 2.x → 3.0: `--section`, `--option`, `--ignore-section`, `--ignore-option` |
| `config update` | Применение изменений при миграции: `--fix`, `--all-recommendations`, те же фильтры section/option |

---

## Переменные окружения

| Переменная | Описание |
|------------|----------|
| `AIRFLOW__{SECTION}__{KEY}` | Установка опции конфигурации (приоритет над airflow.cfg). Пример: `AIRFLOW__CORE__DAGS_FOLDER` |
| `AIRFLOW__{SECTION}__{KEY}_CMD` | Команда, результат которой подставляется как значение (поддерживается для sql_alchemy_conn, fernet_key, broker_url, result_backend, secret_key и др.) |
| `AIRFLOW__{SECTION}__{KEY}_SECRET` | Получить значение из Secrets Backend для той же группы опций, что и _CMD |
| `AIRFLOW_CONFIG` | Путь к файлу конфигурации Airflow |
| `AIRFLOW_HOME` | Корневой каталог Airflow (DAG, логи и т.д.) |
| `AIRFLOW_CONN_{CONN_ID}` | Подключение с именем `{CONN_ID}`; значение — URI. Пример: `AIRFLOW_CONN_PROXY_POSTGRES_TCP` |
| `AIRFLOW_VAR_{KEY}` | Переменная Airflow с именем `{KEY}` |

Подробнее: [Setting Configuration Options](https://airflow.apache.org/docs/apache-airflow/stable/howto/set-config.html), [Storing connections in environment variables](https://airflow.apache.org/docs/apache-airflow/stable/howto/connection.html#environment-variables-connections), [Managing Variables](https://airflow.apache.org/docs/apache-airflow/stable/howto/variable.html#managing-variables).

---

*Источник: [CLI and Environment Variables Reference](https://airflow.apache.org/docs/apache-airflow/stable/cli-and-env-variables-ref.html). Выжимка неофициальная.*
