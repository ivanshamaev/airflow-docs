# Справочник по шаблонам

В шаблонах можно использовать переменные, макросы и фильтры (см. раздел [Jinja Templating](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/operators.html#concepts-jinja-templating)).

Ниже перечислено то, что доступно в Airflow «из коробки». Дополнительные пользовательские макросы можно добавить глобально через [Plugins](https://airflow.apache.org/docs/apache-airflow/stable/administration-and-deployment/plugins.html) или на уровне DAG через аргумент `DAG.user_defined_macros`.

## Переменные

Планировщик Airflow передаёт по умолчанию несколько переменных, доступных во всех шаблонах.

| Переменная | Тип | Описание |
|------------|-----|----------|
| `{{ data_interval_start }}` | [pendulum.DateTime](https://pendulum.eustace.io/docs/#introduction) | Начало интервала данных. Добавлено в версии 2.2. |
| `{{ data_interval_end }}` | [pendulum.DateTime](https://pendulum.eustace.io/docs/#introduction) | Конец интервала данных. Добавлено в версии 2.2. |
| `{{ logical_date }}` | [pendulum.DateTime](https://pendulum.eustace.io/docs/#introduction) | Дата-время, логически идентифицирующее текущий Dag Run. Не несёт семантики, только идентификатор. Для значений с реальной семантикой (например, срез строк по времени) используйте `data_interval_start` и `data_interval_end`. |
| `{{ exception }}` | None \| str \| Exception \| KeyboardInterrupt | Исключение, возникшее при выполнении экземпляра задачи. |
| `{{ prev_data_interval_start_success }}` | [pendulum.DateTime](https://pendulum.eustace.io/docs/#introduction) \| `None` | Начало интервала данных предыдущего успешного DagRun. Добавлено в версии 2.2. |
| `{{ prev_data_interval_end_success }}` | [pendulum.DateTime](https://pendulum.eustace.io/docs/#introduction) \| `None` | Конец интервала данных предыдущего успешного DagRun. Добавлено в версии 2.2. |
| `{{ prev_start_date_success }}` | [pendulum.DateTime](https://pendulum.eustace.io/docs/#introduction) \| `None` | Дата начала предыдущего успешного DagRun (если есть). |
| `{{ prev_end_date_success }}` | [pendulum.DateTime](https://pendulum.eustace.io/docs/#introduction) \| `None` | Дата окончания предыдущего успешного DagRun (если есть). |
| `{{ start_date }}` | [pendulum.DateTime](https://pendulum.eustace.io/docs/#introduction) | Дата и время начала текущей задачи. |
| `{{ inlets }}` | list | Список объявленных на задаче inlet-ассетов. |
| `{{ inlet_events }}` | dict[str, …] | Доступ к прошлым событиям inlet-ассетов. См. [Assets](https://airflow.apache.org/docs/apache-airflow/stable/authoring-and-scheduling/asset-scheduling.html). Добавлено в версии 2.10. |
| `{{ outlets }}` | list | Список объявленных на задаче outlet-ассетов. |
| `{{ outlet_events }}` | dict[str, …] | Доступоры для прикрепления информации к событиям ассетов, которые будет генерировать текущая задача. См. [Assets](https://airflow.apache.org/docs/apache-airflow/stable/authoring-and-scheduling/asset-scheduling.html). Добавлено в версии 2.10. |
| `{{ dag }}` | DAG | Текущий выполняемый DAG. Подробнее: [Dags](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/dags.html). |
| `{{ task }}` | BaseOperator | Текущий выполняемый BaseOperator. Подробнее: [Operators](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/operators.html). |
| `{{ task_reschedule_count }}` | int | Сколько раз текущая задача переносилась по расписанию. Актуально для сенсоров с `mode="reschedule"`. |
| `{{ macros }}` | — | Ссылка на пакет макросов. См. раздел «Макросы» ниже. |
| `{{ task_instance }}` | TaskInstance | Текущий выполняемый TaskInstance. |
| `{{ ti }}` | TaskInstance | То же, что `{{ task_instance }}`. |
| `{{ params }}` | dict[str, Any] | Пользовательские params. Могут переопределяться отображением, переданным в `trigger_dag -c`, если в airflow.cfg включено `dag_run_conf_overrides_params`. |
| `{{ var.value }}` | — | Переменные Airflow. См. «Переменные Airflow в шаблонах» ниже. |
| `{{ var.json }}` | — | Переменные Airflow (JSON). См. «Переменные Airflow в шаблонах» ниже. |
| `{{ conn }}` | — | Подключения Airflow. См. «Подключения Airflow в шаблонах» ниже. |
| `{{ task_instance_key_str }}` | str | Уникальный читаемый ключ экземпляра задачи. Формат: `{dag_id}__{task_id}__{ds_nodash}`. |
| `{{ run_id }}` | str | Идентификатор запуска (run ID) текущего DagRun. |
| `{{ dag_run }}` | DagRun | Текущий выполняемый DagRun. |
| `{{ test_mode }}` | bool | Запущена ли задача через CLI `airflow test`. |
| `{{ map_index_template }}` | None \| str | Шаблон для отображения развёрнутого экземпляра mapped-задачи. Установка этого значения отражается в отрендеренном результате. |
| `{{ expanded_ti_count }}` | int \| `None` | Число экземпляров задачи, на которые развернулась mapped-задача. Если задача не mapped — `None`. Добавлено в версии 2.5. |
| `{{ triggering_asset_events }}` | dict[str, list[AssetEvent]] | В Asset Scheduled DAG — отображение URI ассета в список вызывающих AssetEvent (их может быть несколько при разных частотах ассетов). Подробнее: [Assets](https://airflow.apache.org/docs/apache-airflow/stable/authoring-and-scheduling/asset-scheduling.html). Добавлено в версии 2.4. |

Следующие переменные доступны только когда у DagRun задана `logical_date`:

| Переменная | Тип | Описание |
|------------|-----|----------|
| `{{ ds }}` | str | Логическая дата Dag Run в формате `YYYY-MM-DD`. Эквивалент `{{ logical_date \| ds }}`. |
| `{{ ds_nodash }}` | str | То же, что `{{ logical_date \| ds_nodash }}`. |
| `{{ ts }}` | str | То же, что `{{ logical_date \| ts }}`. Пример: `2018-01-01T00:00:00+00:00`. |
| `{{ ts_nodash_with_tz }}` | str | То же, что `{{ logical_date \| ts_nodash_with_tz }}`. Пример: `20180101T000000+0000`. |
| `{{ ts_nodash }}` | str | То же, что `{{ logical_date \| ts_nodash }}`. Пример: `20180101T000000`. |

Note

Логическая дата Dag Run и производные от неё значения (`ds`, `ts` и т.д.) не следует считать уникальными в рамках DAG. Для уникальности используйте `run_id`.

## Доступ к переменным контекста Airflow из задач TaskFlow

Задачи, оформленные декоратором `@task`, не поддерживают рендеринг Jinja в аргументах, но все перечисленные выше переменные доступны в коде задачи напрямую. Пример доступа к объекту `task_instance` из задачи:

```python
from airflow.models.taskinstance import TaskInstance
from airflow.models.dagrun import DagRun

@task
def print_ti_info(task_instance: TaskInstance, dag_run: DagRun):
    print(f"Run ID: {task_instance.run_id}")  # Run ID: scheduled__2023-08-09T00:00:00+00:00
    print(f"Duration: {task_instance.duration}")  # Duration: 0.972019
    print(f"Dag Run queued at: {dag_run.queued_at}")  # 2023-08-10 00:00:01+02:20
```

К атрибутам и методам объектов можно обращаться через точку: `{{ task.owner }}`, `{{ task.task_id }}`, `{{ ti.hostname }}` и т.д. Подробнее см. в документации по моделям.

## Переменные Airflow в шаблонах

Шаблонная переменная `var` даёт доступ к переменным Airflow. Можно обращаться к ним как к обычному тексту или как к JSON. Для JSON можно обращаться к вложенным структурам, например: `{{ var.json.my_dict_var.key1 }}`.

Получить переменную по строковому ключу (например, если ключ содержит точки) можно так: `{{ var.value.get('my.var', 'fallback') }}` или `{{ var.json.get('my.dict.var', {'key1': 'val1'}) }}`. Можно указывать значение по умолчанию на случай отсутствия переменной.

## Подключения Airflow в шаблонах

Данные подключений Airflow доступны через шаблонную переменную `conn`. Примеры: `{{ conn.my_conn_id.login }}`, `{{ conn.my_conn_id.password }}` и т.д.

Как и для `var`, подключение можно получить по строке (например, `{{ conn.get('my_conn_id_'+index).host }}`) или задать значение по умолчанию (например, `{{ conn.get('my_conn_id', {"host": "host1", "login": "user1"}).host }}`).

Дополнительное поле подключения `extras` доступно как словарь через поле `extra_dejson`, например `conn.my_aws_conn_id.extra_dejson.region_name` вернёт `region_name` из extras. С default тоже можно: `{{ conn.my_aws_conn_id.extra_dejson.get('region_name', 'Europe (Frankfurt)') }}`.

## Фильтры

В Airflow определены Jinja-фильтры для форматирования значений. Например, `{{ logical_date | ds }}` выведет logical_date в формате `YYYY-MM-DD`.

| Фильтр | Применяется к | Описание |
|--------|----------------|----------|
| `ds` | datetime | Формат `YYYY-MM-DD`. |
| `ds_nodash` | datetime | Формат `YYYYMMDD`. |
| `ts` | datetime | Как `.isoformat()`. Пример: `2018-01-01T00:00:00+00:00`. |
| `ts_nodash` | datetime | Как `ts`, но без `-`, `:` и без информации о часовом поясе. Пример: `20180101T000000`. |
| `ts_nodash_with_tz` | datetime | Как `ts`, но без `-` и `:`. Пример: `20180101T000000+0000`. |

## Макросы

Макросы предоставляют объекты в шаблонах и находятся в пространстве имён `macros`.

Доступны распространённые библиотеки и методы:

| Переменная | Описание |
|------------|----------|
| `macros.datetime` | [datetime.datetime](https://docs.python.org/3/library/datetime.html#datetime.datetime) из стандартной библиотеки. |
| `macros.timedelta` | [datetime.timedelta](https://docs.python.org/3/library/datetime.html#datetime.timedelta) из стандартной библиотеки. |
| `macros.dateutil` | Ссылка на пакет `dateutil`. |
| `macros.time` | [time](https://docs.python.org/3/library/time.html#module-time) из стандартной библиотеки. |
| `macros.uuid` | [uuid](https://docs.python.org/3/library/uuid.html#module-uuid) из стандартной библиотеки. |
| `macros.random` | `random.random` из стандартной библиотеки. |

Также определены макросы Airflow:

**datetime_diff_for_humans(dt, since=None)** — [исходный код](https://airflow.apache.org/docs/apache-airflow/stable/_modules/airflow/sdk/execution_time/macros.html#datetime_diff_for_humans)  
Возвращает человекопонятную/приближённую разницу между датами. Если передана одна дата, сравнение идёт с текущим моментом.

- *Параметры:* `dt` — дата, для которой показывается разница; `since` — от какой даты считать (если `None`, то разница между `dt` и текущим временем).

**ds_add(ds, days)** — [исходный код](https://airflow.apache.org/docs/apache-airflow/stable/_modules/airflow/sdk/execution_time/macros.html#ds_add)  
Добавляет или вычитает дни из даты в формате YYYY-MM-DD.

- *Параметры:* `ds` — опорная дата в формате `YYYY-MM-DD`; `days` — количество дней (можно отрицательное).

```python
>>> ds_add("2015-01-01", 5)
'2015-01-06'
>>> ds_add("2015-01-06", -5)
'2015-01-01'
```

**ds_format(ds, input_format, output_format)** — [исходный код](https://airflow.apache.org/docs/apache-airflow/stable/_modules/airflow/sdk/execution_time/macros.html#ds_format)  
Выводит строку даты в заданном формате.

- *Параметры:* `ds` — входная строка с датой; `input_format` — формат входной строки (например, `'%Y-%m-%d'`); `output_format` — формат выходной строки (например, `'%Y-%m-%d'`).

```python
>>> ds_format("2015-01-01", "%Y-%m-%d", "%m-%d-%y")
'01-01-15'
>>> ds_format("1/5/2015", "%m/%d/%Y", "%Y-%m-%d")
'2015-01-05'
>>> ds_format("12/07/2024", "%d/%m/%Y", "%A %d %B %Y", "en_US")
'Friday 12 July 2024'
```

**ds_format_locale(ds, input_format, output_format, locale=None)** — [исходный код](https://airflow.apache.org/docs/apache-airflow/stable/_modules/airflow/sdk/execution_time/macros.html#ds_format_locale)  
Выводит локализованную строку даты в формате Babel.

- *Параметры:* `ds` — входная строка с датой; `input_format` — формат входной строки (например, `'%Y-%m-%d'`); `output_format` — формат вывода в стиле Babel (например, `yyyy-MM-dd`); `locale` — локаль для вывода (например, `'en_US'`). Если не указана, используется LC_TIME по умолчанию, при её отсутствии — `'en_US'`.

```python
>>> ds_format("2015-01-01", "%Y-%m-%d", "MM-dd-yy")
'01-01-15'
>>> ds_format("1/5/2015", "%m/%d/%Y", "yyyy-MM-dd")
'2015-01-05'
>>> ds_format("12/07/2024", "%d/%m/%Y", "EEEE dd MMMM yyyy", "en_US")
'Friday 12 July 2024'
```

Добавлено в версии 2.10.0.

**random()** — возвращает x из интервала [0, 1).

---

*Источник: [Templates reference](https://airflow.apache.org/docs/apache-airflow/stable/templates-ref.html). Перевод неофициальный.*
