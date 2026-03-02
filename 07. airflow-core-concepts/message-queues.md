# Message Queues (очереди сообщений)

**Message Queues** — способ задействовать внешнее событийное планирование DAG.

Apache Airflow изначально рассчитан на планирование по времени и по зависимостям. В то же время современные данные часто требуют почти реального времени и реакции на события из разных источников, в том числе из очередей сообщений.

В Airflow есть встроенная событийная модель: можно строить пайплайны, запускаемые внешними событиями, и делать их более отзывчивыми.

Airflow поддерживает событийное планирование на основе опроса (poll-based): Triggerer может опрашивать внешние очереди сообщений с помощью встроенных классов [airflow.triggers.base.BaseTrigger](https://airflow.apache.org/docs/apache-airflow/stable/_api/airflow/triggers/base/index.html#airflow.triggers.base.BaseTrigger). Так можно создавать workflow’ы, запускаемые внешними событиями — например, появлением сообщений в очереди или изменениями в БД — и делать это эффективно.

Airflow постоянно отслеживает состояние внешнего ресурса и обновляет ассет, когда ресурс достигает заданного состояния (если оно достигается). Для этого используются **Triggers** — небольшие асинхронные фрагменты Python-кода, которые опрашивают состояние внешнего ресурса.

Список поддерживаемых очередей сообщений: [Message Queues](https://airflow.apache.org/docs/apache-airflow-providers/core-extensions/message-queues.html).

---

*Источник: [Airflow 3.1.7 — Message Queues](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/message-queues.html). Перевод неофициальный.*
