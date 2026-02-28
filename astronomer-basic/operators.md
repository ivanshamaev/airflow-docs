# Операторы Airflow

Операторы — один из строительных блоков DAG в Airflow. `PythonOperator` выполняет любую Python-функцию (аналог декоратора `@task`); другие операторы содержат готовую логику: выполнение Bash (`BashOperator`), SQL в БД (`SQLExecuteQueryOperator`) и т.д. Операторы импортируются из пакетов провайдеров Airflow.

Список операторов: [Astronomer Registry — Operators](https://registry.astronomer.io/modules?typeName=Operators).

## Основы

Операторы — классы, инкапсулирующие единицу работы. При создании экземпляра оператора в DAG с нужными параметрами он становится **задачей (task)**.

Базовый набор — в [Airflow standard provider](https://airflow.apache.org/docs/apache-airflow-providers-standard/stable/index.html); остальные — в пакетах провайдеров (Snowflake, Google, Common SQL, Common IO, Common Messaging и др.).

## Примеры операторов

- **PythonOperator** — выполнение Python-функции. Эквивалент `@task`. См. [Декораторы и TaskFlow](https://www.astronomer.io/docs/learn/airflow-decorators).
- **BashOperator** — выполнение bash-команды или скрипта. См. [BashOperator](bashoperator.md).
- **KubernetesPodOperator** — запуск задачи в Pod в Kubernetes.
- **SQLExecuteQueryOperator** — выполнение SQL в реляционной БД.
- **EmptyOperator** — пустая задача (заглушка).

Все операторы наследуются от **BaseOperator**. Общие аргументы:

- **task_id** — уникальный идентификатор задачи (обязателен).
- **retries** — число повторов при сбое (по умолчанию 0).
- **pool** — имя пула (по умолчанию None).
- **execution_timeout** — максимальное время выполнения (рекомендуется задавать).

Эти и другие аргументы можно задать на уровне DAG через **default_args** и переопределять в конкретной задаче.

## Рекомендации

- Используйте [Astronomer Registry](https://registry.astronomer.io/modules?types=operators) для поиска операторов и параметров.
- Если есть оператор под задачу — используйте его вместо своей функции или [хука](hooks.md).
- Операторы и [декораторы](https://www.astronomer.io/docs/learn/airflow-decorators) можно сочетать в одном DAG.
- [Сенсоры](sensors.md) — разновидность операторов, которые ждут наступления условия.
- [Deferrable-операторы](https://www.astronomer.io/docs/learn/deferrable-operators) освобождают слот воркера во время ожидания; для долгих ожиданий рекомендуется включать `deferrable=True` где доступно.
- Для работы с внешними системами обычно нужна [подключение (connection)](connections.md).

---

[← К содержанию](README.md) | [BashOperator →](bashoperator.md) | [Хуки →](hooks.md) | [Сенсоры →](sensors.md)
