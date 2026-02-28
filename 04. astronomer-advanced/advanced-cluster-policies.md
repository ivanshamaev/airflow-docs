# Расширенные политики кластера (Cluster policies)

**Cluster policies** — набор правил, которые администратор задаёт для изменения или проверки ключевых объектов Airflow: переменные контекста, Pod, Task Instance, Task, DAG. Политики позволяют централизованно управлять тем, как пользователи и DAG взаимодействуют с кластером (теги, очереди, лимиты ресурсов, retries и т.д.).

> В отличие от плагинов, политики кластера **не отображаются в UI**. Рекомендуется логировать каждое изменение объекта политикой.

Типы политик: **DAG policy** (при загрузке DAG из DagBag), **Task policy** (при создании задачи при парсинге), **Task Instance policy** (перед постановкой task instance в очередь, через `task_instance_mutation_hook`), **Pod policy** (для Pod при KubernetesPodOperator/KubernetesExecutor, через `pod_mutation_hook`, Airflow 2.6+).

![Схема типов политик кластера](images/airflow-advanced-cluster-policies_diagram.png)

При несоответствии политике можно вызвать `AirflowClusterPolicyViolation` — DAG не загрузится, в UI будет import error. Исключение `AirflowClusterPolicySkipDag` пропускает DAG без отображения в UI (например, при миграции или по тегу окружения).

Реализация через **pluggy** (Airflow 2.6+): пакет с entry point `airflow.policy`, функции `dag_policy(dag)`, `task_policy(task)`, `task_instance_mutation_hook(task_instance)`, `pod_mutation_hook(pod)`. Для версий ниже 2.6 политики задаются в `config/airflow_local_settings.py` в `$AIRFLOW_HOME`. На Astro поддерживается только pluggy. Подключение: добавить пакет в `plugins/`, в Dockerfile — `COPY plugins plugins` и `RUN pip install ./plugins`.

Подробнее: [Cluster policies](https://www.astronomer.io/docs/learn/airflow-advanced-cluster-policies), [Airflow docs](https://airflow.apache.org/docs/apache-airflow/stable/administration-and-deployment/cluster-policies.html).

---

[← К содержанию](README.md) | [MLOps →](airflow-mlops.md)
