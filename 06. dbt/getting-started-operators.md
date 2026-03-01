# Операторы (Operators)

Cosmos предоставляет отдельные операторы для конкретных команд dbt; их можно использовать как обычные [операторы Apache Airflow®](https://airflow.apache.org/). Имена операторов в Cosmos имеют вид `Dbt<Команда><Режим>Operator`, например `DbtBuildLocalOperator`.

## Clone

**Требования**

- Cosmos >= 1.8.0
- dbt-core >= 1.6.0

Оператор `DbtCloneLocalOperator` реализует команду [dbt clone](https://docs.getdbt.com/reference/commands/clone).

Пример использования:

```python
clone_operator = DbtCloneLocalOperator(
    profile_config=profile_config,
    project_dir=DBT_PROJ_DIR,
    task_id="clone",
    dbt_cmd_flags=["--models", "stg_customers", "--state", DBT_ARTIFACT],
    install_deps=True,
    append_env=True,
)
```

## Seed

Оператор `DbtSeedLocalOperator` реализует команду [dbt seed](https://docs.getdbt.com/reference/commands/seed).

В примере ниже `DbtSeedLocalOperator` используется для загрузки семян в `raw_orders`:

```python
seed_raw_orders = DbtSeedLocalOperator(
    profile_config=profile_config,
    project_dir=DBT_PROJ_DIR,
    task_id="seed_raw_orders",
    dbt_cmd_flags=["--select", "raw_orders"],
    install_deps=True,
)
```
