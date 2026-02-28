# Динамические задачи (Dynamic task mapping)

**Dynamic task mapping** позволяет создавать параллельные задачи в **runtime** по списку входных значений (модель MapReduce): для каждого элемента создаётся отдельный экземпляр задачи (map), опционально одна задача собирает результаты (reduce).

## partial() и expand()

- **partial()** — задаёт параметры, общие для всех экземпляров маппленной задачи.
- **expand()** — задаёт параметры, по которым идёт «размножение»: создаётся отдельная задача на каждый элемент (списка или итератора). Для нескольких изменяемых параметров может использоваться **expand_kwargs()**.

Пример:

```python
@task
def add(x: int, y: int):
    return x + y

added_values = add.partial(y=10).expand(x=[1, 2, 3])  # три задачи: 11, 12, 13
```

Традиционный синтаксис: `PythonOperator.partial(...).expand(op_kwargs=[...])` или `.expand(op_args=...)`.

## Важные моменты

- XCom маппленных задач хранятся в виде списка; доступ по индексу (map_index): например, `ti.xcom_pull(task_ids=['my_mapped_task'], map_indexes=[2])`.
- Ограничение параллелизма: **max_active_tis_per_dag**, **max_active_tis_per_dagrun** у маппленной задачи.
- Лимит числа маппленных экземпляров задаётся конфигом **max_map_length** (по умолчанию 1024).
- В **expand()** передаются только keyword arguments; **task_id**, **pool** и ряд аргументов BaseOperator маппить нельзя.
- Пустой список от вышестоящей задачи → маппленная задача skipped; нижестоящие по умолчанию тоже skipped (зависит от trigger_rule).
- Результат маппленной задачи можно передать в следующую маппленную задачу (expand по выходу вышестоящей).
- Можно маппить по нескольким параметрам (несколько списков в expand/expand_kwargs).

## Маппинг по результату другой задачи

- **TaskFlow → TaskFlow:** в `expand()` передаёте вызов вышестоящей задачи: `plus_10_TF.partial().expand(x=one_two_three_TF())`.
- **Традиционный оператор:** используйте `.output` вышестоящей задачи: `expand(x=one_two_three_task.output)`.
- Для **PythonOperator** при маппинге по результату upstream формат должен соответствовать `op_args`/`op_kwargs` (например, список списков для op_args).

Объединение списков от двух задач: метод **.concat()** (например, `t1_obj.concat(t2_obj)`) даёт объединённый список для expand.

## UI

В Grid View маппленные задачи отображаются в одной строке с обозначением `[N]` (количество экземпляров). Логи и XCom каждого экземпляра можно открыть, выбрав соответствующий квадрат в сетке.

Подробнее: [Dynamic Tasks](https://www.astronomer.io/docs/learn/dynamic-tasks), [Dynamic Task Mapping (Airflow)](https://airflow.apache.org/docs/apache-airflow/stable/concepts/dynamic-task-mapping.html).

---

[← К содержанию](README.md) | [Task groups →](task-groups.md) | [Передача данных →](passing-data-between-tasks.md)
