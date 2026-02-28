# KubernetesPodOperator

**KubernetesPodOperator** запускает задачу в отдельном Pod в кластере Kubernetes. Схема подключения к кластеру:

![Подключение к кластеру Kubernetes](images/kubernetes_cluster_connection.png)

Каждый task instance получает свой Pod; после выполнения Pod удаляется. Полная изоляция и контроль ресурсов (CPU, memory, GPU), образ и переменные окружения задаются в операторе или через **pod_override** на уровне задачи.

Основные параметры: `namespace`, `image`, `name` (имя пода), `cmds`/`arguments`, `env_vars`, `config_file`, `cluster_context`, `in_cluster` (использовать ли in-cluster config). Ресурсы задаются через `container_resources` или в `pod_override` (V1ResourceRequirements). Для доступа к образам из приватного registry настраивается `image_pull_secrets`. Логи можно собирать в сторэдж или оставлять в Pod (зависит от конфигурации логирования Airflow).

На Astro при использовании KubernetesExecutor или KubernetesPodOperator провайдер [Kubernetes](https://registry.astronomer.io/providers/apache-airflow-providers-cncf-kubernetes/versions/latest) устанавливается и настраивается в окружении. Запуск скрипта на другом языке: указать образ с нужным рантаймом и передать команду/скрипт через `cmds` и `arguments` или смонтировать конфиг через Volume.

Подробнее: [KubernetesPodOperator](https://www.astronomer.io/docs/learn/kubepod-operator), [Kubernetes Executor (Astro)](https://www.astronomer.io/docs/astro/kubernetes-executor).

---

[← Изолированные окружения](isolated-environments.md) | [К содержанию](README.md) | [Логирование →](logging.md)
