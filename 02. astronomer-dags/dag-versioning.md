# Версионирование DAG и DAG bundles

В Airflow 3.0 появилось **версионирование DAG** — одна из самых востребованных возможностей. С его помощью можно отслеживать изменения DAG во времени в UI и видеть полную историю DAG run. Версионирование включается автоматически, настраивать ничего не нужно. **Versioned DAG bundles** позволяют избежать конфликтов версий при пуше кода и перезапускать старые DAG run с исходным кодом.

В этом руководстве — введение в версионирование DAG и DAG bundles и настройка версионируемого `GitDagBundle`.

## Необходимая база

Полезно понимать:

- Основы GitHub. См. [документацию GitHub](https://docs.github.com/en).
- DAG в Airflow. См. [Введение в DAG Airflow](../01.%20astronomer-basic/dags.md).

## Зачем нужно версионирование DAG

В Airflow 2 и UI, и выполнение DAG всегда использовали последнюю версию кода. Из этого следовали ограничения:

- **Смешение версий в одном run:** если во время выполнения DAG изменили (например, таблицу с X на Y), задача A из старой версии могла передать имя таблицы X, а обновлённая задача B записывала данные для таблицы Y в таблицу X.
- **Конфликты версий при пуше кода:** при изменении кода DAG во время его выполнения часть задач одного run могла выполняться по старой версии, часть — по новой, с риском непредсказуемого результата.
- **Нет истории старых версий:** после изменения DAG (например, удаления задачи) вся история по этой задаче пропадала в grid и graph view в UI.

DAG bundles и версионирование DAG в Airflow 3 решают эти проблемы.

## Версионирование DAG и DAG bundles

В Airflow 3 введены два понятия:

- **DAG versioning (версионирование DAG):** Airflow отслеживает изменения DAG. Это происходит автоматически при любом типе DAG bundle. Новая версия DAG создаётся при каждом создании DAG run для DAG, у которого с прошлого run изменилась **структура** — всё, что влияет на `serdag`: параметры DAG и задач, зависимости, task_id, добавление или удаление задач.

- **DAG bundle:** набор файлов с кодом DAG и вспомогательными файлами. Имя bundle связано с бэкендом хранения (например, `LocalDagBundle` — локальная ФС, `GitDagBundle` — Git-репозиторий). Версия DAG bundle создаётся версионированием бэкенда: у `GitDagBundle` новая версия — при каждом [Git](https://git-scm.com/doc) коммите, даже если DAG не менялись. По умолчанию `LocalDagBundle` **не версионируется**; такие bundle, как `GitDagBundle`, — **версионируются**.

Поведение:

- При создании нового DAG run планировщик использует последнюю версию DAG.
- Каждый DAG run привязан к версии DAG, которая видна в UI.
- Использование bundle, отличного от `LocalDagBundle`, требует настройки конфигурации Airflow.

## Версионирование DAG

Версии DAG можно смотреть в нескольких местах UI. В меню Options у графа DAG можно выбрать версию графа для отображения. На странице деталей DAG показывается последняя доступная версия (она используется для новых DAG run).

В DAG grid сохраняется история по всем задачам, в том числе удалённым в последней версии. Во вкладке Code можно выбрать версию кода DAG для отображения.

## DAG bundles

На Astro в режиме [Hosted execution](https://www.astronomer.io/docs/astro/execution-mode) версионируемый DAG bundle настраивается автоматически. Настраивать свои DAG bundles и использовать несколько bundle в одном Astro Deployment можно только в [Remote Execution](https://www.astronomer.io/docs/astro/execution-mode#remote-execution). Подробнее: [Configure Dag bundles for Remote Execution](https://www.astronomer.io/docs/astro/dag-bundles).

DAG bundle содержит код DAG и вспомогательные файлы. Бывают версионируемые и неверсионируемые bundle: стандартный `LocalDagBundle` не версионируется, `GitDagBundle` — версионируется. Поддержка других бэкендов планируется в следующих релизах.

Поведение при разных сценариях:

| Сценарий | Версионируемый DAG bundle | Неверсионируемый DAG bundle |
| --- | --- | --- |
| **Создание DAG run** | Планировщик по умолчанию использует версию DAG на момент DAG run для определения, какие task instances создавать; воркеры выполняют задачи по коду из версии bundle на момент этого run. Поведение можно изменить: на форме clearing отметить «Run with latest bundle version» или в API при clearing задать `run_on_latest_version=True`. | Используется текущий (последний) код DAG. |
| **Clearing и перезапуск прошлого DAG run** | Каждый закоммиченный или сохранённый структурный переход DAG создаёт новую версию DAG. С каждой новой версией bundle у всех DAG с структурными изменениями появляется новая версия DAG. | Каждое структурное изменение DAG даёт новую версию DAG. |
| **Изменение кода** | DAG run завершается на той версии bundle, с которой начался. | DAG всегда использует текущий код на момент старта задачи (как в Airflow 2). |
| **Изменение кода во время выполнения DAG** | По умолчанию используется код задач из версии bundle на момент исходного DAG run. Можно переключиться на последнюю версию bundle: «Run with latest bundle version» на форме clearing или `run_on_latest_version=True` при clearing task instance в API. | При перезапуске отдельных задач используется последняя версия DAG. |
| **Перезапуск отдельных задач прошлого DAG run** | По умолчанию — код из версии bundle на момент того run; опционально — последняя версия bundle. | Используется последняя версия DAG. |

Подробнее о DAG bundles: [документация Airflow](https://airflow.apache.org/docs/apache-airflow/stable/administration-and-deployment/dag-bundles.html), в том числе создание кастомного DAG bundle.

### Настройка GitDagBundle

Чтобы подтягивать код DAG напрямую из репозитория GitHub, можно использовать **GitDagBundle** (он версионируется). Настройка для проекта на Astro CLI:

1. **Установить [Airflow Git provider](https://airflow.apache.org/docs/apache-airflow-providers-git/stable/index.html):** добавить в `requirements.txt` (подставьте актуальную версию вместо `<version>`):

```text
apache-airflow-providers-git==<version>
```

2. **Установить пакет `git` в проекте:** добавить в `packages.txt`:

```text
git
```

3. **Залить код DAG в [GitHub-репозиторий](https://docs.github.com/en).**

4. **Задать Git-подключение** через переменную окружения в `.env`. Подставьте `<account>` и `<repo>` (аккаунт и репозиторий GitHub), вместо `github_pat_<your-token>` — ваш [GitHub Personal Access Token](https://docs.github.com/en) (достаточно прав на чтение содержимого репозитория):

```text
AIRFLOW_CONN_MY_GIT_CONN='{
    "conn_type": "git",
    "host": "https://github.com/<account>/<repo>.git",
    "password": "github_pat_<your-token>"
}'
```

5. **Переключить конфигурацию на GitDagBundle:** задать в `.env` переменную `[dag_processor].dag_bundle_config_list`. Подставьте имя bundle (`your-bundle-name`), в `subdir` укажите каталог в репозитории с DAG, в `tracking_ref` — ветку (например, `main`):

```text
AIRFLOW__DAG_PROCESSOR__DAG_BUNDLE_CONFIG_LIST='[
    {
        "name": "your-bundle-name",
        "classpath": "airflow.providers.git.bundles.git.GitDagBundle",
        "kwargs": {
            "git_conn_id": "my_git_conn",
            "subdir": "dags",
            "tracking_ref": "main"
        }
    }
]'
```

6. **Перезапустить проект:** `astro dev restart`, чтобы применить изменения.

## Программно генерируемые DAG и DAG bundles

Если DAG создаются программно (код генерирует структуру DAG) и используется версионируемый DAG bundle, нужно добиться, чтобы **не было изменения структуры DAG без изменения версии bundle**.

При clearing DAG run планировщик берёт версию bundle на момент этого run, чтобы определить, какие task instances создавать; воркеры выполняют задачи по коду из той же версии bundle. Если из-за программной генерации структура (и значит версия DAG) меняется без смены версии bundle, планировщик и воркеры могут использовать разные версии DAG для создания и выполнения задач, что ведёт к непредсказуемому поведению.

Безопасный пример для версионируемого bundle — использование [dag-factory](https://pypi.org/project/dag-factory/) или цикл по списку, который меняется только при изменении кода:

```python
# этот список меняется только при изменении кода
my_tables = ["TABLE_A", "TABLE_B", "TABLE_C"]

for my_table in my_tables:
    @task(task_id=f"modify_{my_table}")
    def modify_table(my_table):
        # работа с таблицей
        pass

    modify_table(my_table=my_table)
```

Если используется код верхнего уровня, обращающийся к внешней системе (этого лучше избегать, см. [Избегайте кода на верхнем уровне DAG-файла](dag-best-practices.md)), возможна смена структуры DAG без смены версии bundle. Например, если список `my_tables` в примере выше получается запросом к БД, структура DAG может меняться при каждом парсинге без нового коммита в Git.

---

[← Параметры DAG](dag-parameters.md) | [К содержанию](README.md) | [Отладка →](debugging-dags.md)
