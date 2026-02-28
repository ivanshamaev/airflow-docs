# Airflow Object Storage

В Airflow 2.8 добавлен [Object Storage](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/objectstorage.html) — единый API для работы с облачным и локальным хранилищем (S3, GCS, Azure Blob, локальная ФС). Вместо разных XToYTransferOperator используется **ObjectStoragePath** — объект, похожий на `pathlib.Path`.

## Зачем использовать

- **Эффективная передача файлов** — потоковая передача чанками через `shutil.copyfileobj()`, без полной загрузки в память.
- **Передача между разными хранилищами** без смены кода.
- **Path API** — `iterdir()`, `is_dir()`, `is_file()`, `read_block()`, `copy()`, `unlink()` и т.д. Ограничения см. в [Cloud Object Stores are not real file systems](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/objectstorage.html#cloud-object-stores-are-not-real-file-systems).

## Использование

```python
from airflow.io.path import ObjectStoragePath

# S3
base = ObjectStoragePath("s3://my-bucket/path/", conn_id="aws_default")
# GCS, Azure: gs://, abfs://
# Локально: file://
```

Методы: `base.iterdir()`, `obj.is_file()`, `obj.read_block()`, `src.copy(dst=dst)`, `file.unlink()`. Объекты `ObjectStoragePath` можно передавать между задачами через XCom. Провайдеры: Amazon (s3fs), Google, Azure — устанавливаются в `requirements.txt`.

## Пример MLOps-пайплайна

Типичный сценарий: список файлов в ingest → копирование в train → чтение текста → обучение модели (Naive Bayes) → предсказание по quote из params → копирование в archive → очистка train. Три пути (`base_path_ingest`, `base_path_train`, `base_path_archive`) могут указывать на разные хранилища (S3, file:// и т.д.); смена хранилища — через константы и connection, без правок логики DAG.

Подробнее: [Airflow object storage tutorial](https://www.astronomer.io/docs/learn/airflow-object-storage-tutorial), [Object Storage (Airflow docs)](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/objectstorage.html).

---

[← К содержанию](README.md) | [Документирование DAG →](dag-documentation.md)
