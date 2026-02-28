# Сенсоры (Sensors) в Airflow

[Сенсоры Airflow](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/sensors.html) — операторы, которые **ждут** выполнения условия. Пока условие не выполнено, сенсор проверяет его с заданным интервалом; после выполнения задача помечается успешной и нижестоящие задачи могут запуститься.

Для долгих ожиданий предпочтительны **deferrable**-режим (освобождение слота воркера) или режим **reschedule**. Для запуска по сообщениям в очереди см. [event-driven scheduling](https://www.astronomer.io/docs/learn/airflow-event-driven-scheduling).

## Основы

Сенсоры наследуются от **BaseSensorOperator**. Основные параметры:

- **mode**: `poke` (по умолчанию) — сенсор занимает слот воркера и «дремлет» между проверками; подходит для короткого ожидания. `reschedule` — между проверками сенсор снимается с воркера и планируется на потом; подходит для долгого ожидания.
- **poke_interval** — интервал между проверками в режиме poke (секунды, по умолчанию 60).
- **exponential_backoff** — при True в poke-режиме интервалы между проверками растут экспоненциально.
- **timeout** — максимальное время ожидания (секунды); при превышении задача падает.
- **soft_fail** — при True при таймауте задача помечается skipped, а не failed.

## Часто используемые сенсоры

- **@task.sensor** — превращает функцию, возвращающую `PokeReturnValue`, в сенсор (сложная логика, API без своего сенсора).
- **S3KeySensor** — ожидание появления ключа (файла) в S3.
- **DateTimeSensor** — ожидание наступления даты/времени.
- **ExternalTaskSensor** — ожидание завершения задачи в другом DAG ([cross-DAG dependencies](https://www.astronomer.io/docs/learn/cross-dag-dependencies)).
- **HttpSensor** — ожидание доступности API.
- **SqlSensor** — ожидание появления данных по SQL-запросу.

Полный список: [Astronomer Registry — Sensors](https://registry.astronomer.io/modules?types=sensors).

## Пример: SqlSensor

Ожидание данных в Postgres с проверкой каждые 20 с, режим reschedule, таймаут 5 минут:

```python
waiting_for_partner = SqlSensor(
    task_id="waiting_for_partner",
    conn_id="postgres",
    sql="sql/CHECK_PARTNER.sql",
    parameters={"name": "partner_a"},
    success=_success_criteria,
    failure=_failure_criteria,
    fail_on_empty=False,
    poke_interval=20,
    mode="reschedule",
    timeout=60 * 5,
)
```

## Кастомный сенсор: @task.sensor и PythonSensor

**@task.sensor** — функция возвращает `PokeReturnValue(is_done=..., xcom_value=...)`; при `is_done=True` сенсор завершается, при False — повторная проверка через poke_interval. **PythonSensor** принимает `python_callable`, возвращающую True/False.

## Рекомендации

- Всегда задавайте осмысленный **timeout** (по умолчанию может быть до 7 дней).
- Для долгих сенсоров используйте **deferrable** или **reschedule**, чтобы не занимать слоты воркеров.
- При очень коротком poke_interval (меньше ~5 минут) предпочтительнее **poke**, иначе reschedule может перегрузить планировщик.
- Подбирайте **poke_interval** под задачу.

## Режимы при исключениях

- **soft_fail=True** — при исключении в poke задача помечается skipped.
- **silent_fail=True** — при исключении (кроме AirflowSensorTimeout, AirflowTaskTimeout, AirflowSkipException, AirflowFailException) ошибка логируется, сенсор продолжает работу.
- **never_fail=True** — при любом исключении задача skipped (несовместимо с soft_fail).

## Deferrable-операторы

У многих сенсоров есть параметр **deferrable=True** или отдельный deferrable-класс. Тогда нужен процесс **triggerer**. Рекомендуется использовать deferrable-режим для экономии ресурсов. Конфиг `operators.default_deferrable=True` включает deferrable по умолчанию для поддерживающих его сенсоров.

---

[← Операторы](operators.md) | [К содержанию](README.md) | [Зависимости задач →](task-dependencies.md)
