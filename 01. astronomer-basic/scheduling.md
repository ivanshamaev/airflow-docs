# Планирование DAG в Apache Airflow®

Airflow поддерживает разные способы планирования: от cron до [data-aware](assets.md) и [event-driven](https://www.astronomer.io/docs/learn/airflow-event-driven-scheduling). Ниже — метки времени DAG run, параметры планирования и варианты расписаний.

## Метки времени DAG run

На странице DAG run отображаются:

- **Logical Date** — момент времени, после которого этот DAG run может быть запущен; основная «дата» запуска в UI.
- **Run after** — после какого момента запуск разрешён; при заданном logical date совпадает с ним, иначе выставляется при триггере.
- **Start / Start Date** — фактическое время начала DAG run (не путать с параметром DAG `start_date`).
- **End / End Date** — время окончания.
- **Duration / Run Duration** — длительность.
- **Run ID** — уникальный идентификатор (тип запуска + logical date или run after + суффикс).
- **Last Scheduling Decision**, **Queued at** — служебные метки планировщика.

При использовании **CronDataIntervalTimetable** имеют смысл **Data Interval Start** и **Data Interval End** (интервал данных для этого запуска). Подробнее: [Timetable comparisons](https://airflow.apache.org/docs/apache-airflow/stable/authoring-and-scheduling/timetable.html#timetables-comparisons).

## Параметры DAG для планирования

- **start_date** — после какого момента DAG может запускаться. Для CronDataIntervalTimetable — начало первого интервала. По умолчанию None.
- **schedule** — правила планирования: cron, timedelta, timetable, список ассетов. По умолчанию None.
- **end_date** — после этой даты DAG не планируется. По умолчанию None.
- **catchup** — заполнять ли пропущенные запуски между start_date и текущей датой (по умолчанию False). Запуски за прошлое можно создавать и вручную (backfill).

Не задавайте динамическое расписание (например, `datetime.now()`) — это приведёт к ошибке планировщика.

## Временные расписания (time-based)

- **Cron-выражение** — строка, например `'5 4 * * *'` (ежедневно в 4:05). Подсказки: [crontab guru](https://crontab.guru/).
- **Cron-пресеты** — например `@daily`, `@hourly`. Список: [Cron Presets](https://airflow.apache.org/docs/apache-airflow/stable/authoring-and-scheduling/cron.html#cron-presets).
- **timedelta** (или `duration` из pendulum) — интервал: `timedelta(minutes=30)`, `timedelta(days=1)`.

Cron передаётся в timetable (по умолчанию CronTriggerTimetable). Опция конфига `[scheduler].create_cron_data_intervals` переключает на CronDataIntervalTimetable.

Ограничения cron: разное время в разные дни, исключение праздников, несколько неравномерных запусков в день — для таких случаев нужны [timetables](https://airflow.apache.org/docs/apache-airflow/stable/howto/timetable.html).

## Data-aware планирование (ассеты)

В **schedule** можно передать один или несколько [ассетов](assets.md). DAG будет запускаться при обновлении этих ассетов (задачи с соответствующими outlets, REST API или вручную в UI). Поддерживаются условия (OR/AND) и комбинация с временем через **AssetOrTimeSchedule**. Подробнее: [Assets](assets.md).

## Timetables

За временные расписания отвечают timetables (CronTriggerTimetable, CronDataIntervalTimetable и др.). Для нестандартных правил можно реализовать [свой timetable](https://airflow.apache.org/docs/apache-airflow/stable/howto/timetable.html).

**Continuous timetable:** `schedule="@continuous"` и `max_active_runs=1` — один непрерывный DAG run: следующий стартует сразу после завершения предыдущего (успешного или нет). Удобно для сенсоров и отложенных операторов при нерегулярных событиях. Airflow не рассчитан на стриминг и очень частые (чаще минуты) запуски; для этого комбинируют с системами вроде Apache Kafka.

---

[← DAG](dags.md) | [К содержанию](README.md) | [Ассеты →](assets.md)
