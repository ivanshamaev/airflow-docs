# Планирование DAG в Apache Airflow®

Одна из базовых возможностей [Apache Airflow®](https://airflow.apache.org/) — планирование DAG. В Airflow доступно много вариантов: от простых расписаний по cron до [data-aware scheduling](https://www.astronomer.io/docs/learn/scheduling-in-airflow#data-aware-scheduling) с ассетами и event-driven планирования по сообщениям в очереди.

В этом руководстве вы узнаете:

- Какие варианты планирования DAG доступны.
- Какие параметры DAG управляют планированием.
- Как интерпретировать метки времени, связанные с DAG run.

> **Инфо.** Это руководство даёт обзор вариантов планирования. Подробнее по отдельным типам:
> - [Повторный запуск DAG и задач](https://www.astronomer.io/docs/learn/rerunning-dags) (включая backfill)
> - [Event-driven планирование](https://www.astronomer.io/docs/learn/airflow-event-driven-scheduling)
> - [Ассеты и data-aware планирование](https://www.astronomer.io/docs/learn/airflow-datasets)

## Необходимая база

Чтобы получить максимум от руководства, нужно понимать:

- Модули даты и времени в Python. См. [документацию Python по `datetime`](https://docs.python.org/3/library/datetime.html) и [документацию `pendulum`](https://pendulum.eustace.io/docs/).
- DAG в Airflow. См. [Введение в DAG Airflow](dags.md).
- Основные концепции Airflow. См. [Введение в Apache Airflow](https://www.astronomer.io/docs/learn/intro-to-airflow).

## Метки времени DAG run

DAG run — это один запуск DAG, привязанный к моменту времени. На странице деталей DAG run отображаются разные метки времени.

- **Queued at**: момент постановки первой task instance этого DAG run в очередь.
- **Last Scheduling Decision**: последний момент, когда планировщик пытался запланировать task instances для этого DAG run.
- **Run ID**: уникальный идентификатор DAG run. Run ID складывается из типа запуска (например, `scheduled`) и логической даты. Если логическая дата `None`, используется run after date с добавлением случайного суффикса для уникальности. Run ID используется для идентификации DAG run в метаданных Airflow.
- **Duration** и **Run Duration**: время выполнения DAG run, разница между временем начала и окончания.
- **End** и **End Date**: момент окончания DAG run. Эта метка не связана с [параметром DAG](https://www.astronomer.io/docs/learn/scheduling-in-airflow#scheduling-dag-parameters) `end_date`.
- **Start** и **Start Date**: момент фактического начала DAG run. Эта метка не связана с [параметром DAG](https://www.astronomer.io/docs/learn/scheduling-in-airflow#scheduling-dag-parameters) `start_date`.
- **Run after**: момент времени, после которого этот DAG run может быть запущен. Если задана логическая дата, run after совпадает с ней. Если логическая дата `None`, run after устанавливается в текущее время в момент триггера DAG run.
- **Logical Date**: момент времени, после которого этот DAG run может быть запущен. В Airflow UI эта метка отображается как основная дата DAG run. Логическую дату можно явно задать как `None` при запуске DAG через REST API или UI.

Две дополнительные метки имеют смысл только при использовании **CronDataIntervalTimetable**:

- **Data Interval End**: при использовании CronDataIntervalTimetable data interval end DAG run совпадает с run after текущего запланированного DAG run. Если логическая дата `None`, data interval end тоже `None`.
- **Data Interval Start**: при использовании CronDataIntervalTimetable data interval start DAG run совпадает с run after предыдущего запланированного DAG run того же DAG. При других расписаниях data interval start совпадает с run after текущего DAG run. Если DAG run запущен с логической датой `None`, data interval start тоже `None`.

Подробнее об интервалах данных и отличиях CronDataIntervalTimetable от CronTriggerTimetable: [документация Airflow](https://airflow.apache.org/docs/apache-airflow/stable/authoring-and-scheduling/timetable.html#timetables-comparisons).

## Параметры DAG для планирования

Следующие параметры задают, когда DAG будет выполняться:

- **catchup**: булево значение; нужно ли автоматически создавать все запуски между `start_date` и текущей датой. По умолчанию `False`. Помимо этого можно вручную запускать DAG за любую дату в прошлом. См. [Backfill](https://www.astronomer.io/docs/learn/rerunning-dags#backfill).
- **end_date**: дата, после которой DAG больше не планируется. По умолчанию `None`.
- **schedule**: правила, по которым создаются DAG run. Принимает cron-выражения, объекты timedelta, timetables и списки ассетов. По умолчанию `None`.
- **start_date**: момент времени, после которого DAG может запускаться. При использовании CronDataIntervalTimetable `start_date` — момент, после которого может начаться первый интервал данных. По умолчанию `None`.

В примере ниже DAG задан с `start_date` 1 апреля 2025, расписанием `@daily` и `end_date` 1 апреля 2026. DAG будет запускаться каждый день в полночь UTC с 1 апреля 2025 по 31 марта 2026. Пропущенные запуски автоматически не восполняются (catchup по умолчанию False).

```python
from pendulum import datetime
from airflow.sdk import dag

@dag(
    start_date=datetime(2025, 4, 1),
    schedule="@daily",
    end_date=datetime(2026, 4, 1),
)
def my_dag():
    pass

my_dag()
```

> **Внимание.** Не задавайте динамическое расписание (например, `datetime.now()`)! Это приведёт к ошибке планировщика.

## Временные расписания (time-based)

Для пайплайнов с простыми требованиями к расписанию параметр `schedule` DAG можно задать с помощью:

- объекта **timedelta**;
- **cron-пресета**;
- **cron-выражения**.

Cron-выражения под капотом передаются в timetable. По умолчанию используется **CronTriggerTimetable**. Опция конфигурации [`[scheduler].create_cron_data_intervals`](https://airflow.apache.org/docs/apache-airflow/stable/configurations-ref.html#create-cron-data-intervals) переключает на **CronDataIntervalTimetable** (поведение в более ранних версиях Airflow). Подробнее: [Timetable comparisons](https://airflow.apache.org/docs/apache-airflow/stable/authoring-and-scheduling/timetable.html#timetables-comparisons).

### Cron-выражения

В параметр `schedule` DAG можно передать любое cron-выражение в виде строки. Например, для запуска каждый день в 4:05 утра: `schedule='5 4 * * *'`.

Подсказки по составлению cron-выражений: [crontab guru](https://crontab.guru/).

### Cron-пресеты

В Airflow можно использовать пресеты для типичных расписаний. Например, `schedule='@hourly'` запускает DAG в начале каждого часа. Полный список пресетов: [Cron Presets](https://airflow.apache.org/docs/apache-airflow/stable/authoring-and-scheduling/cron.html#cron-presets).

### Объекты timedelta

Если нужно запускать DAG с заданным интервалом (каждый час, каждые 5 минут и т.д.), а не в фиксированное время, в `schedule` можно передать объект **timedelta** из [пакета `datetime`](https://docs.python.org/3/library/datetime.html) или **duration** из [пакета `pendulum`](https://pendulum.eustace.io/docs/). Например, `schedule=timedelta(minutes=30)` — каждые 30 минут, `schedule=timedelta(days=1)` — раз в день.

### Ограничения расписаний на основе cron

Расписания на основе cron плохо подходят для нерегулярных временных правил, например:

- несколько запусков в день с неравными интервалами (например, 13:00 и 16:30);
- запуск каждый день кроме праздников;
- разное время в разные дни (например, 14:00 по четвергам и 16:00 по субботам).

Такие расписания можно реализовать с помощью [timetables](https://www.astronomer.io/docs/learn/scheduling-in-airflow#timetables).

## Data-aware планирование (ассеты)

С помощью ассетов Airflow может учитывать обновления данных и планировать другие DAG при обновлении этих ассетов. Чтобы задать расписание по ассетам, передайте имя (имена) ассета в параметр `schedule`. Можно задавать условия по нескольким ассетам (OR/AND) и комбинировать расписание по ассетам с временным.

**Список ассетов (условие AND)** — DAG запускается, когда обновлены все указанные ассеты:

```python
from airflow.sdk import Asset

my_asset_1 = Asset("my_asset_1")
my_asset_2 = Asset("my_asset_2")

@dag(
    schedule=[my_asset_1, my_asset_2],  # список задаёт условие AND
)
def my_dag():
    pass

my_dag()
```

Этот DAG запускается, когда и `my_asset_1`, и `my_asset_2` обновлены хотя бы один раз.

**Условие OR** — DAG запускается при обновлении любого из ассетов:

```python
from airflow.sdk import Asset

my_asset_1 = Asset("my_asset_1")
my_asset_2 = Asset("my_asset_2")

@dag(
    schedule=(my_asset_1 | my_asset_2),  # () вместо [] для условного расписания
)
def my_dag():
    pass

my_dag()
```

Этот DAG запускается, когда обновлён `my_asset_1` или `my_asset_2`.

**Комбинация с временем (AssetOrTimeSchedule)** — DAG по расписанию и по ассетам:

```python
from airflow.sdk import Asset
from airflow.timetables.assets import AssetOrTimeSchedule
from airflow.timetables.trigger import CronTriggerTimetable

my_asset_1 = Asset("my_asset_1")
my_asset_2 = Asset("my_asset_2")

@dag(
    schedule=AssetOrTimeSchedule(
        timetable=CronTriggerTimetable("0 0 * * *", timezone="UTC"),
        assets=(my_asset_1 | my_asset_2),
    ),
)
def my_dag():
    pass

my_dag()
```

Этот DAG запускается каждый день в полночь UTC и дополнительно при обновлении `my_asset_1` или `my_asset_2`.

Ассеты могут обновляться задачами любого DAG в том же окружении Airflow, вызовами [эндпоинта ассетов REST API](https://airflow.apache.org/docs/apache-airflow/stable/stable-rest-api-ref.html) или вручную в Airflow UI.

Подробнее об ассетах и data-driven планировании: [Ассеты и data-aware планирование в Airflow](assets.md). Ассеты можно комбинировать с AssetWatchers для event-driven расписаний. См. [Event-Driven Scheduling](https://www.astronomer.io/docs/learn/airflow-event-driven-scheduling).

## Timetables

Каждое временное расписание в Airflow реализовано через timetable. Есть встроенные timetables, в том числе **CronTriggerTimetable** и **CronDataIntervalTimetable**. Если под вашу задачу нет готового timetable, можно [реализовать свой](https://airflow.apache.org/docs/apache-airflow/stable/howto/timetable.html).

### Continuous timetable

DAG можно запускать непрерывно с заданным timetable. Чтобы использовать [ContinuousTimetable](https://airflow.apache.org/docs/apache-airflow/stable/_api/airflow/timetables/simple/index.html#module-airflow.timetables.simple.ContinuousTimetable), задайте расписание `"@continuous"` и `max_active_runs=1`.

```python
from pendulum import datetime
from airflow.sdk import dag

@dag(
    start_date=datetime(2025, 4, 1),
    schedule="@continuous",
    max_active_runs=1,
)
def my_dag():
    pass
```

При таком расписании создаётся один непрерывный DAG run: следующий запуск начинается сразу после завершения предыдущего (успешного или нет). ContinuousTimetable особенно удобен при использовании [сенсоров](sensors.md) или [deferrable-операторов](https://www.astronomer.io/docs/learn/deferrable-operators) для ожидания нерегулярных событий во внешних системах.

> **Внимание.** Airflow рассчитан на оркестрацию пайплайнов пакетами (batch), а не на стриминг и низкую задержку. Для запусков чаще чем раз в минуту лучше комбинировать Airflow с инструментами вроде [Apache Kafka](https://www.astronomer.io/docs/learn/airflow-kafka).
---

[← DAG](dags.md) | [К содержанию](README.md) | [Зависимости задач →](task-dependencies.md)
