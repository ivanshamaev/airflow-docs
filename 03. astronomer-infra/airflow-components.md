# Компоненты Airflow (Airflow components)

Понимание компонентов инфраструктуры Airflow помогает разрабатывать и запускать DAG, устранять неполадки и успешно эксплуатировать Airflow.

> Между Airflow 2 и Airflow 3 произошли существенные архитектурные изменения: улучшена безопасность и появились возможности вроде удалённого выполнения. Для авторов DAG важно: **прямой доступ к метаданным БД из задач больше невозможен**. Подробнее: [Upgrade from Airflow 2 to 3](https://www.astronomer.io/docs/learn/airflow-upgrade-2-3), [Release notes](https://airflow.apache.org/docs/apache-airflow/stable/release_notes.html).

## Основные компоненты

- **Triggerer** — отдельный процесс для асинхронных Python-функций (trigger classes). Нужен для [deferrable-операторов](https://www.astronomer.io/docs/learn/deferrable-operators) и [event-driven scheduling](https://www.astronomer.io/docs/learn/airflow-event-driven-scheduling).
- **Метаданные (metadata database)** — хранит подключения, сериализованные DAG, XCom, историю DAG run и task instance. Чаще всего используется PostgreSQL. См. [supported versions](https://airflow.apache.org/docs/apache-airflow/stable/howto/set-up-database.html).
- **DAG processor** — забирает и парсит файлы из [DAG bundle(s)](../02. astronomer-dags/dag-versioning.md) (см. [Версионирование DAG](../02. astronomer-dags/dag-versioning.md)).
- **API server** — FastAPI-сервер: UI Airflow и три API: для воркеров (выполнение задач), внутренний для UI (состояния задач и DAG run), [публичный REST API](https://airflow.apache.org/docs/apache-airflow/stable/stable-rest-api-ref.html).
- **Scheduler** — ядро Airflow: следит за задачами и DAG, планирует task instance по мере выполнения зависимостей. При создании DAG run выбирает последнюю [версию DAG](https://www.astronomer.io/docs/learn/airflow-dag-versioning). Для запуска задачи использует настроенный [executor](executors.md).

При локальном запуске через [Astro CLI](https://www.astronomer.io/docs/astro/install-cli) (`astro dev start`) поднимается пять контейнеров: API server, DAG processor, Triggerer, Scheduler, Postgres. Дополнительно — один или несколько воркеров; их тип зависит от [executor](executors.md).

## Взаимодействие компонентов

![Архитектура компонентов Airflow](images/3-0_airflow_component_architecture.png)

Кратко, что происходит при добавлении нового DAG:

1. **Scheduler** нужен статус task instance, чтобы понять, какие задачи готовы к запуску. **API server** отдаёт UI данные о DAG и задачах из метаданных БД.
2. Статусы задач важны для планировщика: он следит за DAG и, как только зависимости выполнены, планирует запуск task instance.
3. **Воркер** выполняет задачу; метаданные (статус, [XCom](https://www.astronomer.io/docs/learn/airflow-passing-data-between-tasks)) отправляются через API server в БД. При необходимости (например, [connection](https://www.astronomer.io/docs/learn/connections)) воркер запрашивает данные у API server; логи пишутся в настроенное хранилище.
4. Task instance планируются и попадают в очередь; воркеры забирают задачи из очереди.
5. Когда DAG готов к следующему run, **executor** решает, как и где запустить первые задачи DAG run.
6. Scheduler проверяет сериализованные DAG на предмет [расписания](https://www.astronomer.io/docs/learn/scheduling-in-airflow), обновлений [assets](https://www.astronomer.io/docs/learn/airflow-datasets) и событий [AssetWatchers](https://www.astronomer.io/docs/learn/airflow-event-driven-scheduling).
7. **DAG processor** парсит DAG и сохраняет сериализованную версию в метаданных БД.

## Управление инфраструктурой

Компоненты должны работать на инфраструктуре, соответствующей нагрузке. Локальный запуск через Astro CLI подходит для разработки и тестов, но не для продакшена.

Полезные ресурсы:

- Управляемый Airflow на [Astro](https://www.astronomer.io/product/) (есть [бесплатный пробный период](https://www.astronomer.io/lp/signup)).
- OSS [Official Helm Chart](https://airflow.apache.org/docs/apache-airflow/stable/installation/index.html#using-official-airflow-helm-chart).
- OSS [Production Docker Images](https://airflow.apache.org/docs/apache-airflow/stable/installation/index.html#using-production-docker-images).

Масштабирование: см. [Scaling Airflow](scaling-airflow.md).

## Высокая доступность (HA)

Запуск нескольких реплик Scheduler в режиме active-active повышает отказоустойчивость и производительность. В Astro HA включается при создании Deployment (переключатель в UI или `isHighAvailability: true` в [API](https://www.astronomer.io/docs/api) / [Terraform](https://registry.terraform.io/providers/astronomer/astro/latest/docs)).

---

[← К содержанию](README.md) | [Метаданные БД →](airflow-database.md) | [Исполнители →](executors.md)
