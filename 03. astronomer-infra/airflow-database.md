# Метаданные Airflow (Metadata database)

Метаданная база данных — ключевой компонент Airflow. В ней хранятся роли и права доступа, конфигурация окружения, а также все метаданные прошлых и текущих DAG run и task run.

От состояния метаданных БД зависит работа DAG и доступ к истории запусков. Резервное копирование и план восстановления после сбоев для этой БД обязательны.

В этом разделе: использование REST API для доступа к метаданным, лучшие практики, что хранится в БД, требования к СУБД.

## Спецификации БД

Airflow подключается к БД через SQLAlchemy и ORM. Теоретически подойдёт любая СУБД из [поддерживаемых SQLAlchemy](https://www.sqlalchemy.org/). На практике чаще всего:

- SQLite
- MySQL
- PostgreSQL

SQLite — по умолчанию в Apache Airflow; **PostgreSQL** рекомендуется сообществом и используется в Astronomer (в т.ч. Astro CLI и облачные окружения). Для продакшена обычно берут управляемую БД (автомасштабирование, бэкапы). Размер зависит от нагрузки. По умолчанию Apache Airflow использует SQLite 2 GB (только для разработки); Astro CLI — Postgres 1 GB.

Схема и конфигурация метаданных меняются почти с каждым минорным обновлением. Для отката версии Airflow используйте команду [`db downgrade`](https://airflow.apache.org/docs/apache-airflow/stable/howto/usage-cli.html#downgrading-airflow).

## Содержимое метаданных

- **Безопасность:** пользователи и [права](https://airflow.apache.org/docs/apache-airflow/stable/security/index.html). В UI: Security.
- **Конфигурации и переменные (Admin):** [Pools](https://www.astronomer.io/docs/learn/airflow-pools), [XCom](https://www.astronomer.io/docs/learn/airflow-passing-data-between-tasks), [Connections](https://www.astronomer.io/docs/learn/connections), [Variables](https://airflow.apache.org/docs/apache-airflow/stable/howto/variable.html). В UI: Admin.
- **DAG и task run (Browse):** планировщик опирается на эти данные. В UI: Browse.
  - SLA Misses, Triggers (deferrable), Task Reschedule, Task Instances, Audit logs, Jobs (SchedulerJob, TriggererJob, LocalTaskJob), DAG Runs.
- **Прочее:** теги DAG, сериализованный код DAG, ошибки импорта, состояние сенсоров. Теги отображаются у DAG в UI; ошибки импорта — вверху списка DAG; код DAG — в представлении Code по клику на DAG.

Предпочтительный способ получения данных — **UI или [REST API](https://airflow.apache.org/docs/apache-airflow/stable/stable-rest-api-ref.html)**. Прямые SQL-запросы к БД не рекомендуются (риск повреждения окружения). При необходимости можно использовать SQLAlchemy и модели Airflow.

## Лучшие практики

- Проверяйте [поддержку версии СУБД](https://airflow.apache.org/docs/apache-airflow/stable/howto/set-up-database.html#choosing-database-backend).
- В продакшене используйте управляемую БД (AWS RDS, Google Cloud SQL) или управляемый Airflow (например, [Astro](https://www.astronomer.io/lp/signup/)).
- Ограниченная память БД может ухудшать производительность. Не передавайте большие объёмы через XCom; настройте очистку и архивирование старых записей.
- Осторожно с [очисткой старых записей](https://airflow.apache.org/docs/apache-airflow/stable/usage-cli.html#purge-history-from-metadata-database) (`db clean`): это может повлиять на задачи с `depends_on_past`. Параметр `--clean-before-timestamp` задаёт границу удаления.
- При обновлении/откате Airflow: бэкап БД, проверка deprecated-функций, пауза всех DAG, отсутствие запущенных задач. См. [Upgrading](https://airflow.apache.org/docs/apache-airflow/stable/installation/upgrading.html).

## Доступ через REST API

Ниже — примеры через [stable REST API](https://airflow.apache.org/docs/apache-airflow/stable/stable-rest-api-ref.html). Нужна [настройка авторизации API](https://airflow.apache.org/docs/apache-airflow/stable/security/api.html) и корректный `ENDPOINT_URL` (локально обычно `http://localhost:8080/`).

### Количество успешно выполненных задач

GET-запрос к эндпоинту task instances с фильтром `state=success` (для всех DAG и DAG run можно использовать `~`). В ответе — `total_entries`. Аналогично: Browse → Task Instances в UI, фильтр по состоянию success; счётчик записей справа.

![Успешные задачи в UI](images/successful_tasks_UI.png)

Пример (логин/пароль из переменных окружения):

```python
import requests
import os

ENDPOINT_URL = "http://localhost:8080/"
user_name = os.environ["USERNAME_AIRFLOW_INSTANCE"]
password = os.environ["PASSWORD_AIRFLOW_INSTANCE"]

req = requests.get(
    f"{ENDPOINT_URL}/api/v1/dags/~/dagRuns/~/taskInstances?state=success",
    auth=(user_name, password),
)
print(req.json()["total_entries"])
```

### Пауза и снятие с паузы DAG

PATCH-запрос к `api/v1/dags/{dag_id}?update_mask=is_paused` с телом `{"is_paused": true}` (пауза) или `false` (снять с паузы).

### Удаление DAG

DELETE к `api/v1/dags/{dag_id}`. Нельзя удалить метаданные DAG, пока DAG ещё запущен. Файл с определением DAG не удаляется — при следующем парсинге DAG снова появится в UI без истории.

---

[← Компоненты](airflow-components.md) | [К содержанию](README.md) | [Исполнители →](executors.md)
