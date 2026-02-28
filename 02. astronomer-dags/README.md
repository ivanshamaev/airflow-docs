# Airflow DAGs — перевод на русский

Русский перевод разделов документации [Astronomer.io Docs](https://www.astronomer.io/docs/learn/), посвящённых написанию и отладке DAG.

Источник: [Astronomer Learn](https://www.astronomer.io/docs/learn/). Перевод неофициальный, для личного использования.

## Содержание

| № | Страница | Описание |
|---|----------|----------|
| 01 | [Контекст Airflow (Airflow context)](airflow-context.md) | Словарь контекста задачи: ti, dag_run, params, шаблоны времени |
| 02 | [Декораторы Airflow (Airflow decorators)](airflow-decorators.md) | TaskFlow API: @task, @dag, передача данных, смешение с операторами |
| 03 | [Уведомления об ошибках (Error notifications)](airflow-notifications.md) | Настройка оповещений при сбоях задач и DAG |
| 04 | [Параметры (Params)](airflow-params.md) | DAG- и task-level параметры, переопределение при запуске |
| 05 | [BranchOperator и ветвление](branch-operator.md) | Условные ветви в DAG, @task.branch, ShortCircuit |
| 06 | [Зависимости между DAG (Cross-DAG)](cross-dag-dependencies.md) | ExternalTaskSensor, триггер по ассетам, зависимость от другого DAG |
| 07 | [Кастомные хуки и операторы](custom-hooks-operators.md) | Создание и подключение своих хуков и операторов |
| 08 | [Лучшие практики написания DAG](dag-best-practices.md) | Идемпотентность, размер данных, структура кода |
| 09 | [Параметры DAG (DAG parameters)](dag-parameters.md) | Полный список параметров DAG и их назначение |
| 10 | [Версионирование DAG и DAG bundles](dag-versioning.md) | Версии DAG, каталоги загрузки, .airflowignore |
| 11 | [Отладка DAG](debugging-dags.md) | Локальный тест, логи, шаблоны, проверка перед деплоем |
| 12 | [Динамические задачи (Dynamic tasks)](dynamic-tasks.md) | Dynamic task mapping: expand, partial, маппинг по результату |
| 13 | [Jinja-шаблонирование](jinja-templating.md) | Шаблонируемые поля, макросы, контекст в шаблонах |
| 14 | [Передача данных между задачами (XCom)](passing-data-between-tasks.md) | XCom, размер данных, промежуточное хранилище (S3 и др.) |
| 15 | [Повторный запуск DAG (Rerunning, Backfill)](rerunning-dags.md) | Ручной перезапуск, backfill, очистка задач |
| 16 | [Группы задач (Task groups)](task-groups.md) | @task_group, вложенные группы, динамический маппинг групп |

---

*Документация ориентирована на Airflow 3.x. Импорты и API могут отличаться в других версиях.*
