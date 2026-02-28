# Масштабирование Airflow (Scaling workers)

Airflow позволяет масштабировать окружение под нагрузку. Важно настроить параметры уровня окружения, DAG и задач, а также учитывать выбранный [executor](executors.md).

Параметры, влияющие на производительность:

- **Уровень окружения** — все DAG.
- **Уровень DAG** — один DAG.
- **Уровень задачи** — отдельные задачи.

Справка дана для Airflow 2.0+. Текущие значения смотрите в Admin → Configuration в UI; документация: [Configuration Reference](https://airflow.apache.org/docs/apache-airflow/stable/configurations-ref.html), [Setting Configuration Options](https://airflow.apache.org/docs/apache-airflow/stable/howto/set-config.html). На Astronomer параметры задаются [environment variables](https://www.astronomer.io/docs/astro/environment-variables).

## Параметры уровня окружения (Core)

Переменные вида `AIRFLOW__CORE__PARAMETER_NAME`.

- **dagbag_import_timeout** — таймаут импорта DAG (сек), должен быть меньше `dag_file_processor_timeout`. По умолчанию 30.
- **dag_file_processor_timeout** — таймаут обработки одного DAG-файла. По умолчанию 50 сек.
- **max_active_runs_per_dag** — максимум активных DAG run на один DAG. По умолчанию 16. Важно при backfill.
- **max_active_tasks_per_dag** (ранее dag_concurrency) — максимум одновременно планируемых задач по всем run одного DAG. По умолчанию 16. Если увеличили воркеры, но задачи не идут — поднимите также `parallelism` и этот параметр.
- **parallelism** — максимум задач в состоянии running/queued на один планировщик в окружении. При двух планировщиках и parallelism=32 — не более 64 задач суммарно. По умолчанию 32. На Astro часто выставляется автоматически по числу воркеров.

## Параметры планировщика (Scheduler)

Переменные: `AIRFLOW__SCHEDULER__PARAMETER_NAME`.

- **dag_dir_list_interval** — как часто сканировать каталог DAG (сек). Меньше значение — быстрее появление новых DAG, выше нагрузка на CPU. По умолчанию 300. На Astro при &lt;200 DAG можно задать `AIRFLOW__SCHEDULER__DAG_DIR_LIST_INTERVAL=30`.
- **min_file_process_interval** — интервал парсинга каждого DAG-файла (сек). По умолчанию 30.
- **parsing_processes** (ранее max_threads) — число процессов парсинга DAG. Рекомендация Astronomer: примерно 2× число vCPU. На Astro по умолчанию зависит от размера (Small/Medium/Large/Extra Large).
- **max_tis_per_query** — размер батча запросов к метаданным в основном цикле планировщика. По умолчанию 16; должен быть меньше `core.parallelism`.
- **max_dagruns_to_create_per_loop** — максимум DAG run, создаваемых за один цикл. По умолчанию 10.
- **scheduler_heartbeat_sec** — период цикла планировщика (сек). По умолчанию 5.
- **file_parsing_sort_mode** — порядок разбора DAG-файлов: `modified_time`, `random_seeded_by_host`, `alphabetical`. По умолчанию `modified_time`.

## Параметры уровня DAG

Задаются в коде DAG; переопределяют настройки окружения.

- **concurrency** / **max_active_tasks** — максимум одновременно выполняемых task instance по всем активным DAG run этого DAG. Если не задано — берётся `max_active_tasks_per_dag` окружения.
- **max_active_runs** — максимум активных DAG run для этого DAG. При использовании catchup/backfill лучше задать явно.

Пример:

```python
@dag("my_dag_id", concurrency=10, max_active_runs=3)
def my_dag():
    ...

with DAG("my_dag_id", concurrency=10, max_active_runs=3):
    ...
```

## Параметры уровня задачи

Наследуются от BaseOperator; задаются в операторе.

- **pool** — [пул](https://www.astronomer.io/docs/learn/airflow-pools): ограничение параллельных экземпляров группы задач (например, под лимиты API).
- **max_active_tis_per_dagrun** — максимум одновременных выполнений этой задачи в рамках одного DAG run. Важно для [динамически маппленных задач](https://www.astronomer.io/docs/learn/dynamic-tasks).
- **max_active_tis_per_dag** (ранее task_concurrency) — максимум одновременных выполнений этой задачи по всем DAG run. Для задачи, работающей с единственным ресурсом, можно поставить 1.

Пример:

```python
@task(pool="my_custom_pool", max_active_tis_per_dag=14)
def t1():
    pass

t1 = PythonOperator(..., pool="my_custom_pool", max_active_tis_per_dag=14)
```

## Исполнители и масштабирование

- **CeleryExecutor:** масштабирование — число и размер воркеров. Параметр **worker_concurrency** (`AIRFLOW__CELERY__WORKER_CONCURRENCY`) — сколько задач выполняет один воркер одновременно (по умолчанию 16). При увеличении может потребоваться больше CPU/RAM.
- **KubernetesExecutor:** каждая задача в своём Pod; ресурсы задаются на задачу. Важна настройка кластера (в т.ч. autoscaling). **worker_pods_creation_batch_size** (`AIRFLOW__KUBERNETES__WORKER_PODS_CREATION_BATCH_SIZE`) — сколько подов создаётся за цикл планировщика (по умолчанию 1); увеличение ускоряет запуск при многих параллельных задачах.

## Типичные проблемы при масштабировании

- **Один DAG не параллелит задачи, остальные в порядке** — ограничение на уровне DAG: проверьте `max_active_tasks_per_dag`, пулы, `parallelism`.
- **Задачи в queued, но не запускаются** — превышена ёмкость инфраструктуры. Kubernetes: свободные ресурсы в namespace, `worker_pods_creation_batch_size`. Celery: `worker_concurrency`.
- **Высокая задержка планирования** — планировщику не хватает ресурсов на парсинг DAG. Увеличьте ресурсы планировщика, при Celery — `worker_concurrency` или `parallelism`.

Дополнительно: [Apache Airflow Slack](https://airflow.apache.org/community/), [Astronomer support](https://www.astronomer.io/get-astronomer/).

---

[← Исполнители](executors.md) | [К содержанию](README.md)
