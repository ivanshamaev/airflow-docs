# Мультиязычность (Multilanguage)

В Airflow 3 пользователи могут писать **SDK**, позволяющие определять задачи Airflow на языках, отличных от Python. Экспериментальный SDK для Golang доступен в релизе Airflow 3.0. Поддержка нескольких языков важна для снижения привязки к одному языку и для высокоспециализированных, оптимизированных задач. Поддержка других языков также помогает переносить воркфлоу со старых инструментов, где задачи часто написаны не на Python, в Airflow без дорогостоящего рефакторинга.

> Поддержка мультиязычности пока экспериментальная и в разработке. Это руководство может меняться и со временем будет дополняться. Если вы хотите помочь с поддержкой написания задач Airflow на выбранном вами языке, обратитесь к разработчикам Airflow в [Airflow Slack](https://apache-airflow-slack.herokuapp.com/) или в [Airflow Dev list](https://airflow.apache.org/community/).
>
> Совет

## Необходимая база

Чтобы получить максимум от руководства, нужно понимать:

- Основы Golang. См. [документацию Golang](https://go.dev/doc/).
- Основы Airflow. См. [Введение в Apache Airflow](../01.%20astronomer-basic/README.md).

## Как использовать Golang SDK

Golang SDK экспериментальный и в разработке. Актуальный статус: [здесь](https://github.com/apache/airflow/tree/main/go-sdk).

## Другие способы запуска задач на других языках

Запускать задачи на других языках можно и так:

- **KubernetesPodOperator** — запуск любого Docker-образа, в том числе с кодом на любом языке. Подробнее: [Use the KubernetesPodOperator to run a script in another language](kubernetes-pod-operator.md).
- **BashOperator** — запуск скрипта на другом языке. Например, можно запускать скрипты на JavaScript или R. Подробнее: [Run a script in another programming language](../01.%20astronomer-basic/bashoperator.md).

---

[← Логирование](logging.md) | [К содержанию](README.md) | [Динамические DAG →](dynamic-dags.md)
