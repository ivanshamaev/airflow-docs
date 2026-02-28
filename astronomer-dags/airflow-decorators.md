# Декораторы Airflow (Airflow decorators) и TaskFlow API

**TaskFlow API** — функциональный способ задавать DAG и задачи через декораторы. Это упрощает передачу данных между задачами и объявление зависимостей: вывод одной задачи передаётся аргументом в другую. Декораторы можно сочетать с традиционными операторами.

## Что такое декоратор

В Python декоратор — функция, которая принимает другую функцию и расширяет её поведение. В Airflow декоратор превращает обычную функцию в задачу, группу задач или DAG.

## Когда использовать TaskFlow API

TaskFlow уменьшает объём шаблонного кода по сравнению с традиционными операторами и часто делает DAG короче и понятнее. Выбор стиля — вопрос предпочтений; в одном DAG можно [смешивать декораторы и операторы](https://www.astronomer.io/docs/learn/airflow-decorators#mixing-taskflow-decorators-with-traditional-operators).

## Как использовать

Задайте задачи декоратором `@task`; зависимости выводятся из вызовов: результат одной задачи передаётся аргументом в другую. XCom и зависимости обрабатываются автоматически.

**Пример (декораторы):**

```python
@dag(schedule="@daily", start_date=datetime(2021, 12, 1), catchup=False)
def taskflow():
    @task(task_id="extract", retries=2)
    def extract_bitcoin_price() -> Dict[str, float]:
        return requests.get(API).json()["bitcoin"]

    @task(multiple_outputs=True)
    def process_data(response: Dict[str, float]) -> Dict[str, float]:
        return {"usd": response["usd"], "change": response["usd_24h_change"]}

    @task
    def store_data(data: Dict[str, float]):
        logging.info(f"Store: {data['usd']} with change {data['change']}")

    store_data(process_data(extract_bitcoin_price()))

taskflow()
```

**Важно:**

- Результат вызванной задачи можно сохранить в переменную и передать в несколько нижестоящих: `my_fruits = get_fruit_options(); eat_a_fruit(my_fruits); gift_a_fruit(my_fruits)`.
- Декорировать можно импортированную функцию: `@task` на функцию из другого файла.
- При нескольких вызовах одной и той же задачи без переопределения `task_id` Airflow добавляет суффикс (`say_hello`, `say_hello__1`, …). Явный ID: `say_hello.override(task_id="greet_dog")("Piglet")`.
- Параметры задачи переопределяются через `.override()`: `taskflow_func.override(retries=5, pool="my_pool", task_id="greeting")()`.
- В декораторе можно задать `task_id`, `retries`, `pool` и другие параметры BaseOperator.
- Все декорированные функции должны быть **вызваны** в файле DAG (включая функцию DAG в конце), чтобы задачи и DAG зарегистрировались.

## Смешение декораторов и традиционных операторов

- **TaskFlow → TaskFlow:** передаёте вызов вышестоящей задачи аргументом: `plus_10_TF(get_23_TF())`.
- **Традиционный → TaskFlow:** передайте `.output` оператора в вызов TaskFlow-задачи: `second_task(first_task.output)`; зависимость создаётся автоматически. Если данные не передаёте, связь задайте через `chain()` или `>>`.
- **TaskFlow → традиционный:** результат вызова TaskFlow (XComArg) передаётся в шаблонируемое поле традиционного оператора или в `op_args`/`op_kwargs` (например, `op_args=[_first_task]` у PythonOperator).
- **Традиционный → традиционный:** используйте `op_args=[get_23_task.output]` и явно задайте зависимость: `chain(get_23_task, plus_10_task)`.

## Доступные декораторы

- **@task()** — Python-задача.
- **@dag()** — DAG.
- **@task_group()** — [группа задач](task-groups.md).
- **@task.bash()** — BashOperator.
- **@task.sensor()** — сенсор из функции.
- **@task.branch()** — ветвление по условию.
- **@task.short_circuit()** — пропуск нижестоящих при False.
- **@task.docker()**, **@task.kubernetes()** — Docker/Kubernetes.
- **@task.virtualenv()** — выполнение в виртуальном окружении.
- **@task.branch_virtualenv**, **@task.branch_external_python** — ветвление в отдельном venv.
- **@task.pyspark()** — PySpark (SparkSession/SparkContext).

Подробнее: [Airflow custom decorator](https://airflow.apache.org/docs/apache-airflow/stable/howto/create-custom-decorator.html).

---

[← Контекст](airflow-context.md) | [К содержанию](README.md) | [Передача данных →](passing-data-between-tasks.md) | [Task groups →](task-groups.md)
