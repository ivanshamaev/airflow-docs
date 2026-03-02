# Airflow Concepts: Basics — перевод на русский

Русский перевод раздела **Airflow concepts: Basics** с сайта [Astronomer.io Docs](https://www.astronomer.io/docs/learn/).

Источник: [Astronomer Learn — Airflow concepts](https://www.astronomer.io/docs/learn/). Перевод неофициальный, для личного использования.

## Содержание

| Страница | Описание |
|----------|----------|
| [Введение в Airflow](intro-to-airflow.md) | Концепции Airflow, запуск, сценарии использования, основы пайплайнов |
| [Интерфейс Airflow (Airflow UI)](airflow-ui.md) | Обзор веб-интерфейса: представления, дашборд, DAG, ассеты, админка |
| [Ассеты и data-aware планирование (Assets)](assets.md) | Зависимости DAG по данным, расписание по обновлению ассетов |
| [BashOperator](bashoperator.md) | Выполнение bash-команд и скриптов в DAG |
| [Подключения (Connections)](connections.md) | Настройка и управление подключениями к внешним системам |
| [DAG](dags.md) | Введение в DAG: параметры, запуски, граф, версии |
| [Выполнение SQL (Execute SQL)](execute-sql.md) | Операторы и практики для выполнения SQL из DAG |
| [Хуки (Hooks)](hooks.md) | Абстракции для работы с внешними API в задачах |
| [Управление кодом Airflow](managing-airflow-code.md) | Структура проекта, разделение проектов, переиспользование кода |
| [Операторы (Operators)](operators.md) | Основы операторов, примеры, лучшие практики |
| [Планирование в Airflow (Scheduling)](scheduling.md) | Расписания, метки времени, cron, ассеты, расписания по событиям |
| [Сенсоры (Sensors)](sensors.md) | Ожидание условий, режимы poke/reschedule, отложенные сенсоры |
| [Зависимости задач (Task dependencies)](task-dependencies.md) | Связи между задачами, chain, task groups, trigger rules |
| [Правила срабатывания (Trigger rules)](trigger-rules.md) | Когда задача запускается относительно вышестоящих |
| [Переменные (Variables)](variables.md) | Создание и использование переменных Airflow |

---

*Документация основана на Airflow 3.x (в т.ч. 3.1). Некоторые элементы могут отличаться в других версиях.*
