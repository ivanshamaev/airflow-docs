# Изолированные окружения (Isolated environments)

![Варианты изоляции окружений](images/airflow-isolated-environments_isolated_env_options_graph.png)

Задачи могут требовать отдельных зависимостей (другая версия Python, пакеты, не совместимые с окружением воркера). Варианты изоляции:

- **PythonVirtualenvOperator** / **@task.virtualenv()** — задача выполняется в виртуальном окружении, созданном на лету (или кэшированном по `venv_cache_path`). Указывается версия Python и список зависимостей. Подходит для изоляции библиотек внутри того же образа.
- **KubernetesPodOperator** / **@task.kubernetes()** — задача запускается в отдельном Pod с выбранным образом. Полная изоляция: свой образ, ресурсы, переменные окружения. См. [KubernetesPodOperator](kubernetes-pod-operator.md).
- **DockerOperator** / **@task.docker()** — задача в отдельном контейнере на том же хосте. Меньше изоляции, чем Kubernetes, но проще при одном хосте.

Выбор зависит от инфраструктуры (K8s vs один сервер), требований к воспроизводимости и скорости старта. Для тяжёлых или разнородных зависимостей чаще используют KubernetesPodOperator.

Подробнее: [Isolated environments](https://www.astronomer.io/docs/learn/airflow-isolated-environments), [Virtualenv](https://www.astronomer.io/events/webinars/running-airflow-tasks-in-isolated-environments/).

---

[← Human-in-the-loop](human-in-the-loop.md) | [К содержанию](README.md) | [KubernetesPodOperator →](kubernetes-pod-operator.md)
