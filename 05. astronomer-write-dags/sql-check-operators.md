# SQL check operators (Проверка качества данных)

Операторы из [Common SQL provider](https://registry.astronomer.io/providers/apache-airflow-providers-common-sql/versions/latest) для проверки качества данных в DAG: при падении проверки задача падает, пайплайн останавливается до попадания плохих данных в хранилище.

## Основные операторы

- **SQLColumnCheckOperator** — проверки по колонкам: `null_check`, `distinct_check`, `unique_check`, `min`, `max`. Параметр `column_mapping` — словарь колонка → проверки. Сравнение: `less_than`, `leq_to`, `equal_to`, `geq_to`, `greater_than`. Опционально: `partition_clause` (WHERE без ключевого слова), `tolerance` (допуск), `accept_none`.
- **SQLTableCheckOperator** — произвольные SQL-проверки на уровне таблицы: `row_count_check`, `average_happiness_check` и т.д. Параметр `checks` — словарь. Поддерживается `partition_clause` на уровне оператора и отдельной проверки.
- **SQLCheckOperator** — любой SQL, возвращающий одну строку. Задача падает, если любое значение в строке при приведении к bool даёт `False` (например, 0). Подходит для сложных проверок, сравнений между таблицами. SQL можно задать строкой или ссылкой на файл (например, `custom_check.sql`).

## Дополнительно

- **SQLIntervalCheckOperator** — сравнение текущих данных с историческими.
- Рекомендуется использовать `SQLColumnCheckOperator` и `SQLTableCheckOperator` вместо устаревших `SQLValueCheckOperator` и `SQLThresholdCheckOperator`.

Для работы с файлами SQL задайте `template_searchpath` у DAG (например, `["/usr/local/airflow/include/"]`). Результаты проверок видны в логах задачи.

Подробнее: [SQL data quality](https://www.astronomer.io/docs/learn/airflow-sql-data-quality), [Data quality](https://www.astronomer.io/docs/learn/data-quality).

---

[← PyCharm](pycharm-local-dev.md) | [К содержанию](README.md) | [VS Code →](vscode-local-dev.md)
