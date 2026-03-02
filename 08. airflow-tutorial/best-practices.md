# Лучшие практики

Создание нового DAG — процесс из трёх шагов:

1. написание Python-кода для создания объекта DAG;
2. проверка, что код соответствует вашим ожиданиям;
3. настройка зависимостей окружения для запуска DAG.

В этом руководстве описаны лучшие практики для этих трёх шагов.

## Написание DAG

Создать новый DAG в Airflow несложно. Но нужно учитывать много нюансов, чтобы успешный или неуспешный прогон DAG не давал неожиданных результатов.

### Создание кастомного оператора/хука

Следуйте руководству по [кастомным операторам](https://airflow.apache.org/docs/apache-airflow/stable/howto/custom-operator.html#custom-operator).

### Создание задачи

Задачи в Airflow следует рассматривать как транзакции в базе данных. То есть задача не должна оставлять неполный результат. Например, не записывать неполные данные в HDFS или S3 в конце задачи.

Airflow может повторно запустить задачу при сбое. Поэтому задачи должны давать один и тот же результат при каждом повторном запуске. Как этого добиться:

- **Не используйте INSERT при повторном запуске задачи** — INSERT может привести к дублированию строк. Используйте UPSERT.
- **Читайте и пишите в конкретную партицию.** Не читайте в задаче «последние доступные» данные: между повторными запусками их могут изменить, и результат будет другим. Читайте данные из конкретной партиции. В качестве партиции можно использовать `data_interval_start`. То же правило при записи в S3/HDFS.
- Функция **`now()` из Python datetime** возвращает текущие дату и время. Её не стоит использовать внутри задачи, особенно для важных вычислений — результат будет разным при каждом запуске. Допустимо использовать, например, для временного лога.

Tip

Повторяющиеся параметры (например, `connection_id` или пути S3) лучше задавать в `default_args`, а не в каждой задаче. Так меньше шансов на опечатки. У многих типов подключений параметр в задачах один (например, `gcp_conn_id`), поэтому подключение можно указать один раз в `default_args` — все операторы этого типа будут его использовать.

### Удаление задачи

Осторожно удаляйте задачу из DAG: в Graph View, Grid View и т.п. она исчезнет, и смотреть её логи в веб-интерфейсе будет нельзя. Если это нежелательно, создайте новый DAG.

### Взаимодействие между задачами

При [Kubernetes executor](https://airflow.apache.org/docs/apache-airflow-providers-cncf-kubernetes/stable/kubernetes_executor.html) или [Celery executor](https://airflow.apache.org/docs/apache-airflow-providers-celery/stable/celery_executor.html) задачи одного DAG выполняются на разных серверах. Поэтому не храните файлы и конфиги в локальной файловой системе — следующая задача может запуститься на другом сервере и до них не доберётся (например, задача скачивает файл, а следующая его обрабатывает). При Local executor файлы на диске тоже усложняют повторные запуски (например, если задаче нужен конфиг, который удаляет другая задача DAG).

По возможности используйте **XCom** для коротких сообщений между задачами; для больших объёмов данных — удалённое хранилище (S3/HDFS). Например, задача пишет результат в S3 и кладёт путь в XCom, а следующие задачи читают путь из XCom и по нему загружают данные.

В самих задачах не храните учётные данные (пароли, токены). По возможности используйте [Connections](https://airflow.apache.org/docs/apache-airflow/stable/authoring-and-scheduling/connections.html): храните данные в бэкенде Airflow и обращайтесь по уникальному connection id.

### Код верхнего уровня (Top level Python Code)

Не пишите код верхнего уровня, который не нужен для создания операторов и связей DAG. Это связано с работой планировщика Airflow и с тем, что скорость разбора кода верхнего уровня влияет на производительность и масштабируемость.

Планировщик выполняет код вне методов `execute` операторов не реже чем раз в [min_file_process_interval](https://airflow.apache.org/docs/apache-airflow/stable/configurations-ref.html#config-dag-processor-min-file-process-interval) секунд — чтобы поддерживать динамическое расписание (когда расписание и зависимости со временем меняются). Планировщик постоянно сверяет содержимое DAG с запланированными задачами.

В коде верхнего уровня не должно быть обращений к БД, тяжёлых вычислений и сетевых операций.

На время загрузки DAG сильно влияют импорты верхнего уровня — их часто недооценивают. Долгие импорты дают лишнюю нагрузку; по возможности переносите их внутрь callable (локальные импорты).

Сравните два примера. В первом DAG будет разбираться на 1000 секунд дольше, чем во втором, где `expensive_api_call` вызывается только в контексте задачи.

**Плохо — код верхнего уровня:**

```python
import pendulum
from airflow.sdk import DAG
from airflow.sdk import task

def expensive_api_call():
    print("Hello from Airflow!")
    sleep(1000)

my_expensive_response = expensive_api_call()

with DAG(
    dag_id="example_python_operator",
    schedule=None,
    start_date=pendulum.datetime(2021, 1, 1, tz="UTC"),
    catchup=False,
    tags=["example"],
) as dag:
    @task()
    def print_expensive_api_call():
        print(my_expensive_response)
```

**Хорошо — вызов внутри задачи:**

```python
import pendulum
from airflow.sdk import DAG
from airflow.sdk import task

def expensive_api_call():
    sleep(1000)
    return "Hello from Airflow!"

with DAG(
    dag_id="example_python_operator",
    schedule=None,
    start_date=pendulum.datetime(2021, 1, 1, tz="UTC"),
    catchup=False,
    tags=["example"],
) as dag:
    @task()
    def print_expensive_api_call():
        my_expensive_response = expensive_api_call()
        print(my_expensive_response)
```

В первом примере `expensive_api_call` выполняется при каждом разборе файла DAG и замедляет обработку. Во втором она вызывается только при запуске задачи, и разбор DAG не страдает. Можно проверить: реализуйте первый вариант и увидите «Hello from Airflow!» в логах планировщика.

Note

Импорты тоже считаются кодом верхнего уровня. Долгий импорт или модуль с кодом на верхнем уровне при импорте тоже влияет на планировщик. Тяжёлые импорты лучше делать локально внутри callable:

```python
# Допустимо импортировать лёгкие модули на верхнем уровне
import random
import pendulum

# Тяжёлые импорты не делайте на верхнем уровне
# import pandas
# import torch
# import tensorflow

@task()
def do_stuff_with_pandas_and_torch():
    import pandas
    import torch
    # операции с pandas и torch

@task()
def do_stuff_with_tensorflow():
    import tensorflow
    # операции с tensorflow
```

### Как понять, что код «верхнего уровня»

Нужно учитывать особенности разбора Python. Обычно при разборе файла выполняется весь видимый код, кроме (как правило) тела функций, которые в этот момент не вызываются.

Есть неочевидные случаи: к верхнему уровню относится и код, определяющий значения по умолчанию аргументов функций.

Проверить просто: добавьте в подозрительный код `print` и запустите файл как `python your_dag_file.py`. Если `print` сработал — это код верхнего уровня.

Пример: в коде ниже при разборе вызывается `get_task_id()` (например, для `task_id=get_task_id()`), но не `get_array()` — она передаётся как callable и выполнится только при запуске задачи. При запуске `python test_python.py` выведется только «Executing 1».

```python
def get_task_id():
    print("Executing 1")
    return "print_array_task"  # выполнится при разборе — ДА

def get_array():
    print("Executing 2")
    return [1, 2, 3]  # при разборе не выполнится — НЕТ

with DAG(...) as dag:
    operator = PythonOperator(
        task_id=get_task_id(),
        python_callable=get_array,
        dag=dag,
    )
```

### Качество кода и линтинг

Качество кода важно для надёжности и поддержки пайплайнов. Помогают линтеры, в том числе **ruff** — быстрый линтер для Python с правилами для Airflow.

ruff помогает находить устаревшие конструкции и то, что может помешать переходу на Airflow 3.0. Есть правила с префиксом `AIR`. Полный список: [Airflow (AIR)](https://docs.astral.sh/ruff/rules/#airflow-air).

**Установка:**

```bash
pip install "ruff>=0.14.14"
```

**Запуск проверки DAG:**

```bash
ruff check dags/ --select AIR3
```

Команда проверяет каталог `dags/` и сообщает о проблемах по выбранным правилам.

**Пример:** для legacy-DAG с `@dag()`, `Dataset`, `FileSensor` и т.п. ruff может выдать, например: `AIR301` (нужен явный schedule), `AIR302` (schedule_interval/Dataset удалены в 3.0), `AIR303` (сенсор перенесён в standard provider). Включение ruff в процесс разработки помогает заранее править устаревания и упрощает переход между версиями Airflow.

### Динамическая генерация DAG

Иногда писать DAG вручную неудобно: много похожих DAG с разными параметрами или набор DAG для загрузки таблиц, которые не хочется обновлять вручную при каждом изменении. В таких случаях удобнее генерировать DAG динамически.

Избегание лишней обработки на верхнем уровне (см. выше) особенно важно при динамической конфигурации. Её можно задавать:

- [переменными окружения](https://wiki.archlinux.org/title/environment_variables) (не путать с [Airflow Variables](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/variables.html));
- сгенерированным извне Python-кодом с метаданными в папке DAG;
- сгенерированным извне конфигурационным файлом метаданных в папке DAG.

Подробнее: [Dynamic DAG Generation](https://airflow.apache.org/docs/apache-airflow/stable/howto/dynamic-dag-generation.html).

### Airflow Variables

Использование Airflow Variables приводит к сетевым запросам и обращению к БД, поэтому в коде верхнего уровня DAG их по возможности не используйте (см. раздел про код верхнего уровня). Если переменные всё же нужны на верхнем уровне, можно [включить экспериментальный кэш](https://airflow.apache.org/docs/apache-airflow/stable/configurations-ref.html#config-secrets-use-cache) с разумным [ttl](https://airflow.apache.org/docs/apache-airflow/stable/configurations-ref.html#config-secrets-cache-ttl-seconds).

Переменные можно свободно использовать внутри `execute()` операторов. Также их можно передавать в операторы через Jinja-шаблон — значение будет прочитано только при выполнении задачи.

Синтаксис:

```
{{ var.value.<variable_name> }}
```

или для JSON из переменной:

```
{{ var.json.<variable_name> }}
```

В коде верхнего уровня шаблоны с переменными не вызывают запрос до запуска задачи, а `Variable.get()` без кэша — при каждом разборе DAG планировщиком и может замедлить разбор или привести к таймауту.

**Плохо:**

```python
foo_var = Variable.get("foo")  # ИЗБЕГАТЬ
# или в bash_command f-string, или в env= с Variable.get("foo")
```

**Хорошо:**

```python
bash_use_variable_good = BashOperator(
    task_id="bash_use_variable_good",
    bash_command="echo variable foo=${foo_env}",
    env={"foo_env": "{{ var.value.get('foo') }}"},
)

@task
def my_task():
    var = Variable.get("foo")  # нормально — вызывается только при запуске задачи
    print(var)
```

Для чувствительных данных рекомендуется [Secrets Backend](https://airflow.apache.org/docs/apache-airflow/stable/security/secrets/secrets-backend/index.html#secrets-backend-configuration).

### Timetables

В коде расписания (timetable) на верхнем уровне не используйте Airflow Variables/Connections и не обращайтесь к БД Airflow. Обращение к БД откладывайте до времени выполнения DAG. Не передавайте получение переменных/подключений в аргументы инициализации класса timetable и не используйте Variable/Connection на верхнем уровне модуля с кастомным timetable.

**Плохо:** `def __init__(self, *args, something=Variable.get("something"), **kwargs)`  
**Хорошо:** передавать идентификатор (например, строку), а значение получать внутри при необходимости уже при выполнении.

### Запуск DAG после изменений

Не запускайте DAG сразу после изменения файлов DAG или других файлов в папке DAG.

Дайте системе время обработать изменения: файлы должны попасть к планировщику (через общую ФС или Git-Sync), планировщик должен разобрать Python и сохранить DAG в БД. В зависимости от конфигурации, скорости ФС, числа файлов и DAG, размера изменений, числа планировщиков и CPU это может занять от секунд до минут. Дождитесь появления DAG в UI перед запуском.

При долгих задержках можно настроить параметры (см. ссылки на документацию):

- [scheduler_idle_sleep_time](https://airflow.apache.org/docs/apache-airflow/stable/configurations-ref.html#config-scheduler-scheduler-idle-sleep-time)
- [min_file_process_interval](https://airflow.apache.org/docs/apache-airflow/stable/configurations-ref.html#config-dag-processor-min-file-process-interval)
- [refresh_interval](https://airflow.apache.org/docs/apache-airflow/stable/configurations-ref.html#config-dag-processor-refresh-interval)
- [parsing_processes](https://airflow.apache.org/docs/apache-airflow/stable/configurations-ref.html#config-dag-processor-parsing-processes)
- [file_parsing_sort_mode](https://airflow.apache.org/docs/apache-airflow/stable/configurations-ref.html#config-dag-processor-file-parsing-sort-mode)

### Пример паттерна «наблюдатель» (watcher) с trigger rules

Паттерн «наблюдатель» — DAG с задачей, которая «следит» за состоянием остальных задач. Цель — переводить Dag Run в состояние failed, если любая другая задача упала (например, для системных тестов Airflow, где DAG играет роль набора шагов).

Обычно при падении одной задачи остальные не выполняются и весь Dag Run получает статус failed. Но при использовании trigger rules поток меняется: например, teardown с `TriggerRule.ALL_DONE` выполнится в любом случае, и статус Dag Run будет определяться этой задачей — информация о падении других задач может потеряться. Чтобы DAG с teardown всё равно падал при падении любой задачи, добавляют задачу-наблюдатель (watcher): она всегда падает, если запущена, и должна запускаться только если упала хотя бы одна другая задача. У неё задают `TriggerRule.ONE_FAILED` и делают её downstream для всех остальных задач. Тогда при успехе всех задач watcher пропускается, а при падении любой — выполняется и падает, и Dag Run тоже получает failed.

Note

Trigger rules учитывают только прямых предшественников (родителей). Например, `TriggerRule.ONE_FAILED` не учитывает failed/upstream_failed задачи, которые не являются прямым родителем данной задачи.

Пример DAG с watcher:

```python
from datetime import datetime
from airflow.sdk import DAG, task
from airflow.exceptions import AirflowException
from airflow.providers.standard.operators.bash import BashOperator
from airflow.utils.trigger_rule import TriggerRule

@task(trigger_rule=TriggerRule.ONE_FAILED, retries=0)
def watcher():
    raise AirflowException("Failing task because one or more upstream tasks failed.")

with DAG(
    dag_id="watcher_example",
    schedule="@once",
    start_date=datetime(2021, 1, 1),
    catchup=False,
) as dag:
    failing_task = BashOperator(task_id="failing_task", bash_command="exit 1", retries=0)
    passing_task = BashOperator(task_id="passing_task", bash_command="echo passing_task")
    teardown = BashOperator(
        task_id="teardown",
        bash_command="echo teardown",
        trigger_rule=TriggerRule.ALL_DONE,
    )
    failing_task >> passing_task >> teardown
    list(dag.tasks) >> watcher()
```

Роли задач: `failing_task` всегда падает; `passing_task` при выполнении успешен; `teardown` всегда запускается и должен быть успешен; `watcher` — downstream для всех остальных, срабатывает при падении любой и сам падает (листовая задача). Без watcher этот Dag Run получил бы статус success (листовая задача — teardown — успешна). Чтобы watcher учитывал все задачи, он должен зависеть от каждой из них. Без teardown watcher не нужен: failed от `failing_task` передался бы по цепочке и Dag Run стал бы failed.

### Пропуск отдельных DAG через AirflowClusterPolicySkipDag (с версии 2.7)

DAG обычно разворачивают через определённую ветку Git (например, git-sync). При нескольких кластерах Airflow содержать несколько веток неудобно (cherry-pick трудоёмок, hard-reset не подходит для GitOps). Можно держать несколько кластеров на одной ветке (например, `main`) и различать их переменными окружения и разными Connection с одним и тем же `connection_id`. При необходимости в cluster policy можно вызывать [AirflowClusterPolicySkipDag](https://airflow.apache.org/docs/apache-airflow/stable/_api/airflow/exceptions/index.html#airflow.exceptions.AirflowClusterPolicySkipDag), чтобы не загружать определённые DAG в DagBag на данном кластере.

Пример — не загружать DAG с тегом `only_for_beta` на production:

```python
def dag_policy(dag: DAG):
    if "only_for_beta" in dag.tags:
        raise AirflowClusterPolicySkipDag(
            f"Dag {dag.dag_id} is not loaded on the production cluster, due to `only_for_beta` tag."
        )
```

## Снижение сложности DAG

Airflow хорошо справляется с большим числом DAG и задач, но при высокой сложности это сказывается на планировании. Чтобы система оставалась производительной, упрощайте и оптимизируйте DAG: разбор и создание DAG — это выполнение Python-кода, и от вас зависит, насколько он быстрый. Универсальных метрик «достаточной простоты» нет, но улучшения заметны.

Рекомендации:

1. **Ускорить загрузку DAG** — сильнее всего влияет на планировщик. Используйте советы из раздела про код верхнего уровня и тест загрузки DAG (Dag Loader Test).
2. **Упрощать структуру DAG** — каждая зависимость добавляет нагрузку. Линейная цепочка A → B → C планируется быстрее, чем глубокое дерево с большим числом зависимостей. Более линейные DAG обычно лучше для планирования.
3. **Меньше DAG в одном файле** — в Airflow 2 несколько DAG в файле поддерживаются, но один файл обрабатывается одним FileProcessor, что может замедлять отражение изменений в UI. При многих DAG в одном файле рассмотрите разбиение по файлам.
4. **Писать эффективный Python** — баланс между количеством файлов и объёмом кода. Не копируйте почти одинаковый код в множество файлов (лишние импорты и дубли). Минимизируйте повторения и при необходимости используйте [Dynamic DAG Generation](https://airflow.apache.org/docs/apache-airflow/stable/howto/dynamic-dag-generation.html).

## Тестирование DAG

DAG стоит рассматривать как production-код и сопровождать тестами.

### Dag Loader Test

Проверка, что DAG загружается без ошибок (нет отсутствующих зависимостей, синтаксических ошибок и т.п.). Дополнительный код не нужен — достаточно выполнить:

```bash
python your-dag-file.py
```

Запускайте в окружении, максимально близком к планировщику (те же зависимости, переменные окружения, общий код). Так же можно замерять время загрузки после оптимизаций — например, командой `time` в Linux, несколько раз подряд с учётом кэша. Сравнивайте до и после в одинаковых условиях.

```bash
time python airflow/example_dags/example_python_operator.py
```

Важна метрика «real time». Учтите, что при таком запуске стартует новый интерпретатор — есть начальная задержка. Её можно оценить: `time python -c ''` и вычесть из времени загрузки DAG.

Подробнее о тестировании отдельных операторов: [Testing a Dag](https://airflow.apache.org/docs/apache-airflow/stable/tutorial/fundamentals.html#testing).

### Unit-тесты

Unit-тесты проверяют корректность кода DAG и задач.

Пример: загрузка DAG и проверка структуры через DagBag:

```python
import pytest
from airflow.models import DagBag

@pytest.fixture()
def dagbag():
    return DagBag()

def test_dag_loaded(dagbag):
    dag = dagbag.get_dag(dag_id="hello_world")
    assert dagbag.import_errors == {}
    assert dag is not None
    assert len(dag.tasks) == 1
```

Проверка структуры DAG (например, соответствие словарю task_id → список downstream):

```python
def assert_dag_dict_equal(source, dag):
    assert dag.task_dict.keys() == source.keys()
    for task_id, downstream_list in source.items():
        assert dag.has_task(task_id)
        task = dag.get_task(task_id)
        assert task.downstream_task_ids == set(downstream_list)
```

Пример теста кастомного оператора через `dag.test()` и проверку состояния задачи (и при необходимости XCom).

### Self-Checks

В DAG можно встроить проверки результата задач. Например, после записи в S3 следующая задача может проверить наличие партиции и базовую корректность данных. Для микросервисов в Kubernetes/Mesos можно использовать [HttpSensor](https://airflow.apache.org/docs/apache-airflow-providers-http/stable/_api/airflow/providers/http/sensors/http/index.html#airflow.providers.http.sensors.http.HttpSensor), чтобы убедиться, что сервис поднялся.

Пример: задача записи в S3 и сенсор проверки ключа:

```python
task = PushToS3(...)
check = S3KeySensor(
    task_id="check_parquet_exists",
    bucket_key="s3://bucket/key/foo.parquet",
    poke_interval=0,
    timeout=0,
)
task >> check
```

### Staging-окружение

По возможности держите staging для полного прогона DAG перед выкладкой в production. Параметризуйте DAG (пути S3, БД и т.д.), не хардкодьте значения под среду. Можно использовать переменные окружения:

```python
import os
dest = os.environ.get("MY_DAG_DEST_PATH", "s3://default-target/path/")
```

## Мокирование переменных и подключений

В тестах кода, использующего Variables или Connections, эти объекты должны существовать. Можно сохранять их в БД, но это замедляет тесты. Для ускорения можно имитировать их через переменные окружения: для переменной — [AIRFLOW_VAR_{KEY}](https://airflow.apache.org/docs/apache-airflow/stable/cli-and-env-variables-ref.html#envvar-AIRFLOW_VAR_-KEY), для подключения — [AIRFLOW_CONN_{CONN_ID}](https://airflow.apache.org/docs/apache-airflow/stable/cli-and-env-variables-ref.html#envvar-AIRFLOW_CONN_-CONN_ID), с подстановкой через `unittest.mock.patch.dict("os.environ", ...)`.

## Обслуживание БД метаданных

Со временем БД метаданных растёт из-за прогонов DAG и задач и логов. Для очистки старых данных используйте CLI: `airflow db clean`. Подробнее: [db clean usage](https://airflow.apache.org/docs/apache-airflow/stable/howto/usage-cli.html#cli-db-clean).

## Обновления и откаты

### Резервная копия БД

Перед любыми действиями, меняющими БД, делайте резервную копию метаданных.

### Отключение планировщика

На время обслуживания можно отключить планирование: параметр `[scheduler] > use_job_schedule` = `False` (дождаться завершения текущих прогонов; новые создаваться не будут, кроме ручного запуска). Либо вручную приостановить DAG: сохранить список активных DAG (`airflow dags list`), перед обслуживанием выполнить `airflow dags pause` для каждого, после — `airflow dags unpause` по тому же списку. Так можно сначала включить один-два тестовых DAG и проверить работу после обновления.

### «Интеграционные» тестовые DAG

Полезно иметь один-два DAG, которые используют типичные сервисы (S3, Snowflake, Vault и т.д.) с тестовыми ресурсами/аккаунтами. Их можно включать первыми после обновления: при падении последствия минимальны, при успехе — кластер в целом готов к работе (подключения, KubernetesPodOperator, S3, БД и т.д.).

### Очистка данных перед обновлением

Часть миграций БД может быть долгой. При большом объёме метаданных рассмотрите предварительную очистку старых данных командой [db clean](https://airflow.apache.org/docs/apache-airflow/stable/howto/usage-cli.html#cli-db-clean). Действуйте осторожно.

## Конфликтующие и сложные зависимости Python

У Airflow много зависимостей; иногда они конфликтуют с зависимостями вашего кода задач. По умолчанию одно окружение Python на весь Airflow, и разные задачи могут требовать несовместимые пакеты.

При использовании готовых операторов конфликты встречаются реже — провайдеры и constraints подобраны. При TaskFlow API, кастомном коде в задачах или собственных операторах конфликты возможны и между вашим кодом и Airflow, и между разными вашими операторами.

Варианты решения (от простых к более сложным в развёртывании):

### PythonVirtualenvOperator

Самый простой, но ограниченный вариант. Оператор создаёт виртуальное окружение на время выполнения задачи; в TaskFlow — декоратор `@task.virtualenv`. У каждой задачи может быть свой набор зависимостей.

Плюсы: не нужно заранее готовить venv; разные задачи с разными зависимостями на одних воркерах; авторам DAG не нужна помощь админов; не требуется менять деплой (Local/Docker/Kubernetes); не нужно знать контейнеры. Ограничения: callable должен сериализоваться (pickle/dill); тяжёлые библиотеки импортировать только внутри callable; только Python-зависимости, не системные; накладные расходы на создание venv при каждом запуске; воркеры должны иметь доступ к PyPI/приватным репозиториям; возможны временные сбои при установке; зависимости могут обновиться и сломать задачу или стать риском supply chain; задачи изолированы только разными venv, не процессами. Примеры: [TaskFlow virtualenv](https://airflow.apache.org/docs/apache-airflow/stable/tutorial/taskflow.html#taskflow-dynamically-created-virtualenv).

### ExternalPythonOperator (с версии 2.4)

Требует заранее подготовленное неизменяемое Python-окружение на всех воркерах. В TaskFlow — `@task.external_python`. Новые зависимости в это окружение на лету не добавить.

Плюсы: нет накладных расходов на создание venv при запуске задачи; зависимости можно проверить заранее (безопасность и стабильность); не нужен доступ воркеров к PyPI при выполнении; не нужно переходить на контейнеры. Минусы: окружения готовят и разворачивают заранее (часто админы); callable сериализуемый; тяжёлые импорты только внутри callable; только Python-зависимости; изоляция только за счёт разных окружений. Обычно разработка идёт с `@task.virtualenv`, а в production переходят на `ExternalPythonOperator`/`@task.external_python` после выкладки нужных venv. Примеры: [TaskFlow External Python](https://airflow.apache.org/docs/apache-airflow/stable/tutorial/taskflow.html#taskflow-external-python-environment).

### DockerOperator и KubernetesPodOperator

Требуют доступ к Docker или кластеру Kubernetes. Задачи полностью изолированы, можно использовать любые зависимости и даже другой язык или архитектуру (x86/arm). С версии 2.2 — `@task.docker`, с 2.4 — `@task.kubernetes`.

Плюсы: полная изоляция; образы можно кэшировать; зависимости проверяются до деплоя. Минусы: накладные расходы на запуск контейнера/pod; при декораторах callable сериализуется и передаётся в контейнер (есть ограничения по размеру); нужны два процесса (задача + супервизор в воркере); образы нужно собирать и публиковать заранее; нужны базовые знания Docker/Kubernetes. Примеры: [TaskFlow Docker](https://airflow.apache.org/docs/apache-airflow/stable/tutorial/taskflow.html#taskflow-docker-environment), [TaskFlow Kubernetes](https://airflow.apache.org/docs/apache-airflow/stable/tutorial/taskflow.html#tasfklow-kpo).

### Несколько Docker-образов и очереди Celery

Теоретически можно привязать разные задачи к разным очередям Celery и разным образам воркеров. Это требует глубокой настройки деплоя и даёт большую операционную сложность и меньшую эффективность использования ресурсов. Пока не реализованы [AIP-46 Runtime isolation](https://cwiki.apache.org/confluence/display/AIRFLOW/AIP-46+Runtime+isolation+for+airflow+tasks+and+dag+parsing) и [AIP-43 Dag Processor Separation](https://cwiki.apache.org/confluence/display/AIRFLOW/AIP-43+DAG+Processor+separation), такой подход почти не даёт выигрыша и не рекомендуется. После реализации AIP станет возможной мультитенантная изоляция зависимостей на всём цикле — от разбора DAG до выполнения задач.

---

*Источник: [Best Practices](https://airflow.apache.org/docs/apache-airflow/stable/best-practices.html). Перевод неофициальный.*
