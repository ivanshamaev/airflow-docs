# Исполнители в Airflow (Executors)

**Executor** (исполнитель) — параметр конфигурации [компонента планировщика Airflow](airflow-components.md). Выбранный исполнитель определяет, **где** и **как** выполняется задача. Можно выбрать один из предустановленных исполнителей под разные сценарии или определить [свой кастомный executor](https://airflow.apache.org/docs/apache-airflow/stable/executor/index.html).

В этом руководстве — какие исполнители доступны в Airflow 3 и как выбрать подходящий.

## Необходимая база

Полезно понимать:

- Компоненты Airflow. См. [Компоненты Airflow](airflow-components.md).
- Основные концепции Airflow. См. [Introduction to Apache Airflow](https://www.astronomer.io/docs/learn/intro-to-airflow).

## Как выбрать исполнителя

Для продакшен-деплойментов Airflow рекомендуются три основных исполнителя:

## AstroExecutor

**AstroExecutor** — проприетарный исполнитель, доступный только пользователям Astro в Airflow 3 Deployments. Использует агентов, управляемых компонентом API server; поддерживает как hosted, так и remote execution в Deployments.

## KubernetesExecutor

**KubernetesExecutor** — контейнерный исполнитель: каждый экземпляр задачи выполняется в отдельном Pod в Kubernetes. На Astro доступен для Deployments в режиме hosted execution в Airflow 2 и Airflow 3.

## CeleryExecutor

**CeleryExecutor** — исполнитель с очередью: задачи отправляются в брокер Celery и забираются Celery workers. На Astro доступен для Deployments в режиме hosted execution в Airflow 2 и Airflow 3.

### AstroExecutor

[AstroExecutor](https://www.astronomer.io/docs/astro/astro-executor) — исполнитель по умолчанию для всех Airflow 3.x Deployments. Использует агентов (воркеров), которые получают задания от компонента API server и выполняют их в подпроцессах. API server управляет жизненным циклом агентов и логикой назначения задач; это надёжнее, чем CeleryExecutor, и задачи стартуют быстрее, чем с KubernetesExecutor.

Этот исполнитель можно использовать для Deployments на Astro как в [hosted](https://www.astronomer.io/docs/astro/execution-mode#hosted-execution), так и в [remote](https://www.astronomer.io/docs/astro/execution-mode#remote-execution) режиме выполнения. Он единственный обеспечивает remote execution на Astro; в других окружениях Airflow для удалённого выполнения можно использовать [EdgeExecutor](https://airflow.apache.org/docs/apache-airflow-providers-edge3/stable/edge_executor.html).

Выбирайте **AstroExecutor** для:

- всех Deployments в режиме remote execution на Astro;
- всех Airflow 3 Deployments на Astro, если нет особых причин использовать другой исполнитель.

С AstroExecutor можно использовать [worker queues](https://www.astronomer.io/docs/astro/configure-worker-queues), чтобы запускать задачи на воркерах разных типов с разной конфигурацией ресурсов.

### KubernetesExecutor

**KubernetesExecutor** — контейнерный исполнитель: каждый экземпляр задачи выполняется в своём Kubernetes Pod. Для его использования нужен доступ к кластеру Kubernetes. KubernetesExecutor даёт полную изоляцию задач и точный контроль над ресурсами на задачу; минус — более долгий старт задачи.

Выбирайте **KubernetesExecutor** для:

- деплойментов, где многим задачам нужна особая конфигурация ресурсов;
- деплойментов с высокими требованиями к изоляции задач.

Дополнительные требования к KubernetesExecutor выполняются автоматически в Astro Deployments, настроенных на этот исполнитель:

- Метаданная БД Airflow не может быть SQLite.
- Должен быть установлен [Airflow Kubernetes provider](https://registry.astronomer.io/providers/apache-airflow-providers-cncf-kubernetes/versions/latest).

Поды Kubernetes можно настраивать: базовая конфигурация и переопределение на уровне задачи через параметр `pod_override`. Подробнее: [Configure tasks to run with the KubernetesExecutor](https://www.astronomer.io/docs/astro/kubernetes-executor) для Astro и [Kubernetes Executor - Configuration](https://airflow.apache.org/docs/apache-airflow-providers-cncf-kubernetes/stable/kubernetes_executor.html#configuration) для self-hosted Airflow.

### CeleryExecutor

**CeleryExecutor** позволяет масштабировать нагрузку по горизонтали: задачи выполняются на нескольких [Celery](https://docs.celeryq.dev/en/latest/getting-started/) workers, забирающих задачи из очереди. Задачи стартуют быстро, так как после первоначальной настройки дополнительная инфраструктура не нужна; можно масштабироваться по горизонтали и выполнять много задач параллельно. Поэтому CeleryExecutor часто выбирают по умолчанию для Airflow 2 Deployments на Astro и для self-managed Airflow 2 и 3.

Выбирайте **CeleryExecutor** для:

- деплойментов с высокой потребностью в горизонтальном масштабировании и параллельном выполнении многих задач;
- деплойментов со стабильной нагрузкой и частым запуском задач;
- всех Airflow 2 Deployments на Astro, если нет особых причин использовать KubernetesExecutor.

Для CeleryExecutor нужны дополнительные компоненты; в Astro Deployments с этим исполнителем они настраиваются автоматически:

- Установленный и настроенный бэкенд Celery: Redis, RabbitMQ или Redis Sentinel. В Astro Deployments используется Redis.
- Должен быть установлен [Airflow Celery provider](https://registry.astronomer.io/providers/apache-airflow-providers-celery/versions/latest).

На Astro с CeleryExecutor можно использовать [worker queues](https://www.astronomer.io/docs/astro/configure-worker-queues) для запуска задач на воркерах разных типов с разной конфигурацией ресурсов.

Подробнее о настройке CeleryExecutor: [Configure the CeleryExecutor](https://www.astronomer.io/docs/astro/celery-executor) для Astro и [Celery Executor](https://airflow.apache.org/docs/apache-airflow-providers-celery/stable/celery_executor.html) для self-managed Airflow.

### LocalExecutor

**LocalExecutor** выполняется в процессе планировщика и в Airflow 3 является самым простым вариантом выполнения задач (SequentialExecutor и DebugExecutor в Airflow 3 удалены). Так как LocalExecutor работает в том же процессе, что и планировщик, дополнительная инфраструктура не нужна; при большом числе задач это может влиять на производительность планировщика.

Выбирайте **LocalExecutor** для:

- лёгких продакшен-окружений при self-managed Airflow;
- локальной разработки и тестов. [Astro CLI](https://www.astronomer.io/docs/astro/cli/overview) использует LocalExecutor.

Переменная конфигурации [`[core].parallelism`](https://airflow.apache.org/docs/apache-airflow/stable/configurations-ref.html#parallelism) задаёт максимальное число задач, которые могут выполняться параллельно с LocalExecutor на один планировщик. Значение должно быть не меньше 1 (по умолчанию: 32).

### Прочие исполнители

В self-managed Airflow доступны и другие исполнители:

- [AWS Lambda Executor](https://airflow.apache.org/docs/apache-airflow-providers-amazon/stable/executors/lambda-executor.html) ([experimental](https://airflow.apache.org/docs/apache-airflow/stable/release-process.html#experimental-features)): **AwsLambdaExecutor** отправляет задачи в AWS Lambda для асинхронного выполнения. Входит в пакет Amazon provider.
- [AWS Batch Executor](https://airflow.apache.org/docs/apache-airflow-providers-amazon/stable/executors/batch-executor.html): **AwsBatchExecutor** выполняет задачи в отдельных контейнерах, планируемых AWS Batch. Входит в пакет Amazon provider.
- [AWS ECS Executor](https://airflow.apache.org/docs/apache-airflow-providers-amazon/stable/executors/ecs-executor.html): **AwsEcsExecutor** — контейнерный исполнитель, каждый экземпляр задачи выполняется в отдельной ECS task. Входит в пакет [Amazon provider](https://airflow.apache.org/docs/apache-airflow-providers-amazon/stable/index.html).
- [EdgeExecutor](https://airflow.apache.org/docs/apache-airflow-providers-edge3/stable/edge_executor.html): позволяет распределять задачи по воркерам в удалённых локациях через HTTP(s). Входит в пакет [Edge3 provider](https://airflow.apache.org/docs/apache-airflow-providers-edge3/stable/index.html) для Airflow 2.10 и выше.

Гибридные исполнители CeleryKubernetesExecutor и LocalKubernetesExecutor в Airflow 3 удалены. В self-managed окружении вместо них можно настроить несколько исполнителей, см. [Configure an executor in self-hosted Airflow](#configure-an-executor-in-self-hosted-airflow).

## Настройка исполнителя на Astro {#configure-an-executor-on-astro}

На Astro исполнитель задаётся при создании Deployment. По умолчанию используется **AstroExecutor**.

При создании Deployment в Astro UI можно дополнительно настроить AstroExecutor (например, worker queues) или выбрать другой исполнитель:

1. Нажмите кнопку **Switch to custom configuration**.
2. Выберите нужный исполнитель.

На Astro Deployment можно создавать и программно, задавая исполнителя параметром конфигурации:

- [Astro Terraform Provider](https://registry.terraform.io/providers/astronomer/astro/latest/docs/resources/deployment): параметр `executor` в ресурсе `astro_deployment`. Варианты: `ASTRO`, `CELERY`, `KUBERNETES`.
- [Astro API](https://www.astronomer.io/docs/astro/astro-api/platform-api-reference/deployment/create-deployment): параметр `executor` в POST-запросе. Варианты: `ASTRO`, `CELERY`, `KUBERNETES`.
- [Astro CLI](https://www.astronomer.io/docs/astro/cli/astro-deployment-create): флаг `--executor`. Варианты: `AstroExecutor`, `CeleryExecutor`, `KubernetesExecutor`.

![Выбор исполнителя в Astro UI](images/airflow-executors-explained_astro_ui_1.png)

![Настройка исполнителя в Astro UI](images/airflow-executors-explained_astro_ui_2.png)

## Настройка исполнителя в self-hosted Airflow {#configure-an-executor-in-self-hosted-airflow}

В open-source Airflow можно настроить [несколько исполнителей](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/executor/index.html#using-multiple-executors-concurrently) и назначать каждой задаче исполнителя через параметр `executor`.

В конфигурационном файле Airflow задайте переменную `[core].executor` списком нужных исполнителей через запятую:

```text
[core]
executor = KubernetesExecutor,CeleryExecutor,MyExecutor:my.custom.module.ExecutorClass
```

Два экземпляра одного и того же исполнителя в одном окружении Airflow использовать нельзя.

В коде задачи исполнитель задаётся параметром `executor`. Первый исполнитель в списке — исполнитель по умолчанию для всех задач, у которых он не указан.

```python
# from airflow.sdk import task

@task(executor="CeleryExecutor")
def my_task():
    pass

# from airflow.providers.standard.operators.bash import BashOperator

BashOperator(
    task_id="my_task",
    executor="MyExecutor",
    bash_command="echo 'Hello World!'",
)
```

---

[← Метаданные БД](airflow-database.md) | [К содержанию](README.md) | [Масштабирование →](scaling-airflow.md)
