# Event-driven планирование (Event-driven scheduling)

**Event-driven scheduling** — подвид data-aware планирования: DAG запускается при появлении сообщений в очереди (SQS, Kafka и т.д.). Подходит для реакций на внешние события: доставка данных во внешнюю систему, события IoT, пайплайны инференса.

![Паттерны event-driven планирования](images/3-0-airflow-event-driven-scheduling_patterns.png)

Ключевые понятия: **Asset** — сущность данных; **AssetEvent** — одно обновление ассета (в контексте event-driven — сообщение из очереди); **AssetWatcher** — следит за триггером и при срабатывании обновляет ассет; **Trigger** (наследник `BaseEventTrigger`) — асинхронно опрашивает очередь, при новом сообщении создаёт `TriggerEvent`, сообщение удаляется из очереди. В Airflow 3 поддерживаются Amazon SQS и Apache Kafka (через [Common Messaging](https://airflow.apache.org/docs/apache-airflow-providers-common-messaging/stable/triggers.html) и провайдеры Amazon/Kafka).

Пример с SQS: задать connection с `region_name` в extra; установить провайдеры (amazon, common-messaging); создать очередь; в DAG задать `MessageQueueTrigger(aws_conn_id=..., queue=URL, waiter_delay=...)`, ассет с `AssetWatcher` на этот триггер, `schedule=[sqs_queue_asset]`. Задача получает `triggering_asset_events` из контекста и обрабатывает тело сообщения. Для Kafka используется `MessageQueueTrigger` с `queue=kafka://...` и `apply_function` для разбора сообщения; connection и провайдер Kafka настраиваются отдельно. DAG может быть запущен с `logical_date=None` для одновременных run.

Подробнее: [Event-driven scheduling](https://www.astronomer.io/docs/learn/airflow-event-driven-scheduling), [Assets and data-aware scheduling](https://www.astronomer.io/docs/learn/airflow-datasets).

---

[← Deferrable](deferrable-operators.md) | [К содержанию](README.md) | [Human-in-the-loop →](human-in-the-loop.md)
