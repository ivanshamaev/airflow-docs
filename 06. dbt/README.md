# 06. dbt

Раздел посвящён интеграции **dbt Core** с **Apache Airflow** с помощью пакета **Cosmos** (Astronomer).

## Содержание раздела

- **[Airflow и dbt](airflow-dbt.md)** — зачем связывать Airflow и dbt, настройка Astro-проекта, подготовка проекта dbt, подключение к хранилищу, написание DAG с Cosmos (DbtTaskGroup), альтернативы (BashOperator, manifest-файл).
- **[Cosmos на Astro (Getting Started)](astro-getting-started.md)** — быстрый старт Cosmos на Astro: режим local, venv в Dockerfile, установка Cosmos, размещение dbt-проекта, DbtDag, запуск.
- **[Cosmos на open-source Airflow](getting-started-open-source.md)** — быстрый старт Cosmos на открытом Airflow: правка образа, venv в Dockerfile, установка Cosmos, размещение dbt-проекта, DbtDag.
- **[Режимы выполнения (Execution Modes)](execution-modes.md)** — local, virtualenv, docker, kubernetes, AWS EKS, Azure Container Instance, GCP Cloud Run Job, AWS ECS, airflow_async, watcher, watcher_kubernetes; сравнение; Invocation Modes.
- **[Режим Docker](getting-started-docker.md)** — пошаговый туториал: требования, установка Airflow и Cosmos, Postgres, сборка образа dbt, запуск DAG, ProfileConfig в Docker.
- **[Операторы Cosmos](getting-started-operators.md)** — операторы под команды dbt (формат имени), DbtCloneLocalOperator (dbt clone), DbtSeedLocalOperator (dbt seed).
