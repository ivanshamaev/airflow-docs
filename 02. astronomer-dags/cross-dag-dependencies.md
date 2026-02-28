# Зависимости между DAG (Cross-DAG dependencies)

Когда один DAG должен запускаться или ждать завершения другого DAG (или конкретной задачи), используются **cross-DAG dependencies**.

## Способы

- **ExternalTaskSensor** — задача ждёт завершения задачи в другом DAG (и опционально в том же logical date / data interval). Параметры: `external_dag_id`, `external_task_id`, `execution_delta` или `execution_date_fn` для выравнивания по времени.
- **TriggerDagRunOperator** — задача запускает другой DAG (передаёт конфиг в DAG run). Подходит для каскадного запуска.
- **Ассеты (Assets / Datasets)** — DAG A обновляет ассет (outlets), DAG B запланирован по этому ассету (`schedule=[Asset("...")]`). DAG B запускается после успешного обновления ассета задачей из DAG A. Не занимает слот воркера в отличие от сенсора. См. [Ассеты](../01. astronomer-basic/assets.md).
- **REST API** — внешняя система или задача вызывает API Airflow для запуска DAG или проверки статуса.

## Рекомендации

- Для data-driven сценариев предпочтительны ассеты; для простого «подождать другую задачу» — ExternalTaskSensor.
- Учитывайте выравнивание по датам: при разных расписаниях DAG нужно корректно задать `execution_delta` или эквивалент, чтобы сенсор ждал нужный run.

Подробнее: [Cross-DAG dependencies](https://www.astronomer.io/docs/learn/cross-dag-dependencies), [ExternalTaskSensor](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/sensors.html#externaltasksensor).

---

[← К содержанию](README.md) | [Ассеты →](../01. astronomer-basic/assets.md) | [Сенсоры →](../01. astronomer-basic/sensors.md)
