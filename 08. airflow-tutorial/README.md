# Airflow Tutorial — перевод на русский

Русский перевод официального **туториала Apache Airflow** из [документации Airflow](https://airflow.apache.org/docs/apache-airflow/stable/tutorial/index.html).

## Содержание

| Страница | Описание |
|----------|----------|
| [Airflow 101: первый workflow](fundamentals.md) | Основы: что такое DAG, пример пайплайна, операторы, задачи, Jinja, документация, зависимости, тестирование |
| [Pythonic DAG с TaskFlow API](taskflow.md) | ETL на @dag и @task, передача данных, multiple_outputs, virtualenv/Docker/K8s, сенсоры, шаблоны, run_if/skip_if |
| [Построение простого пайплайна](pipeline.md) | CSV → Postgres: SQLExecuteQueryOperator, подключения, staging, merge, Docker Compose |
| [Object Storage (облачное хранилище)](objectstorage.md) | ObjectStoragePath, API → Parquet в S3, анализ DuckDB, fsspec, переносимые пайплайны |
| [HITL (Human-in-the-Loop)](hitl.md) | HITLEntryOperator, HITLOperator, ApprovalOperator, HITLBranchOperator, notifiers, ввод и выбор вариантов |
| [Лучшие практики](best-practices.md) | Написание DAG, код верхнего уровня, переменные, расписания, watcher, снижение сложности, тестирование, моки, БД, обновления, конфликтующие зависимости (virtualenv, external Python, Docker, K8s) |

---

*Источник: [Airflow Tutorial](https://airflow.apache.org/docs/apache-airflow/stable/tutorial/index.html), [Best Practices](https://airflow.apache.org/docs/apache-airflow/stable/best-practices.html). Перевод неофициальный.*
