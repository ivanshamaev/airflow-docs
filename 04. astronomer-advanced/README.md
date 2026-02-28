# Airflow: продвинутые темы — перевод на русский

Русский перевод разделов документации [Astronomer.io Docs](https://www.astronomer.io/docs/learn/), посвящённых продвинутым возможностям Airflow.

Источник: [Astronomer Learn](https://www.astronomer.io/docs/learn/). Перевод неофициальный, для личного использования.

## Содержание

| № | Страница | Описание |
|---|----------|----------|
| 01 | [Расширенные политики кластера](advanced-cluster-policies.md) | DAG/task/pod policy, pluggy, AirflowClusterPolicyViolation |
| 02 | [Airflow для MLOps](airflow-mlops.md) | Оркестрация ML/LLM пайплайнов, компоненты MLOps, интеграции |
| 03 | [Плагины Airflow](airflow-plugins.md) | AirflowPlugin, внешние представления, React, FastAPI, макросы |
| 04 | [Пуллы (Pools)](airflow-pools.md) | Ограничение параллелизма, слоты, приоритеты |
| 05 | [Кастомные XCom backend](custom-xcom-backends.md) | Object Storage backend, своя сериализация |
| 06 | [Deferrable-операторы](deferrable-operators.md) | Триггеры, освобождение воркера, BaseTrigger |
| 07 | [Event-driven планирование](event-driven-scheduling.md) | Очереди сообщений, AssetWatcher, SQS, Kafka |
| 08 | [Human-in-the-loop](human-in-the-loop.md) | Ожидание решения пользователя в пайплайне |
| 09 | [Изолированные окружения](isolated-environments.md) | virtualenv, KubernetesPodOperator, Docker |
| 10 | [KubernetesPodOperator](kubernetes-pod-operator.md) | Запуск задачи в отдельном Pod |
| 11 | [Логирование](logging.md) | Настройка логов, remote logging, формат |
| 12 | [Мультиязычность](multilanguage.md) | SDK для других языков, Go, BashOperator |
| 13 | [Динамическая генерация DAG](dynamic-dags.md) | Программное создание DAG |
| 14 | [Setup и teardown](setup-teardown.md) | Блоки setup/teardown для ресурсов |
| 15 | [Общий код между проектами](sharing-code.md) | /include, Python-пакет, несколько репозиториев |
| 16 | [Синхронное выполнение DAG](synchronous-execution.md) | API wait, ожидание завершения и XCom |
| 17 | [Тестирование Airflow](testing-airflow.md) | Тесты DAG, задач, pytest |

---

*Документация ориентирована на Airflow 3.x. Импорты и API могут отличаться в других версиях.*
