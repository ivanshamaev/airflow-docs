# Cloud-Native Workflows с Object Storage

*Добавлено в версии 2.8.*

Финальный туториал серии. К этому моменту вы уже собирали DAG на Python и TaskFlow API, передавали данные через XCom и связывали задачи в понятные переиспользуемые пайплайны.

В этом туториале добавляется **Object Storage API**: удобная работа с облачным хранилищем (Amazon S3, Google Cloud Storage, Azure Blob Storage) без привязки к SDK конкретного провайдера и без ручного управления учётными данными.

Разберём практический сценарий:

- Получение данных из публичного API
- Сохранение в объектное хранилище в формате Parquet
- Анализ через SQL с DuckDB

По пути покажем абстракцию **ObjectStoragePath**, настройку учётных данных через подключения (connections) Airflow и то, как это даёт переносимые, облачно-независимые пайплайны.

## Зачем это нужно

Многие пайплайны завязаны на файлы: сырые CSV, промежуточные Parquet, артефакты моделей. Раньше под каждый облачный бэкенд писали свой код. С **ObjectStoragePath** можно писать один и тот же код для разных провайдеров — достаточно настроить нужное подключение в Airflow.

## Предварительные требования

Перед началом понадобится:

- **DuckDB** (in-process SQL): `pip install duckdb`
- **Доступ к Amazon S3** и провайдер Amazon с s3fs: `pip install apache-airflow-providers-amazon[s3fs]`  
  (Можно заменить на другой провайдер, поменяв протокол в URL хранилища и установив соответствующий пакет.)
- **Pandas**: `pip install pandas`

## Создание ObjectStoragePath

Основа туториала — **ObjectStoragePath**, абстракция для путей в облачных объектных хранилищах. По смыслу это как `pathlib.Path`, но для бакетов, а не для локальной ФС.

*Источник: [airflow/example_dags/tutorial_objectstorage.py](https://airflow.apache.org/docs/apache-airflow/stable/_modules/airflow/example_dags/tutorial_objectstorage.html)*

```python
base = ObjectStoragePath("s3://aws_default@airflow-tutorial-data/")
```

Синтаксис URL: `protocol://bucket/path/to/file`

- **protocol** (например `s3`, `gs`, `azure`) задаёт бэкенд.
- Часть до `@` может быть **conn_id** — какое подключение Airflow использовать для аутентификации.
- Если conn_id не указан, берётся подключение по умолчанию для этого бэкенда.

Подключение можно передать и отдельным аргументом:

```python
ObjectStoragePath("s3://airflow-tutorial-data/", conn_id="aws_default")
```

Так удобнее, когда путь задаётся где-то ещё (например, в Asset) или когда подключение не зашито в URL. Ключевой аргумент имеет приоритет над URL.

> **Совет.** Создавать `ObjectStoragePath` в глобальной области DAG безопасно: подключение разрешается в момент использования пути, а не при создании объекта.

## Сохранение данных в объектное хранилище

Получим данные и сохраним их в облако.

*Источник: [airflow/example_dags/tutorial_objectstorage.py](https://airflow.apache.org/docs/apache-airflow/stable/_modules/airflow/example_dags/tutorial_objectstorage.html)*

```python
    @task
    def get_air_quality_data(**kwargs) -> ObjectStoragePath:
        """
        #### Get Air Quality Data
        This task gets air quality data from the Finnish Meteorological Institute's
        open data API. The data is saved as parquet.
        """
        import pandas as pd
        from typing import Mapping

        logical_date = kwargs["logical_date"]

        latitude = 28.6139
        longitude = 77.2090

        params: Mapping[str, str | float] = {
            "latitude": latitude,
            "longitude": longitude,
            "hourly": ",".join(aq_fields.keys()),
            "timezone": "UTC",
        }

        response = requests.get(API, params=params)
        response.raise_for_status()

        data = response.json()
        hourly_data = data.get("hourly", {})

        df = pd.DataFrame(hourly_data)
        df["time"] = pd.to_datetime(df["time"])

        # убедиться, что бакет существует
        base.mkdir(exist_ok=True)

        formatted_date = logical_date.format("YYYYMMDD")
        path = base / f"air_quality_{formatted_date}.parquet"

        with path.open("wb") as file:
            df.to_parquet(file)
        return path
```

Что происходит:

- Вызов публичного API (в примере — данные о качестве воздуха).
- Ответ в JSON превращается в pandas DataFrame.
- Имя файла строится по logical date задачи.
- Через **ObjectStoragePath** данные записываются в облако в формате Parquet.

Классический паттерн TaskFlow: ключ объекта меняется каждый день, пайплайн можно запускать ежедневно и накапливать набор данных. Возвращаем итоговый путь объекта для следующей задачи.

Плюс подхода: без boto3, без настройки GCS-клиента, без возни с credentials — только привычная файловая семантика, работающая поверх разных бэкендов.

## Анализ данных с DuckDB

Данные из хранилища анализируем через SQL с DuckDB.

*Источник: [airflow/example_dags/tutorial_objectstorage.py](https://airflow.apache.org/docs/apache-airflow/stable/_modules/airflow/example_dags/tutorial_objectstorage.html)*

```python
    @task
    def analyze(
        path: ObjectStoragePath,
    ):
        """
        #### Analyze
        This task analyzes the air quality data, prints the results
        """
        import duckdb

        conn = duckdb.connect(database=":memory:")
        conn.register_filesystem(path.fs)
        s3_path = path.path
        conn.execute(
            f"CREATE OR REPLACE TABLE airquality_urban AS SELECT * FROM read_parquet('{path.protocol}://{s3_path}')"
        )

        df2 = conn.execute("SELECT * FROM airquality_urban").fetchdf()
        print(df2.head())
```

Важно:

- DuckDB умеет читать Parquet напрямую.
- И DuckDB, и ObjectStoragePath опираются на **fsspec**, поэтому бэкенд хранилища легко зарегистрировать в DuckDB.
- Через **path.fs** получаем нужный объект файловой системы и регистрируем его в DuckDB.
- Далее запрос к Parquet через SQL и получение pandas DataFrame.

Путь в эту задачу приходит от предыдущей задачи через XCom — функция не собирает его вручную. Задача остаётся переносимой и не завязанной на предыдущую логику.

## Собираем всё в один DAG

Полный DAG:

*Источник: [airflow/example_dags/tutorial_objectstorage.py](https://airflow.apache.org/docs/apache-airflow/stable/_modules/airflow/example_dags/tutorial_objectstorage.html)*

```python
import pendulum
import requests
from typing import Mapping

from airflow.sdk import ObjectStoragePath, dag, task

API = "https://air-quality-api.open-meteo.com/v1/air-quality"

aq_fields = {
    "pm10": "float64",
    "pm2_5": "float64",
    "carbon_monoxide": "float64",
    "nitrogen_dioxide": "float64",
    "sulphur_dioxide": "float64",
    "ozone": "float64",
    "european_aqi": "float64",
    "us_aqi": "float64",
}
base = ObjectStoragePath("s3://aws_default@airflow-tutorial-data/")


@dag(
    schedule=None,
    start_date=pendulum.datetime(2021, 1, 1, tz="UTC"),
    catchup=False,
    tags=["example"],
)
def tutorial_objectstorage():
    """
    ### Object Storage Tutorial Documentation
    This is a tutorial DAG to showcase the usage of the Object Storage API.
    Documentation that goes along with the Airflow Object Storage tutorial is
    located
    [here](https://airflow.apache.org/docs/apache-airflow/stable/tutorial/objectstorage.html)
    """
    @task
    def get_air_quality_data(**kwargs) -> ObjectStoragePath:
        """Get air quality data from API, save as parquet."""
        import pandas as pd

        logical_date = kwargs["logical_date"]
        latitude = 28.6139
        longitude = 77.2090

        params: Mapping[str, str | float] = {
            "latitude": latitude,
            "longitude": longitude,
            "hourly": ",".join(aq_fields.keys()),
            "timezone": "UTC",
        }

        response = requests.get(API, params=params)
        response.raise_for_status()
        data = response.json()
        hourly_data = data.get("hourly", {})
        df = pd.DataFrame(hourly_data)
        df["time"] = pd.to_datetime(df["time"])

        base.mkdir(exist_ok=True)
        formatted_date = logical_date.format("YYYYMMDD")
        path = base / f"air_quality_{formatted_date}.parquet"
        with path.open("wb") as file:
            df.to_parquet(file)
        return path

    @task
    def analyze(path: ObjectStoragePath):
        """Analyze air quality data with DuckDB."""
        import duckdb

        conn = duckdb.connect(database=":memory:")
        conn.register_filesystem(path.fs)
        s3_path = path.path
        conn.execute(
            f"CREATE OR REPLACE TABLE airquality_urban AS SELECT * FROM read_parquet('{path.protocol}://{s3_path}')"
        )
        df2 = conn.execute("SELECT * FROM airquality_urban").fetchdf()
        print(df2.head())

    obj_path = get_air_quality_data()
    analyze(obj_path)

tutorial_objectstorage()
```

DAG можно запустить и посмотреть в Graph View в UI. Входы и выходы задач видны в логах; возвращённые пути — во вкладке XCom.

## Что попробовать дальше

Идеи для развития:

- Использовать object-сенсоры (например, `S3KeySensor`) для ожидания файлов от внешних систем.
- Оркестрировать переносы S3 → GCS или синхронизацию между регионами.
- Добавить ветвление при отсутствии или некорректном файле.
- Поэкспериментировать с другими форматами (CSV, JSON).

См. также:

- [Managing Connections](https://airflow.apache.org/docs/apache-airflow/stable/authoring-and-scheduling/connections.html) — настройка безопасного доступа к облачным сервисам через подключения Airflow.
- [Event-Driven Scheduling](https://airflow.apache.org/docs/apache-airflow/stable/authoring-and-scheduling/event-scheduling.html) — пайплайны по событиям (загрузка файлов, внешние триггеры).
- [TaskFlow API](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/taskflow.html) — декораторы, возвращаемые значения, цепочки задач.

---

*Источник: [Airflow 3.1.7 — Tutorial: Cloud-Native Workflows with Object Storage](https://airflow.apache.org/docs/apache-airflow/stable/tutorial/objectstorage.html). Перевод неофициальный.*
