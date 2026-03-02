# Airflow Core Concepts — перевод на русский

Русский перевод раздела **Core Concepts** из официальной документации [Apache Airflow](https://airflow.apache.org/docs/apache-airflow/stable/).

Источник: [Airflow Documentation — Core Concepts](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/index.html). Перевод неофициальный, для личного использования.

## Содержание

| Страница | Описание |
|----------|----------|
| [Обзор архитектуры](architecture-overview.md) | Компоненты Airflow, scheduler, DAG processor, webserver, воркеры, базовая и распределённая архитектура |
| [DAG](dags.md) | Объявление DAG, зависимости задач, загрузка и запуск, default_args, декоратор @dag, ветвление, trigger rules, TaskGroups, Edge Labels, документация, упаковка, .airflowignore, пауза и удаление |
| [Dag Run](dag-run.md) | Статус Dag Run, data interval, catchup, backfill, повторный запуск задач, внешние запуски, передача параметров, ожидание завершения |
| [Tasks (задачи)](tasks.md) | Типы задач, связи upstream/downstream, Task Instance, состояния, таймауты, SLA, исключения, heartbeat, executor_config |
| [Operators (операторы)](operators.md) | Шаблоны задач, Jinja-шаблонизация, template_fields, literal(), render_template_as_native_obj, params, f-строки |
| [Sensors (сенсоры)](sensors.md) | Ожидание условий, режимы poke и reschedule, BaseSensorOperator, параметры сенсоров |
| [TaskFlow](taskflow.md) | Декоратор @task, XComArg, контекст, логирование, передача объектов, сериализация, сенсоры |
| [Справочник по шаблонам](templates-reference.md) | Переменные шаблонов, var/conn, фильтры ds/ts, макросы, TaskFlow-контекст |
| [Executor](executor.md) | Типы исполнителей, несколько исполнителей, BaseExecutor, написание своего исполнителя |
| [Auth manager](auth-manager.md) | Аутентификация и авторизация, BaseAuthManager, JWT, методы авторизации, CLI, расширение API |
| [Object Storage](objectstorage.md) | Абстракция объектных хранилищ (s3, gcs, Azure), Path API, fsspec, копирование, интеграции |
| [Backfill](backfill.md) | Создание запусков за прошедшие даты, переобработка, max_active_runs, dry run, CLI и UI |
| [Message Queues](message-queues.md) | Событийное планирование DAG, опрос очередей, Triggers, BaseTrigger |
| [XComs](xcoms.md) | Обмен данными между задачами, xcom_push/xcom_pull, бэкенды XCom, Object Storage, кастомные бэкенды |
| [Variables](variables.md) | Глобальное хранилище ключ–значение, Variable.get(), шаблоны, отличия от XCom |
| [Params](params.md) | Runtime-параметры DAG и задач, JSON Schema, форма Trigger UI, секции, обязательные поля |
| [CLI (краткий обзор)](cli-overview.md) | Выжимка по CLI и переменным окружения: группы команд, dags/tasks/db/variables/connections/backfill/config, таблицы аргументов, env vars |

---

*Документация основана на Airflow 3.x. Некоторые элементы могут отличаться в других версиях.*
