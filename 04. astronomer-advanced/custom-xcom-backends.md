# Кастомные XCom backend

По умолчанию XCom хранятся в [метаданных БД](../03. astronomer-infra/airflow-database.md). **Custom XCom backend** задаёт, где и как хранить XCom, а также свою сериализацию/десериализацию. Применение: несколько хранилищ одновременно, ограничение типов значений, доступ без прямого обращения к БД, политики хранения и бэкапов, больший объём данных.

Два основных подхода: **Object Storage XCom Backend** (провайдер Common IO) — XCom выше порога сохраняются в S3/GCS/Azure Blob; **кастомный класс**, наследующий `BaseXCom`, с переопределением `serialize_value()` и `deserialize_value()`. Переменные для Object Storage: `AIRFLOW__COMMON_IO__XCOM_OBJECTSTORAGE_PATH` (путь вида `s3://conn@bucket/xcom`), `AIRFLOW__COMMON_IO__XCOM_OBJECTSTORAGE_THRESHOLD` (порог в байтах), `AIRFLOW__COMMON_IO__XCOM_OBJECTSTORAGE_COMPRESSION`, `AIRFLOW__CORE__XCOM_BACKEND` (класс backend). Для своего класса: положить модуль в `include/`, задать `AIRFLOW__CORE__XCOM_BACKEND=include.имя_файла.ИмяКласса`.

Кастомная сериализация нужна, если стандартных сериализаторов (JSON, pandas, NumPy и т.д.) недостаточно: можно реализовать её в backend или добавить `serialize()`/`deserialize()` к классу объекта, либо зарегистрировать кастомный сериализатор. Для больших объёмов данных масштабировать нужно и другие компоненты Airflow.

Подробнее: [Custom XCom backend strategies](https://www.astronomer.io/docs/learn/custom-xcom-backend-strategies), [Custom XCom backends tutorial](https://www.astronomer.io/docs/learn/custom-xcom-backends-tutorial).

---

[← Пуллы](airflow-pools.md) | [К содержанию](README.md) | [Deferrable →](deferrable-operators.md)
