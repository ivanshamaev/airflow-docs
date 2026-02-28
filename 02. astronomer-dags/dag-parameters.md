# Параметры DAG (DAG parameters)

Ниже перечислены основные параметры объекта DAG, влияющие на планирование, выполнение и отображение в UI.

## Планирование и даты

- **schedule** — расписание (cron, timedelta, timetable, список ассетов). `None` — только ручной/API запуск.
- **start_date** — дата/время, после которых DAG может планироваться.
- **end_date** — после этой даты новые запуски не создаются.
- **catchup** — заполнять ли пропущенные запуски между start_date и «сейчас» (по умолчанию False).

## Выполнение

- **max_active_runs** — максимум одновременных запусков этого DAG.
- **max_active_tasks** — максимум одновременных задач по всему DAG.
- **concurrency** — лимит одновременных задач (устаревший в пользу max_active_tasks в части версий).
- **dagrun_timeout** — таймаут одного DAG run.

## Поведение при сбоях

- **default_args** — словарь аргументов по умолчанию для всех задач (retries, retry_delay, email_on_failure, on_failure_callback и т.д.).
- **fail_fast** (Airflow 3+) — при True при падении одной задачи все остальные running помечаются failed, не запущенные — skipped. Несовместим с trigger_rule отличными от all_success.

## Идентификация и отображение

- **dag_id** — уникальный идентификатор DAG.
- **tags** — список тегов для фильтра в UI.
- **doc_md** / **doc** — документация DAG (Markdown или строка).

## Прочее

- **template_searchpath** — каталоги для поиска шаблонов (SQL, Jinja).
- **params** — словарь параметров DAG по умолчанию (см. [Params](airflow-params.md)).
- **render_template_as_native_obj** — рендерить Jinja в нативные объекты Python где поддерживается.

Полный и актуальный список: [DAG-level parameters](https://www.astronomer.io/docs/learn/airflow-dag-parameters), [Airflow DAG documentation](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/dags.html).

---

[← Лучшие практики](dag-best-practices.md) | [К содержанию](README.md) | [Версионирование →](dag-versioning.md)
