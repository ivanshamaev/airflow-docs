# Кастомные стратегии XCom backend (Custom XCom backend strategies)

[XCom в Airflow](https://www.astronomer.io/docs/learn/airflow-passing-data-between-tasks) позволяют передавать данные между задачами. По умолчанию Airflow хранит XCom в [базе метаданных](https://www.astronomer.io/docs/learn/airflow-database), что удобно для локальной разработки, но ограничено по производительности. При настройке кастомного XCom backend можно задать, где и как Airflow хранит XCom, а также методы сериализации и десериализации.

В этом руководстве вы узнаете:

- Как использовать кастомный класс XCom backend с собственными методами сериализации и десериализации.
- Как настроить кастомный XCom backend с помощью Object Storage XCom Backend.
- В каких случаях использовать кастомный XCom backend.

> Кастомный XCom backend позволяет хранить практически неограниченный объём данных в XCom, но для передачи больших объёмов между задачами потребуется масштабировать и другие компоненты Airflow. По вопросам масштабирования Airflow обращайтесь в [Astronomer](https://www.astronomer.io/lp/signup/).
>
> Предупреждение

## Необходимая база

Чтобы получить максимум от руководства, нужно понимать:

- Основы облачного object storage: [AWS S3](https://aws.amazon.com/s3/), [GCP Cloud Storage](https://cloud.google.com/storage) или [Azure Blob Storage](https://azure.microsoft.com/en-us/products/storage/blobs/).
- Основы XCom. См. [Передача данных между задачами](https://www.astronomer.io/docs/learn/airflow-passing-data-between-tasks).

## Зачем использовать кастомный XCom backend?

Типичные причины использовать кастомный XCom backend:

- Нужно сохранять XCom в нескольких местах одновременно.
- Нужно ограничить допустимые типы значений XCom.
- Нужен доступ к XCom без обращения к базе метаданных.
- Production-окружение с собственными политиками хранения, удаления и резервного копирования XCom.
- Требуется больший объём хранилища для XCom, чем может предоставить база метаданных Airflow.

Кастомный XCom backend также позволяет задать собственные методы сериализации и десериализации, если нужно добавить сериализацию для класса или регистрация кастомного сериализатора невозможна. Подробнее: [Кастомная сериализация и десериализация](https://www.astronomer.io/docs/learn/custom-xcom-backend-strategies#custom-serialization-and-deserialization).

## Как настроить кастомный XCom backend

Есть два основных способа:

- **Кастомный класс XCom backend:** когда нужно детально настроить хранение XCom, например сохранять XCom одновременно в двух разных хранилищах.
- **Object Storage XCom Backend:** когда XCom нужно хранить в облачном object storage (AWS S3, GCP Cloud Storage, Azure Blob Storage). Этот вариант рекомендуется, если достаточно одного удалённого хранилища и устраивают порог и сжатие Object Storage XCom Backend.

Кроме того, некоторые провайдеры поставляют готовые XCom backend. Например, [Snowpark provider](https://www.astronomer.io/docs/learn/airflow-snowpark) содержит XCom backend для Snowflake.

### Object Storage XCom Backend

Кастомный XCom backend можно построить на object storage. Object Storage XCom Backend входит в провайдер [Common IO](https://registry.astronomer.io/providers/apache-airflow-providers-common-io/versions/latest) и настраивается переменными окружения:

- **`AIRFLOW__COMMON_IO__XCOM_OBJECTSTORAGE_COMPRESSION`:** (опционально) алгоритм сжатия при хранении XCom в object storage, например `zip`. По умолчанию `None`.
- **`AIRFLOW__COMMON_IO__XCOM_OBJECTSTORAGE_THRESHOLD`:** порог в байтах: XCom не больше этого размера хранятся в базе метаданных, больше — в object storage. По умолчанию `-1` (все XCom в базе метаданных).
- **`AIRFLOW__COMMON_IO__XCOM_OBJECTSTORAGE_PATH`:** путь в object storage для XCom в формате `scheme://connection/bucket/xcom`. Пример: `s3://my-s3-connection@my-bucket/xcom`. Типичные схемы: `s3`, `gs`, `abfs` для Amazon S3, Google Cloud Storage и Azure Blob Storage.
- **`AIRFLOW__CORE__XCOM_BACKEND`:** класс XCom backend. Для Object Storage XCom Backend задайте `airflow.providers.common.io.xcom.backend.XComObjectStorageBackend`.

Пошаговая настройка кастомного XCom backend на базе Object Storage для Amazon S3, Google Cloud Storage и Azure Blob Storage: [Set up a custom XCom backend using object storage](https://www.astronomer.io/docs/learn/custom-xcom-backends-tutorial).

### Кастомный класс XCom backend

Чтобы реализовать кастомный XCom backend, нужно описать класс, наследующий `BaseXCom`.

Ниже — пример класса `MyCustomXComBackend`, который допускает только JSON-сериализуемые XCom и сохраняет их и в Amazon S3, и в Google Cloud Storage в методе `serialize_value()`. Метод `deserialize_value()` забирает XCom из бакета Amazon S3 и возвращает значение.

В базе метаданных Airflow хранится строка-ссылка на XCom (она отображается во вкладке XCom в UI). Строка имеет префикс `s3_and_gs://`, чтобы было видно, что XCom лежит в S3 и GCS. В `serialize_value()` и `deserialize_value()` можно добавлять любую нужную логику сериализации и десериализации; подробнее: [Кастомная сериализация и десериализация](https://www.astronomer.io/docs/learn/custom-xcom-backend-strategies#custom-serialization-and-deserialization).

```python
class MyCustomXComBackend(BaseXCom):
    # префикс опционален, упрощает распознавание ссылок в базе метаданных
    # на XCom, хранящиеся в удалённом хранилище
    PREFIX = "s3_and_gs://"
    S3_BUCKET_NAME = "s3-xcom-backend-example"
    GS_BUCKET_NAME = "gcs-xcom-backend-example"

    @staticmethod
    def serialize_value(
        value,
        key=None,
        task_id=None,
        dag_id=None,
        run_id=None,
        map_index=None,
        **kwargs,
    ):
        # проверка JSON-сериализуемости
        try:
            serialized_value = json.dumps(value)
        except TypeError as e:
            raise ValueError(f"XCom value is not JSON-serializable!: {e}")

        # временный JSON-файл со значением
        with tempfile.NamedTemporaryFile(mode="w+", delete=False) as tmp_file:
            tmp_file.write(serialized_value)
            tmp_file.flush()
            tmp_file_name = tmp_file.name

            # подключение к AWS через S3 hook
            hook = S3Hook(aws_conn_id="my_aws_conn_id")
            # уникальный file_id: комбинация task_id, run_id, map_index или uuid
            filename = "data_" + str(uuid.uuid4()) + ".json"
            key = f"{dag_id}/{run_id}/{task_id}/{map_index}/{key}_{filename}"

            # загрузка локального JSON в S3
            hook.load_file(
                filename=tmp_file_name,
                key=key,
                bucket_name=MyCustomXComBackend.S3_BUCKET_NAME,
                replace=True,
            )

            # подключение к GCS через GCS hook
            hook = GCSHook(gcp_conn_id="my_gcs_conn_id")

            if hook.exists(MyCustomXComBackend.GS_BUCKET_NAME, key):
                print(
                    f"File {key} already exists in the bucket {MyCustomXComBackend.GS_BUCKET_NAME}."
                )
            else:
                # загрузка локального JSON в GCS
                hook.upload(
                    filename=tmp_file_name,
                    object_name=key,
                    bucket_name=MyCustomXComBackend.GS_BUCKET_NAME,
                )

        # строка-ссылка для сохранения в базе метаданных Airflow
        reference_string = MyCustomXComBackend.PREFIX + key

        # сериализация ссылки в JSON для записи в базу метаданных (как обычный XCom)
        return BaseXCom.serialize_value(value=reference_string)

    @staticmethod
    def deserialize_value(result):
        import logging

        reference_string = BaseXCom.deserialize_value(result=result)
        hook = S3Hook(aws_conn_id="my_aws_conn")
        key = reference_string.replace(MyCustomXComBackend.PREFIX, "")

        with tempfile.TemporaryDirectory() as tmp_dir:
            local_file_path = hook.download_file(
                key=key,
                bucket_name=MyCustomXComBackend.S3_BUCKET_NAME,
                local_path=tmp_dir,
            )

            file_size = os.path.getsize(local_file_path)
            logging.info(f"Downloaded file size: {file_size} bytes.")
            if file_size == 0:
                raise ValueError(
                    f"The downloaded file is empty. Check the content of the S3 object at {key}."
                )

            with open(local_file_path, "r") as file:
                try:
                    output = json.load(file)
                except json.JSONDecodeError as e:
                    logging.error(f"Error decoding JSON from the file: {e}")
                    raise

        return output
```

Чтобы использовать кастомный класс XCom backend, сохраните его в Python-файле в каталоге `include` вашего проекта Airflow. Затем задайте в окружении Airflow переменную `AIRFLOW__CORE__XCOM_BACKEND` путём к этому классу. При локальном запуске с Astro CLI переменную можно задать в файле `.env` проекта Astro. В Astro — [через UI Astro](https://docs.astronomer.io/astro/environment-variables).

```text
AIRFLOW__CORE__XCOM_BACKEND=include.<your-file-name>.MyCustomXComBackend
```

Для дальнейшей настройки поведения можно переопределять другие методы [модуля XCom](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/xcoms.html) ([исходный код](https://github.com/apache/airflow/blob/main/airflow-core/src/airflow/models/xcom.py)).

## Кастомная сериализация и десериализация

По умолчанию в Airflow есть [методы сериализации](https://airflow.apache.org/docs/apache-airflow/stable/authoring-and-scheduling/serializers.html) для распространённых типов: [JSON](https://www.json.org/json-en.html), [pandas DataFrame](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.html), [NumPy](https://numpy.org/).

Если через XCom нужно передавать объекты, для которых сериализатора нет, возможны варианты:

- **Кастомный XCom backend** с собственными методами сериализации и десериализации. См. [Кастомный класс XCom backend](https://www.astronomer.io/docs/learn/custom-xcom-backend-strategies#use-a-custom-xcom-backend-class).
- **Методы `serialize()` и `deserialize()`** в классе передаваемого объекта. См. [Serialization](https://airflow.apache.org/docs/apache-airflow/stable/authoring-and-scheduling/serializers.html).
- **Регистрация кастомного сериализатора.** См. [Serialization](https://airflow.apache.org/docs/apache-airflow/stable/authoring-and-scheduling/serializers.html).

---

[← Пуллы](airflow-pools.md) | [К содержанию](README.md) | [Deferrable →](deferrable-operators.md)
