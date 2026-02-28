# Ассеты и data-aware планирование в Airflow

С помощью **ассетов (Assets)** DAG, работающие с одними и теми же данными, получают явные, видимые связи, а DAG можно планировать по **обновлениям этих ассетов**. Это делает Airflow «осведомлённым о данных» и расширяет возможности планирования за пределы чисто временных методов вроде cron.

Ассеты помогают решать типичные задачи. Например: у команды инженеров данных есть DAG, создающий ассет, у ML-команды — DAG, обучающий модель по этому ассету. С ассетами DAG ML-команды запускается только после того, как DAG инженеров данных обновил ассет.

В этом руководстве вы узнаете:

- Как прикреплять данные к событиям ассетов и получать их из событий.
- Как использовать алиасы ассетов для динамических расписаний по ассетам.
- Как запускать DAG по простым и продвинутым расписаниям на основе ассетов.
- Как объявлять задачи Airflow производителями (producers) ассетов.
- Как использовать синтаксис `@asset` для data-oriented пайплайнов.
- Когда уместно использовать ассеты в Airflow.

> Ассеты можно использовать для планирования DAG по сообщениям в очереди (event-driven scheduling). Подробнее: [Schedule DAGs based on Events in a Message Queue](https://www.astronomer.io/docs/learn/airflow-event-driven-scheduling).

> **Инфо.** Ассеты — отдельная возможность от object storage, который позволяет работать с файлами в облачном и локальном хранилищах. См. [Use Airflow object storage to interact with cloud storage in an ML pipeline](https://www.astronomer.io/docs/learn/airflow-object-storage-tutorial).

## Необходимая база

Чтобы получить максимум от руководства, нужно понимать:

- Основы планирования в Airflow. См. [Schedule DAGs in Airflow](scheduling.md).

## Синтаксис `@asset`

Декоратор `@asset` — краткий способ создать один DAG с одной задачей, которая производит ассет. Он используется в asset-oriented подходе к написанию DAG: данные (ассет) ставятся в центр. Выбор между asset-oriented и task-oriented подходом — вопрос предпочтений. DAG, созданные через `@asset`, отображаются в Airflow UI как обычные DAG.

В примере ниже задаётся DAG с ID `my_asset`, расписание `@daily`, и одна задача с task ID `my_asset`, которая при успешном завершении обновляет ассет с именем `my_asset`.

```python
from airflow.sdk import asset

@asset(schedule="@daily")
def my_asset():
    # ваша логика задачи
    pass
```

Расписание ассета можно задать по другим ассетам, чтобы строить data-centric пайплайны. Так как каждый `@asset` создаёт один DAG, данные между DAG передаются через cross-DAG XCom. Ниже — один и тот же простой ETL: вариант с asset-oriented и с task-oriented подходом.

```python
from airflow.sdk import asset


@asset(schedule="@daily")
def extracted_data():
    return {"a": 1, "b": 2}


@asset(schedule=extracted_data)
def transformed_data(context):

    data = context["ti"].xcom_pull(
        dag_id="extracted_data",
        task_ids="extracted_data",
        key="return_value",
        include_prior_dates=True,
    )
    return {k: v * 2 for k, v in data.items()}


@asset(schedule=transformed_data)
def loaded_data(context):

    data = context["task_instance"].xcom_pull(
        dag_id="transformed_data",
        task_ids="transformed_data",
        key="return_value",
        include_prior_dates=True,
    )
    summed_data = sum(data.values())
    print(f"Summed data: {summed_data}")
```

```python
from airflow.sdk import Asset, dag, task


@dag(schedule="@daily")
def extract_dag():

    @task(outlets=[Asset("extracted_data")])
    def extract_task():
        return {"a": 1, "b": 2}

    extract_task()


extract_dag()


@dag(schedule=[Asset("extracted_data")])
def transform_dag():

    @task(outlets=[Asset("transformed_data")])
    def transform_task(**context):
        data = context["ti"].xcom_pull(
            dag_id="extract_dag",
            task_ids="extract_task",
            key="return_value",
            include_prior_dates=True,
        )
        return {k: v * 2 for k, v in data.items()}

    transform_task()


transform_dag()


@dag(schedule=[Asset("transformed_data")])
def load_dag():

    @task
    def load_task(**context):
        data = context["ti"].xcom_pull(
            dag_id="transform_dag",
            task_ids="transform_task",
            key="return_value",
            include_prior_dates=True,
        )
        summed_data = sum(data.values())
        print(f"Summed data: {summed_data}")

    load_task()


load_dag()
```

Код выше создаёт 3 зависимых друг от друга DAG; в каждом одна задача обновляет один ассет.

Чтобы обновлять несколько ассетов из одного DAG в asset-oriented подходе, используется `@asset.multi`. Пример ниже создаёт один DAG с ID `my_multi_asset`, одну задачу `my_multi_asset`, которая при успешном завершении обновляет два ассета: `asset_a` и `asset_b`.

```python
from airflow.sdk import Asset, asset

@asset.multi(schedule="@daily", outlets=[Asset("asset_a"), Asset("asset_b")])
def my_multi_asset():
    pass
```

## Концепции ассетов

Помимо декоратора `@asset`, ассеты можно задавать в обычном (task-oriented) коде DAG и использовать для кросс-DAG и даже кросс-деплой зависимостей, отражающих поток конкретных или абстрактных данных. В этом разделе — терминология и общие принципы использования.

### Терминология ассетов

Ассеты задаются в коде DAG и используются для кросс-DAG зависимостей. В Airflow используются такие термины:

- **AssetWatcher** — класс для [event-driven scheduling](https://www.astronomer.io/docs/learn/airflow-event-driven-scheduling): ожидание `TriggerEvent` от сообщения в очереди.
- **Metadata** — класс для прикрепления к ассету поля `extra` из producer task. См. [Прикрепление информации к событию ассета](#прикрепление-информации-к-событию-ассета).
- **AssetAlias** — объект, связанный с одним или несколькими ассетами; используется для расписаний по ассетам, созданным в runtime. См. [Использование алиасов ассетов](#использование-алиасов-ассетов). Задаётся уникальным именем.
- **Queued asset event** — DAG часто планируют так, чтобы он запускался, как только каждый из набора ассетов получил хотя бы одно обновление. Пока не хватает событий для запуска DAG, все уже пришедшие события по остальным ассетам считаются queued. Queued asset event задаётся ассетом, временной меткой и DAG, для которого он в очереди. Одно событие ассета может породить queued events для нескольких DAG. Доступ к ним — через [Airflow REST API](https://airflow.apache.org/docs/apache-airflow/stable/stable-rest-api-ref.html#operation/get_dag_dataset_queued_event).
- **Asset expression** — логическое выражение с операторами AND (`&`) и OR (`|`) для расписания DAG по обновлениям нескольких ассетов.
- **Producer task** — задача, которая обновляет один или несколько ассетов из параметра `outlets` и при успешном завершении создаёт события ассетов.
- **Asset schedule** — расписание DAG по событиям одного или нескольких ассетов. Все ассеты, по которым запланирован DAG, отображаются в графе DAG в UI и на вкладке Assets.
- **Asset event** — событие, привязанное к ассету при обновлении его producer task. Задаётся ассетом и временной меткой; опционально — словарь `extra`.
- **@asset** — декоратор для asset-oriented DAG: один DAG с одной задачей, обновляющей ассет. Подробнее в разделе [Синтаксис @asset](#синтаксис-asset).
- **Asset** — объект в Airflow, представляющий конкретную или абстрактную сущность данных; задаётся уникальным именем. Опционально к ассету можно привязать URI (файл, таблица).

У всех операторов и декораторов Airflow есть два параметра, связанных с ассетами:

- **Inlets** — список ассетов, к событиям которых задача имеет доступ (например, к `extra`). Inlets не влияют на расписание DAG и не отображаются в UI.
- **Outlets** — список ассетов, которые задача обновляет при успешном завершении. Outlets отображаются в графе DAG и на вкладке Assets сразу после разбора кода DAG (независимо от того, были ли события). Airflow не проверяет сами данные; решать, какая задача считается producer для ассета, нужно вам. Задача с outlet считается producer даже если не работает с данным ассетом.

Итого: задачи обновляют ассеты из `outlets` и создают события ассетов. DAG можно планировать по событиям одного или нескольких ассетов; задаче можно дать доступ к событиям ассета через `inlets`. Ассет появляется в метаданных Airflow, как только на него ссылаются в `outlets` задачи или в `schedule` DAG.

### Зачем использовать ассеты в Airflow

Ассеты задают явные зависимости между DAG и обновлениями данных. Это позволяет:

- Планировать DAG по сообщениям в очереди через [event-driven scheduling](https://www.astronomer.io/docs/learn/airflow-event-driven-scheduling).
- Строить сложные data-driven расписания с [условным расписанием по ассетам](#условное-расписание-по-ассетам) и [комбинированным расписанием по ассетам и времени](#комбинированное-расписание-по-ассетам-и-времени).
- Создавать кросс-деплой зависимости через [Airflow REST API](https://airflow.apache.org/docs/apache-airflow/stable/stable-rest-api-ref.html#operation/create_dataset_event). Клиенты [Astro](https://www.astronomer.io/lp/signup/?referral=docs-what-astro-banner) могут ориентироваться на [Cross-deployment dependencies](https://www.astronomer.io/docs/astro/best-practices/cross-deployment-dependencies).
- Снижать затраты: ассеты не занимают слот воркера в отличие от сенсоров и [других реализаций кросс-DAG зависимостей](../02.%20astronomer-dags/cross-dag-dependencies.md).
- Улучшать видимость связей DAG и зависимостей от данных: графы Assets в UI показывают зависимости ассетов и DAG.
- Уменьшать объём кода для [кросс-DAG зависимостей](../02.%20astronomer-dags/cross-dag-dependencies.md). Даже если DAG не зависит от обновлений данных, можно задать зависимость: DAG запускается после обновления ассета задачей другого DAG.
- Стандартизировать обмен между командами: ассеты работают как API — сигнализируют, что данные в заданном месте обновлены и готовы к использованию.

### Работа с ассетами

При использовании ассетов учитывайте:

- Вкладка **Assets** в [Airflow UI](airflow-ui.md) показывает список активных ассетов и граф зависимостей каждого ассета с DAG и другими ассетами.
- Airflow учитывает ассеты только в контексте DAG и задач. Обновления ассетов вне Airflow (например, ручное добавление файла в S3) не отслеживаются. Для зависимостей от внешних событий используйте [сенсоры](sensors.md), [deferrable-операторы](https://www.astronomer.io/docs/learn/deferrable-operators) или [event-driven scheduling](https://www.astronomer.io/docs/learn/airflow-event-driven-scheduling) через очередь сообщений.
- События ассетов регистрируются только DAG или слушателями в том же окружении Airflow. Для кросс-деплой зависимостей нужно создавать события через [Airflow REST API](https://airflow.apache.org/docs/apache-airflow/stable/stable-rest-api-ref.html) в окружении, где находится consumer DAG. Пример: [Cross-deployment dependencies](https://www.astronomer.io/docs/astro/best-practices/cross-deployment-dependencies#datasets-example).

> Примеры см. в [Create Airflow listeners tutorial](https://www.astronomer.io/docs/learn/airflow-listeners). Listeners — продвинутая возможность; они выполняются внутри компонентов Airflow и могут замедлить или нарушить работу экземпляра. Asset Event listeners — [экспериментальная возможность](https://airflow.apache.org/docs/apache-airflow/stable/release-process.html#experimental). Доступны хуки: `on_asset_updated`, `on_asset_alias_created`, `on_asset_created`. Подробнее: [listeners](https://airflow.apache.org/docs/apache-airflow/stable/administration-and-deployment/listeners.html#listeners).

## Определение ассета

Ассет появляется в метаданных Airflow, как только на него ссылаются в `outlets` задачи или в `schedule` DAG. Про алиасы см. [Использование алиасов ассетов](#использование-алиасов-ассетов).

### Базовое определение ассета

Простейший вариант: один DAG планируется по обновлениям одного ассета, который производит одна задача. В примере задача `my_producer_task` в DAG `my_producer_dag` обновляет ассет `my_asset`, создавая события; DAG `my_consumer_dag` запускается по каждому такому событию.

Сначала укажите ассет в параметре `outlets` producer task.

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

```python
from airflow.sdk import Asset, DAG
from airflow.providers.standard.operators.python import PythonOperator

with DAG(dag_id="my_producer_dag"):

    def my_function():
        pass

    my_task = PythonOperator(
        task_id="my_producer_task",
        python_callable=my_function,
        outlets=[Asset("my_asset")]
    )
```

Связь DAG producer с ассетом видна в графе Assets на вкладке Assets в UI. В графе `my_producer_dag` ассет тоже отображается, если в Options включены external conditions или all DAG dependencies.

Затем запланируйте `my_consumer_dag` на запуск при появлении нового события ассета `my_asset`.

```python
from airflow.sdk import Asset, dag
from airflow.providers.standard.operators.empty import EmptyOperator

@dag(
    schedule=[Asset("my_asset")],
)
def my_consumer_dag():

    EmptyOperator(task_id="empty_task")

my_consumer_dag()
```

```python
from airflow.sdk import Asset, DAG
from airflow.operators.empty import EmptyOperator

with DAG(
    dag_id="my_consumer_dag",
    schedule=[Asset("my_asset")]
):

    EmptyOperator(task_id="empty_task")
```

Связь producer DAG, consumer DAG и ассета видна в графе на вкладке Assets. В графе `my_consumer_dag` ассет отображается при включённых external conditions или all DAG dependencies.

После включения (unpause) `my_consumer_dag` каждое успешное завершение `my_producer_task` запускает один run `my_consumer_dag`.

На странице деталей producer task перечислены вызванные им Asset Events со ссылкой на Triggered Dag Run. В запуске consumer DAG тоже указано событие ассета со ссылкой на исходный DAG.

### Использование алиасов ассетов

Алиасы ассетов позволяют планировать DAG по ассетам с именами, сформированными в runtime. Алиас задаётся уникальной строкой `name` и может использоваться вместо обычного ассета в `outlets` и `schedule`. К одному алиасу можно привязать события разных ассетов.

Добавить событие к алиасу можно двумя способами:

- Через `outlet_events` из контекста Airflow.
- Через класс `Metadata`.

В примерах ниже имя ассета определяется в runtime внутри producer task.

```python
# from airflow.sdk import Asset, AssetAlias, Metadata, task

my_alias_name = "my_alias"

@task(outlets=[AssetAlias(my_alias_name)])
def attach_event_to_alias_metadata():
    bucket_name = "my-bucket"  # определяется в runtime
    yield Metadata(
        asset=Asset(f"updated_{bucket_name}"),
        extra={"k": "v"},  # extra обязателен, может быть {}
        alias=AssetAlias(my_alias_name),
    )

attach_event_to_alias_metadata()
```

```python
# from airflow.sdk import Asset, AssetAlias, Metadata, task

my_alias_name = "my_alias"

@task(outlets=[AssetAlias(my_alias_name)])
def attach_event_to_alias_context(outlet_events):  # outlet_events из контекста
    bucket_name = "my-other-bucket"
    outlet_events[AssetAlias(my_alias_name)].add(
        Asset(f"updated_{bucket_name}"), extra={"k": "v"}
    )  # extra опционален

attach_event_to_alias_context()
```

В consumer DAG вместо обычного ассета можно указать алиас.

```python
from airflow.sdk import AssetAlias, dag
from airflow.providers.standard.operators.empty import EmptyOperator

my_alias_name = "my_alias"

@dag(schedule=[AssetAlias(my_alias_name)])
def my_consumer_dag():

    EmptyOperator(task_id="empty_task")

my_consumer_dag()
```

После успешного завершения DAG с задачей `attach_event_to_alias_metadata` автоматически запускается переразбор всех DAG, запланированных на алиас `my_alias`. При переразборе ассет `updated_{bucket_name}` привязывается к алиасу и расписание разрешается — запускается один run `my_consumer_dag`.

Любое следующее событие ассета `updated_{bucket_name}` будет запускать `my_consumer_dag`. Если к одному алиасу привязаны события нескольких ассетов, DAG на этом алиасе запускается, как только любой из когда-либо привязанных к алиасу ассетов получит обновление.

Подробнее и ещё примеры: [Dynamic data events emitting and asset creation through AssetAlias](https://airflow.apache.org/docs/apache-airflow/stable/authoring-and-scheduling/datasets.html#dynamic-data-events-emitting-and-dataset-creation-through-datasetalias).

Для традиционных операторов событие к алиасу нужно добавлять внутри логики оператора: в методе `.execute` кастомного оператора или через callable `post_execute` ([экспериментально](https://airflow.apache.org/docs/apache-airflow/stable/release-process.html#experimental-features)). Используйте `outlet_events`. Для [deferrable-операторов](https://www.astronomer.io/docs/learn/deferrable-operators) прикрепление события к алиасу поддерживается только в `execute_complete` или `post_execute`.

```python
def _attach_event_to_alias(context, result):  # result — возвращаемое значение execute
    uri = "s3://my-bucket/my_file.txt"
    context["outlet_events"][AssetAlias(my_alias_name)].add(Asset(uri))

BashOperator(
    task_id="t2",
    bash_command="echo hi",
    outlets=[AssetAlias(my_alias_name)],
    post_execute=_attach_event_to_alias,  # post_execute — экспериментальный параметр
)
```

## Обновление ассета

Обновить ассет можно пятью способами:

1. Успешное завершение DAG, определённого через `@asset` (под капотом один DAG с одной задачей-producer).
2. Успешное завершение задачи с параметром `outlets`, ссылающимся на этот ассет.
3. `POST`-запрос к [эндпоинту ассетов Airflow REST API](https://airflow.apache.org/docs/apache-airflow/stable/stable-rest-api-ref.html#tag/Dataset).
4. **AssetWatcher**, слушающий `TriggerEvent` от сообщения в очереди. См. [event-driven scheduling](https://www.astronomer.io/docs/learn/airflow-event-driven-scheduling).
5. Ручное обновление в Airflow UI кнопкой **Create Asset Event** на графе ассета. Два варианта:
   - **Materialize** — запускается полный DAG с задачей, производящей это событие ассета.
   - **Manual** — создаётся новое событие ассета без выполнения задачи. Удобно для тестов или когда ассет не обновляется из DAG в этом экземпляре Airflow.

![Снимок экрана: ручное обновление ассета в Airflow UI.](https://files.buildwithfern.com/astronomer.docs.buildwithfern.com/docs/d9ab962b5300c2c847febbaf9fff5c905083b63a0054201917c451f138be47bf/docs/assets/img/guides/3-0_airflow-datasets_manually_update_dataset.png)

### Прикрепление информации к событию ассета

При обновлении ассета в UI или через `POST` в REST API к событию можно прикрепить дополнительные данные, передав JSON `extra`. Из producer task то же самое делается через класс `Metadata` или через `outlet_events` из контекста. В `extra` можно передать любую вычисленную в задаче информацию.

Пример с классом `Metadata`. Ассет в metadata должен совпадать с одним из outlets задачи.

```python
# from airflow.sdk import Asset, Metadata, task

my_asset_1 = Asset("x-asset1")

@task(outlets=[my_asset_1])
def attach_extra_using_metadata():
    num = 23
    yield Metadata(my_asset_1, {"myNum": num})

    return "hello :)"

attach_extra_using_metadata()
```

Добавить `extra` к событию ассета можно и через `outlet_events` из контекста.

```python
from airflow.sdk import Asset, Metadata, task

my_asset_2 = Asset("x-asset2")

@task(outlets=[my_asset_2])
def use_outlet_events(outlet_events):  # outlet_events из контекста
    num = 19
    outlet_events[my_asset_2].extra = {"my_num": num}

    return "hello :)"

use_outlet_events()
```

Поле `extra` отображается в UI в графе ассета.

### Получение информации об ассете в downstream-задаче

Данные из `extra` можно получать в задачах программно. У любого экземпляра задачи в DAG run есть доступ к списку событий ассетов, запустивших этот run (`triggering_asset_events`). Задаче можно также дать доступ ко всем событиям конкретного ассета через параметр `inlets`. Inlets не влияют на расписание DAG.

В задаче TaskFlow API `triggering_asset_events` берётся из [контекста Airflow](../02.%20astronomer-dags/airflow-context.md). В традиционном операторе — [Jinja-шаблонирование](../02.%20astronomer-dags/jinja-templating.md) в любом шаблонируемом поле.

```python
# from airflow.sdk import task

@task
def get_extra_triggering_run(triggering_asset_events):
    # triggering_asset_events — все события, запустившие этот DAG run (из контекста)
    # цикл не выполнится при ручном запуске DAG
    for asset, asset_list in triggering_asset_events.items():
        print(asset, asset_list)
        print(asset_list[0].extra)
        print(asset_list[0].source_run_id)  # run_id и др. по upstream DAG
```

```python
# from airflow.operators.bash import BashOperator

get_extra_triggering_run_bash = BashOperator(
    task_id="get_extra_triggering_run_bash",
    # Ошибка, если нет triggering events (например при ручном запуске)!
    bash_command="echo {{ (triggering_asset_events.values() | first | first).extra }} ",
)
```

Чтобы получать `extra` независимо от того, какие события запустили DAG, укажите ассет в `inlets` задачи. В TaskFlow API `inlet_events` берётся из [контекста Airflow](../02.%20astronomer-dags/airflow-context.md).

```python
# from airflow.sdk import Asset, task

my_asset_2 = Asset("x-asset2")

# my_asset_2 не обязан входить в schedule DAG; inlets может быть несколько
@task(inlets=[my_asset_2])
def get_extra_inlet(inlet_events):  # inlet_events из контекста
    # события перечислены от ранних к поздним по timestamp
    asset_events = inlet_events[my_asset_2]
    if len(asset_events) == 0:
        print(f"No asset_events for {my_asset_2.uri}")
    else:
        my_extra = asset_events[-1].extra  # последнее событие; если extra нет — None
        print(my_extra)

get_extra_inlet()
```

Программное получение данных из алиасов ассетов: [Fetching information from previously emitted asset events through resolved asset aliases](https://airflow.apache.org/docs/apache-airflow/stable/authoring-and-scheduling/datasets.html#fetching-information-from-previously-emitted-dataset-events-through-resolved-dataset-aliases).

## Расписания по ассетам

В параметр `schedule` можно передать любое число ассетов. Есть три типа расписаний по ассетам:

- **AssetOrTimeSchedule** — сочетание расписания по времени и выражений по ассетам. См. [комбинированное расписание по ассетам и времени](#комбинированное-расписание-по-ассетам-и-времени).
- **schedule=(Asset("a") | Asset("b"))** — операторы AND (`&`) и OR (`|`) для [условного выражения по ассетам](#условное-расписание-по-ассетам). Выражения берутся в круглые скобки `()`.
- **schedule=[Asset("a"), Asset("b")]** — один или несколько ассетов списком. DAG запускается, когда **все** ассеты из списка получили хотя бы одно обновление.

Важно:

- У DAG, запускаемых по ассетам, нет концепции data interval. Информацию о запустившем событии можно брать из `triggering_asset_events` в контексте (поля: timestamp, source_dag_id, source_task_id, source_run_id, source_map_index). См. [Получение информации в downstream-задаче](#получение-информации-об-ассете-в-downstream-задаче).
- При паузе consumer DAG все обновления ассетов за время паузы игнорируются; после unpause очередь пуста.
- Consumer DAG по нескольким ассетам запускается, как только выражение выполняется (хотя бы по одному событию на каждый ассет). Дополнительные обновления одного ассета не создают отдельные запуски; все queued события по одному ассету потребляются одним запуском. Подробнее: [Multiple Assets](https://airflow.apache.org/docs/apache-airflow/stable/authoring-and-scheduling/datasets.html#multiple-datasets).
- Consumer DAG по ассету запускается, как только завершается **первая** задача с этим ассетом в outlets, даже если в DAG есть ещё downstream producer по тому же ассету.
- Consumer DAG по ассету запускается при **каждом** успешном завершении задачи, обновляющей этот ассет. Если `task1` и `task2` оба производят `asset_a`, consumer DAG `asset_a` запустится дважды — после `task1` и после `task2`.

### Условное расписание по ассетам

В `schedule` можно комбинировать ассеты логическими операторами: `|` (OR) и `&` (AND).

Пример: DAG запускается при обновлении любого из `asset1`–`asset4`. Вся конструкция в круглых скобках `()`.

```python
from airflow.sdk import Asset, dag

@dag(
    schedule=(
        Asset("asset1")
        | Asset("asset2")
        | Asset("asset3")
        | Asset("asset4")
    ),  # () вместо [] для условного расписания!
)
def downstream1_on_any():

    # ваши задачи

downstream1_on_any()
```

Более сложный пример: DAG запускается при обновлении (asset1 или asset2) **и** (asset3 или asset4).

```python
from airflow.sdk import Asset, dag

@dag(
    schedule=(
        (Asset("asset1") | Asset("asset2"))
        & (Asset("asset3") | Asset("asset4"))
    ),
)
def downstream2_one_in_each_group():

    # ваши задачи

downstream2_one_in_each_group()
```

### Комбинированное расписание по ассетам и времени

Расписание по ассетам можно сочетать с временным через timetable **AssetOrTimeSchedule**. DAG запускается, когда выполняется условие `timetable` **или** условие по `asset`.

Пример: DAG по cron `0 0 * * *` (каждый день в полночь) и при обновлении `asset3` или `asset4`.

```python
from airflow.sdk import Asset, dag, task
from pendulum import datetime
from airflow.timetables.assets import AssetOrTimeSchedule
from airflow.timetables.trigger import CronTriggerTimetable

@dag(
    start_date=datetime(2025, 3, 1),
    schedule=AssetOrTimeSchedule(
        timetable=CronTriggerTimetable("0 0 * * *", timezone="UTC"),
        assets=(Asset("asset3") | Asset("asset4")),
    )
)
def toy_downstream3_asset_and_time_schedule():

    # ваши задачи

toy_downstream3_asset_and_time_schedule()
```

```python
from airflow.sdk import Asset, DAG
from pendulum import datetime
from airflow.timetables.assets import AssetOrTimeSchedule
from airflow.timetables.trigger import CronTriggerTimetable

with DAG(
    dag_id="toy_downstream3_asset_and_time_schedule",
    start_date=datetime(2024, 3, 1),
    schedule=AssetOrTimeSchedule(
        timetable=CronTriggerTimetable("0 0 * * *", timezone="UTC"),
        assets=(Asset("asset3") | Asset("asset4")),
    )
):
    # ваши задачи
```

### Пример реализации

В примере ниже `write_instructions_to_file` и `write_info_to_file` — producer tasks (у них заданы outlets).

```python
from airflow.sdk import Asset, dag, task

API = "https://www.thecocktaildb.com/api/json/v1/1/random.php"
INSTRUCTIONS = Asset("file://localhost/airflow/include/cocktail_instructions.txt")
INFO = Asset("file://localhost/airflow/include/cocktail_info.txt")


@dag
def assets_producer_dag():
    @task
    def get_cocktail(api):
        import requests

        r = requests.get(api)
        return r.json()

    @task(outlets=[INSTRUCTIONS])
    def write_instructions_to_file(response):
        cocktail_name = response["drinks"][0]["strDrink"]
        cocktail_instructions = response["drinks"][0]["strInstructions"]
        msg = f"See how to prepare {cocktail_name}: {cocktail_instructions}"

        f = open("include/cocktail_instructions.txt", "a")
        f.write(msg)
        f.close()

    @task(outlets=[INFO])
    def write_info_to_file(response):
        import time

        time.sleep(30)
        cocktail_name = response["drinks"][0]["strDrink"]
        cocktail_category = response["drinks"][0]["strCategory"]
        alcohol = response["drinks"][0]["strAlcoholic"]
        msg = f"{cocktail_name} is a(n) {alcohol} cocktail from category {cocktail_category}."
        f = open("include/cocktail_info.txt", "a")
        f.write(msg)
        f.close()

    cocktail = get_cocktail(api=API)

    write_instructions_to_file(cocktail)
    write_info_to_file(cocktail)


assets_producer_dag()
```

Consumer DAG запускается при обновлении ассетов из его `schedule` producer task, а не по времени. Пример: DAG должен запускаться при обновлении ассетов `INSTRUCTIONS` и `INFO` — они указываются в `schedule`.

DAG считается consumer по ассету, если он запланирован по этому ассету, даже если не обращается к нему в коде. Корректно ссылаться и использовать ассеты — ответственность автора DAG.

```python
from airflow.sdk import Asset, dag, task

INSTRUCTIONS = Asset("file://localhost/airflow/include/cocktail_instructions.txt")
INFO = Asset("file://localhost/airflow/include/cocktail_info.txt")


@dag(schedule=[INSTRUCTIONS, INFO])  # по обоим ассетам
def assets_consumer_dag():
    @task
    def read_about_cocktail():
        cocktail = []
        for filename in ("info", "instructions"):
            with open(f"include/cocktail_{filename}.txt", "r") as f:
                contents = f.readlines()
                cocktail.append(contents)

        return [item for sublist in cocktail for item in sublist]

    read_about_cocktail()


assets_consumer_dag()
```

```python
from airflow.sdk import DAG, Asset
from airflow.providers.standard.operators.python import PythonOperator

INSTRUCTIONS = Asset("file://localhost/airflow/include/cocktail_instructions.txt")
INFO = Asset("file://localhost/airflow/include/cocktail_info.txt")


def read_about_cocktail_func():
    cocktail = []
    for filename in ("info", "instructions"):
        with open(f"include/cocktail_{filename}.txt", "r") as f:
            contents = f.readlines()
            cocktail.append(contents)

    return [item for sublist in cocktail for item in sublist]


with DAG(
    dag_id="assets_consumer_dag",
    schedule=[INSTRUCTIONS, INFO],  # по обоим ассетам
):
    PythonOperator(
        task_id="read_about_cocktail",
        python_callable=read_about_cocktail_func,
    )
```

---

[← Интерфейс Airflow (UI)](airflow-ui.md) | [К содержанию](README.md) | [Планирование →](scheduling.md)
