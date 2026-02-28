# Синхронное выполнение DAG (Synchronous execution)

В Airflow 3.1+ добавлено **синхронное выполнение**: запуск DAG run через API и ожидание его завершения с получением XCom, запушенных указанными задачами. Эндпоинт:

```text
GET api/v2/dags/{dag_id}/dagRuns/{dag_run_id}/wait
```

Параметры пути: `dag_id`, `dag_run_id`. Query-параметры: **interval** (обязательный) — интервал в секундах между проверками состояния; **result** (опционально) — массив task_id, для которых вернуть XCom по ключу `return_value`. Ответ — поток JSON-объектов со статусом (queued, running, success и т.д.); при завершении в последнем объекте поле **results** содержит запрошенные XCom.

Применение: бэкенд для сервисов, обрабатывающих запросы пользователя (сайт, мобильное приложение, бот): отправка данных в DAG и немедленный ответ с результатом, ad-hoc запросы на анализ, инференс (ввод пользователя → пайплайн с LLM/AI → ответ после завершения DAG). Функция экспериментальная в 3.1.

Подробнее: [Synchronous dag execution](https://www.astronomer.io/docs/learn/airflow-synchronous-dag-execution), [REST API](https://airflow.apache.org/docs/apache-airflow/stable/stable-rest-api-ref.html).

---

[← Общий код](sharing-code.md) | [К содержанию](README.md) | [Тестирование →](testing-airflow.md)
