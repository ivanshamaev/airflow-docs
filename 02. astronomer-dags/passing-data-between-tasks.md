# Передача данных между задачами в Airflow

> Эта страница ещё не обновлена под Airflow 3. Описанные концепции актуальны, но часть примеров кода может потребовать правок. При запуске примеров обновляйте импорты и учитывайте возможные breaking changes.
>
> Информация

Обмен данными между задачами — типичная задача в Airflow. При разбиении DAG на небольшие задачи (рекомендуется для отладки и быстрого восстановления после сбоев) часто нужно передать метаданные вышестоящей задачи или её результат в следующую.

Есть несколько способов передавать данные между задачами. В этом руководстве разобраны два самых распространённых: когда их использовать и как реализовать на примере DAG.

> Вебинар: [How to pass data between your Airflow tasks](https://www.astronomer.io/events/webinars/how-to-pass-data-between-your-airflow-tasks/).
> Astronomer Academy: модуль [Airflow: XComs 101](https://academy.astronomer.io/path/airflow-101/astro-runtime-xcoms-101).
>
> Дополнительные материалы по теме см. в разделе «Other ways to learn».

## Необходимая база

Полезно понимать:

- Лучшие практики написания DAG. См. [Лучшие практики написания DAG](dag-best-practices.md).
- Операторы Airflow. См. [Что такое оператор?](../01.%20astronomer-basic/operators.md).

## Рекомендации

Перед деталями реализации стоит учесть два момента при проектировании DAG, которые передают данные между задачами.

### Идемпотентность

Важное свойство любого пайплайна, в том числе DAG в Airflow, — [идемпотентность](https://en.wikipedia.org/wiki/Idempotence): повторное выполнение операции даёт тот же результат. Это обычно относят ко всему DAG: один и тот же DAG run при повторном запуске даёт тот же результат. То же применимо и к отдельным задачам: если каждая задача идемпотентна, и весь DAG идемпотентен.

При передаче данных между задачами важно обеспечивать идемпотентность каждой задачи. Это упрощает восстановление и снижает риск потери данных при сбоях.

### Учёт размера данных

От размера передаваемых данных зависит выбор способа реализации. XCom — один из способов, но подходит только для небольших объёмов. Для больших данных нужен другой подход: промежуточное хранилище и при необходимости внешний фреймворк обработки.

## XCom

Первый способ передавать данные между задачами — **XCom**, встроенный в Airflow механизм обмена данными задач.

### Что такое XCom

[XCom](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/xcoms.html) — встроенная возможность Airflow. XCom позволяют задачам обмениваться метаданными или небольшими данными. Запись XCom задаётся ключом, значением и временной меткой.

XCom можно «пушить» (отправлять из задачи) или «пуллить» (получать в задаче). При push значение сохраняется в метаданных Airflow и становится доступным остальным задачам. Любое значение, возвращаемое задачей (например, из callable у [PythonOperator](https://registry.astronomer.io/providers/apache-airflow/modules/pythonoperator)), автоматически пушится в XCom. Задачи также могут явно пушить XCom через метод `xcom_push()`. Для получения используется `xcom_pull()`.

Просмотр XCom в UI: **Admin → XComs**. Там отображаются ключи, значения и связанные задачи.

### Когда использовать XCom

XCom стоит использовать для передачи небольших данных: метаданные задач, даты, метрики модели, одиночные результаты запросов и т.п.

Технически через XCom можно передавать и большие объёмы, но делать это не рекомендуется без [кастомного XCom backend](https://www.astronomer.io/docs/learn/custom-xcom-backend-strategies) и [масштабирования воркеров](https://www.astronomer.io/docs/learn/airflow-scaling-workers).

При стандартном XCom backend лимит размера задаётся метаданной БД, например:

- MySQL: 64 KB
- SQLite: 2 GB
- Postgres: 1 GB

Если данные могут превысить эти лимиты, используйте кастомный XCom backend или [промежуточное хранилище](#intermediary-data-storage).

Второе ограничение стандартного backend — сериализуются не все типы данных.

По умолчанию поддерживается сериализация для:

- [Apache Iceberg](https://iceberg.apache.org/) (Airflow 2.8+)
- [Delta Lake](https://delta.io/) (Airflow 2.8+)
- [pandas DataFrame](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.html) (Airflow 2.6+)
- [JSON](https://www.json.org/json-en.html)

Для других типов нужен [кастомный XCom backend](https://www.astronomer.io/docs/learn/custom-xcom-backend-strategies).

### Кастомные XCom backend

При [кастомном XCom backend](https://www.astronomer.io/docs/learn/custom-xcom-backend-strategies) XCom можно пушить и пуллить во внешнюю систему (S3, GCS, HDFS) вместо БД метаданных Airflow. Можно также задать свою логику сериализации и десериализации. Пошаговый пример для Amazon S3, Google Cloud Storage и Azure Blob Storage: [туториал](https://www.astronomer.io/docs/learn/custom-xcom-backends-tutorial).

### Пример DAG с XCom

Ниже — DAG, в котором XCom передаёт данные между задачами. Первая задача запрашивает [API с фактами о котах](http://catfact.ninja/fact) и из ответа берёт поле `fact`. Вторая получает этот результат и выполняет анализ. Для XCom это подходящий случай: между задачами передаётся короткая строка.

В [TaskFlow API](https://airflow.apache.org/docs/apache-airflow/stable/tutorial_taskflow_api.html) значение пушится в XCom при возврате из задачи (как и у традиционных операторов). Чтобы получить значение, передайте объект вышестоящей задачи как аргумент нижестоящей.

С TaskFlow API обычно нужно меньше кода для передачи данных между задачами.

```python
from airflow.decorators import dag, task
from pendulum import datetime

import requests
import json

url = "http://catfact.ninja/fact"

default_args = {"start_date": datetime(2021, 1, 1)}


@dag(schedule="@daily", default_args=default_args, catchup=False)
def xcom_taskflow_dag():
    @task
    def get_a_cat_fact():
        """
        Gets a cat fact from the CatFacts API
        """
        res = requests.get(url)
        return {"cat_fact": json.loads(res.text)["fact"]}

    @task
    def print_the_cat_fact(cat_fact: str):
        """
        Prints the cat fact
        """
        print("Cat fact for today:", cat_fact)
        # run some further cat analysis here

    print_the_cat_fact(get_a_cat_fact())


xcom_taskflow_dag()
```

Ниже тот же DAG в традиционном синтаксисе: два `PythonOperator` обмениваются данными через `xcom_push` и `xcom_pull`. В `get_a_cat_fact` используется `xcom_push`, чтобы задать имя ключа. Можно было бы просто вернуть значение из функции — возвращаемое значение оператора автоматически пушится в XCom.

В `analyze_cat_facts` в `xcom_pull` указываются `key` и `task_ids` нужного XCom. Так можно получить любое значение (или несколько) в любой задаче, не только из непосредственно предыдущей.

```python
import json
from pendulum import datetime, duration
import requests
from airflow import DAG
from airflow.operators.python import PythonOperator


def get_a_cat_fact(ti):
    """
    Gets a cat fact from the CatFacts API
    """
    url = "http://catfact.ninja/fact"
    res = requests.get(url)
    ti.xcom_push(key="cat_fact", value=json.loads(res.text)["fact"])


def analyze_cat_facts(ti):
    """
    Prints the cat fact
    """
    cat_fact = ti.xcom_pull(key="cat_fact", task_ids="get_a_cat_fact")
    print("Cat fact for today:", cat_fact)
    # run some analysis here


with DAG(
    "xcom_dag",
    start_date=datetime(2021, 1, 1),
    max_active_runs=2,
    schedule=duration(minutes=30),
    default_args={"retries": 1, "retry_delay": duration(minutes=5)},
    catchup=False,
) as dag:
    get_cat_data = PythonOperator(
        task_id="get_a_cat_fact", python_callable=get_a_cat_fact
    )

    analyze_cat_data = PythonOperator(
        task_id="analyze_data", python_callable=analyze_cat_facts
    )

    get_cat_data >> analyze_cat_data
```

После запуска DAG на странице XCom в UI появится запись для задачи `get_a_cat_fact` с ключом `cat_fact` и значением из API.

В логах задачи `analyze_data` будет видно выведенное значение из предыдущей задачи — оно успешно получено из XCom.

## Промежуточное хранилище {#intermediary-data-storage}

XCom не требует внешних систем, но рассчитан только на небольшие объёмы. Если нужно передать данные побольше (например, небольшой датафрейм), удобнее использовать промежуточное хранилище.

Идея: одна задача сохраняет данные во внешнюю систему, следующая читает их оттуда. Часто используют облачное хранилище (S3, GCS, Azure Blob Storage) или временную/постоянную таблицу в БД.

Так можно передавать данные, которые не помещаются или не подходят для XCom. При этом Airflow остаётся оркестратором, а не движком обработки: для очень больших данных лучше использовать Spark, Snowflake, dbt и т.п.

### Пример DAG

Продолжая пример с фактами о котах: нужно получить несколько фактов и обработать их. Для XCom это уже не идеально, но раз результат — небольшой датафрейм, обработку можно оставить в Airflow.

```python
from pendulum import datetime, duration
from io import StringIO

import pandas as pd
import requests
from airflow.decorators import dag, task
from airflow.providers.amazon.aws.hooks.s3 import S3Hook

S3_CONN_ID = "aws_conn"
BUCKET = "myexamplebucketone"


@task
def upload_to_s3(cat_fact_number):
    s3_hook = S3Hook(aws_conn_id=S3_CONN_ID)

    url = "http://catfact.ninja/fact"

    res = requests.get(url).json()

    res_df = pd.DataFrame.from_dict([res])
    res_csv = res_df.to_csv()

    s3_hook.load_string(
        res_csv,
        "cat_fact_{0}.csv".format(cat_fact_number),
        bucket_name=BUCKET,
        replace=True,
    )


@task
def process_data(cat_fact_number):
    """Reads data from S3, processes, and saves to new S3 file"""
    s3_hook = S3Hook(aws_conn_id=S3_CONN_ID)

    data = StringIO(
        s3_hook.read_key(
            key="cat_fact_{0}.csv".format(cat_fact_number), bucket_name=BUCKET
        )
    )
    df = pd.read_csv(data, sep=",")

    processed_data = df[["fact"]]
    print(processed_data)

    s3_hook.load_string(
        processed_data.to_csv(),
        "cat_fact_{0}_processed.csv".format(cat_fact_number),
        bucket_name=BUCKET,
        replace=True,
    )


@dag(
    start_date=datetime(2021, 1, 1),
    max_active_runs=1,
    schedule="@daily",
    default_args={"retries": 1, "retry_delay": duration(minutes=1)},
    catchup=False,
)
def intermediary_data_storage_dag():
    upload_to_s3(cat_fact_number=1) >> process_data(cat_fact_number=1)


intermediary_data_storage_dag()
```

```python
from pendulum import datetime, duration
from io import StringIO

import pandas as pd
import requests
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.providers.amazon.aws.hooks.s3 import S3Hook

S3_CONN_ID = "aws_conn"
BUCKET = "myexamplebucketone"


def upload_to_s3(cat_fact_number):
    s3_hook = S3Hook(aws_conn_id=S3_CONN_ID)

    url = "http://catfact.ninja/fact"

    res = requests.get(url).json()

    res_df = pd.DataFrame.from_dict([res])
    res_csv = res_df.to_csv()

    s3_hook.load_string(
        res_csv,
        "cat_fact_{0}.csv".format(cat_fact_number),
        bucket_name=BUCKET,
        replace=True,
    )


def process_data(cat_fact_number):
    """Reads data from S3, processes, and saves to new S3 file"""
    s3_hook = S3Hook(aws_conn_id=S3_CONN_ID)

    data = StringIO(
        s3_hook.read_key(
            key="cat_fact_{0}.csv".format(cat_fact_number), bucket_name=BUCKET
        )
    )
    df = pd.read_csv(data, sep=",")

    processed_data = df[["fact"]]
    print(processed_data)

    s3_hook.load_string(
        processed_data.to_csv(),
        "cat_fact_{0}_processed.csv".format(cat_fact_number),
        bucket_name=BUCKET,
        replace=True,
    )


with DAG(
    "intermediary_data_storage_dag",
    start_date=datetime(2021, 1, 1),
    max_active_runs=1,
    schedule="@daily",
    default_args={"retries": 1, "retry_delay": duration(minutes=1)},
    catchup=False,
) as dag:
    generate_file_task = PythonOperator(
        task_id="generate_file",
        python_callable=upload_to_s3,
        op_kwargs={"cat_fact_number": 1},
    )

    process_data_task = PythonOperator(
        task_id="process_data",
        python_callable=process_data,
        op_kwargs={"cat_fact_number": 1},
    )

    generate_file_task >> process_data_task
```

В этом DAG задача `generate_file` с помощью [S3Hook](https://registry.astronomer.io/providers/amazon/modules/s3hook) сохраняет ответ API в CSV в S3. Задача `process_data` читает данные из S3, превращает их в датафрейм, обрабатывает и сохраняет результат в новый CSV в S3.

---

[← Шаблонирование](jinja-templating.md) | [К содержанию](README.md) | [Контекст →](airflow-context.md)
