# Ассеты и data-aware планирование в Airflow

С помощью **ассетов (Assets)** DAG, работающие с одними и теми же данными, получают явные зависимости, а DAG можно планировать по **обновлению ассетов**. Это выходит за рамки чисто временного (cron) планирования.

Пример: DAG инженеров данных создаёт ассет, DAG ML-команды обучает модель по этому ассету. Через ассеты DAG ML запускается только после обновления ассета командой данных.

## Синтаксис @asset

Декоратор `@asset` создаёт один DAG с одной задачей, которая обновляет ассет. DAG ID и task ID по умолчанию совпадают с именем ассета.

```python
from airflow.sdk import asset

@asset(schedule="@daily")
def my_asset():
    # логика задачи
    pass
```

Расписание можно задать по другому ассету (data-centric). Данные между DAG при этом передаются через cross-DAG XCom.

**Обновление нескольких ассетов из одного DAG** — `@asset.multi`:

```python
from airflow.sdk import Asset, asset

@asset.multi(schedule="@daily", outlets=[Asset("asset_a"), Asset("asset_b")])
def my_multi_asset():
    pass
```

## Основные понятия

- **Asset** — объект с уникальным именем, представляющий конкретный или абстрактный объект данных; опционально URI (файл, таблица).
- **Asset event** — событие, привязанное к ассету при успешном обновлении producing task; может содержать `extra`.
- **Asset schedule** — расписание DAG по одному или нескольким ассетам (DAG запускается при появлении событий).
- **Producer task** — задача, у которой в параметре **outlets** указаны ассеты; при успешном завершении создаёт события ассетов.
- **Outlets** — список ассетов, которые задача обновляет. **Inlets** — ассеты, к событиям которых задача получает доступ (на расписание DAG не влияют).

Задачи объявляют ассеты в **outlets**; DAG объявляют расписание по ассетам в **schedule**. Ассет появляется в метаданных при первом упоминании в outlets или schedule.

## Базовое использование

**Producing DAG** — задача с `outlets=[Asset("my_asset")]`:

```python
from airflow.sdk import Asset, dag, task

@dag
def my_producer_dag():
    @task(outlets=[Asset("my_asset")])
    def my_producer_task():
        pass
    my_producer_task()

my_producer_dag()
```

**Consumer DAG** — расписание по ассету:

```python
@dag(schedule=[Asset("my_asset")])
def my_consumer_dag():
    EmptyOperator(task_id="empty_task")

my_consumer_dag()
```

После включения (unpause) consumer DAG каждый успешный запуск producer task создаёт запуск consumer DAG.

## Расписания по ассетам

- Список ассетов `schedule=[Asset("a"), Asset("b")]` — DAG запускается, когда **все** указанные ассеты получили хотя бы одно обновление.
- Условия: `schedule=(Asset("a") | Asset("b"))` (OR) и `&` (AND), скобки обязательны.
- **AssetOrTimeSchedule** — комбинация времени и ассетов: DAG запускается по расписанию или при обновлении ассетов.

Обновить ассет можно: успешным завершением задачи с этим ассетом в outlets, POST в REST API, вручную в UI (Materialize или Manual), через AssetWatcher (очередь сообщений). Подробнее: [Assets and data-aware scheduling](https://www.astronomer.io/docs/learn/airflow-datasets).

---

[← Airflow UI](airflow-ui.md) | [К содержанию](README.md) | [Планирование →](scheduling.md)
