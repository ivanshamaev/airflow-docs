# Логирование (Logging)

Логи задач по умолчанию пишутся в указанное в конфигурации хранилище (локальная ФС или remote: S3, GCS, Azure Blob). Ключевые настройки: **remote_logging** (включение отправки в облако), **remote_base_log_folder** (путь в облаке), **logging_level** (уровень для логгеров Airflow), **task_log_reader** (как читать логи в UI — из локального файла или из remote).

![Логи задач в Grid](images/logging-task_logs_grid_view.gif)

Формат логов и имя файла можно настроить через конфиг; для задач путь обычно включает `dag_id`, `task_id`, `execution_date`, `try_number`. При использовании custom task handler или remote logging нужно установить соответствующий **task_log_handler** и при необходимости свой класс для чтения логов в UI.

![Кастомные цвета логов в UI](images/logging_logs_custom_colors.webp)

Логи планировщика и воркеров пишутся отдельно; для централизованного сбора часто используют внешние системы (CloudWatch, Stackdriver, ELK).

![Группы логов (CloudWatch)](images/logging_log_groups.gif)

При отправке логов в S3 используется настроенный bucket и структура каталогов:

![S3 bucket для логов](images/logs_s3_bucket.webp)

В UI Airflow доступны журнал событий (event log) и при необходимости аудит-логи:

![Журнал событий](images/logging_event_logs.webp)

![Аудит-логи](images/audit-logs-cap.webp)

В коде задач рекомендуется использовать стандартный модуль `logging`: `logging.getLogger(__name__)` или передавать логгер из контекста. Так логи корректно попадают в вывод задачи и в UI. Избегайте print для важной информации — используйте логгер с подходящим уровнем.

Подробнее: [Logging](https://www.astronomer.io/docs/learn/logging), [Airflow logging](https://airflow.apache.org/docs/apache-airflow/stable/logging-monitoring/logging-tasks.html).

---

[← KubernetesPodOperator](kubernetes-pod-operator.md) | [К содержанию](README.md) | [Мультиязычность →](multilanguage.md)
