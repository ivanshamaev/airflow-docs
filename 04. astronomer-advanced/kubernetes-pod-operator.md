# KubernetesPodOperator

> Эта страница ещё не обновлена под Airflow 3. Показанные концепции актуальны, но часть кода может потребовать правок. При запуске примеров обновите импорты и учтите возможные breaking changes.
>
> Информация

**KubernetesPodOperator (KPO)** запускает Docker-образ в отдельном Pod в Kubernetes. Оператор абстрагирует вызовы Kubernetes API и позволяет запускать Pod’ы из Airflow через код DAG.

В этом руководстве:

- Чем отличаются KubernetesPodOperator и Kubernetes executor.
- Как настраивать KubernetesPodOperator.
- Когда использовать KubernetesPodOperator.
- Что нужно для запуска KubernetesPodOperator.

Также рассмотрены: запуск задачи на языке, отличном от Python, работа KubernetesPodOperator с XCom и запуск Pod в удалённом кластере AWS EKS.

> На Astro вся инфраструктура для работы KubernetesPodOperator размещена у Astronomer и управляется автоматически. Поэтому часть сценариев на этой странице может быть проще, если вы запускаете KubernetesPodOperator на Astro. Подробнее: [Run the KubernetesPodOperator on Astro](https://www.astronomer.io/docs/astro/kubernetespodoperator).
>
> Совет

> - Вебинар: [Running Airflow Tasks in Isolated Environments](https://www.astronomer.io/events/webinars/running-airflow-tasks-in-isolated-environments/).
> - Astronomer Academy: модуль [Airflow: The KubernetesPodOperator](https://academy.astronomer.io/astro-runtime-the-kubernetespodoperator-1).
>
> По теме есть и другие материалы. См. также:
>
> Другие способы изучить тему

## Необходимая база

Чтобы получить максимум от руководства, нужно понимать:

- Основы Kubernetes. См. [Kubernetes Documentation](https://kubernetes.io/docs/home/).
- Операторы Airflow. См. [Операторы 101](https://www.astronomer.io/docs/learn/what-is-an-operator).

## Предварительные требования

Для использования KubernetesPodOperator нужно установить пакет провайдера Kubernetes. Установка через pip:

```bash
pip install apache-airflow-providers-cncf-kubernetes==<version>
```

Если вы используете [Astro CLI](https://www.astronomer.io/docs/astro/cli/overview), пакет можно добавить в проект Astro строкой:

```text
apache-airflow-providers-cncf-kubernetes==<version>
```

Проверьте [документацию провайдера Kubernetes для Airflow](https://airflow.apache.org/docs/apache-airflow-providers-cncf-kubernetes/stable/index.html#requirements), чтобы установить версию провайдера, совместимую с вашей версией Airflow.

Также нужен существующий кластер Kubernetes для подключения. Обычно это тот же кластер, на котором запущен Airflow, но не обязательно.

Использовать Kubernetes executor для KubernetesPodOperator не обязательно. Можно выбрать один из исполнителей:

- CeleryKubernetes executor
- Kubernetes executor
- Celery executor
- LocalKubernetes executor
- Local executor

На Astro инфраструктура для запуска KubernetesPodOperator с Celery executor по умолчанию входит во все кластеры. Подробнее: [Run the KubernetesPodOperator on Astro](https://www.astronomer.io/docs/astro/kubernetespodoperator).

### Запуск KubernetesPodOperator локально

Настройка локального окружения для KubernetesPodOperator помогает избежать долгих деплоев в удалённые среды.

Ниже — быстрая настройка локального окружения для KubernetesPodOperator с помощью [Astro CLI](https://www.astronomer.io/docs/astro/cli/overview). Альтернатива — [Helm Chart для Apache Airflow](https://airflow.apache.org/docs/helm-chart/stable/index.html) для запуска open source Airflow в локальном кластере Kubernetes. См. [Getting Started With the Official Airflow Helm Chart](https://www.youtube.com/watch?v=39k2Sz9jZ2c&ab_channel=Astronomer).

#### Шаг 1: Настройка Kubernetes

**Windows и Mac**

В последних версиях Docker для Windows и Mac можно запустить одноузловой кластер Kubernetes локально. Для Windows: [Setting Up Docker for Windows and WSL to Work Flawlessly](https://nickjanetakis.com/blog/setting-up-docker-for-windows-and-wsl-to-work-flawlessly). Для Mac: [Docker Desktop for Mac user manual](https://nickjanetakis.com/blog/setting-up-docker-for-windows-and-wsl-to-work-flawlessly). Устанавливать Docker Compose не обязательно.

1. В диалоге установки Kubernetes нажмите Install.
2. Нажмите Apply and Restart.
3. Включите опцию Enable Kubernetes.
4. Откройте Docker → Settings → Kubernetes.

Docker перезапустится, индикатор статуса станет зелёным — Kubernetes запущен.

**Linux**

1. Установите Microk8s. См. [Microk8s](https://microk8s.io/).
2. Выполните `microk8s.start`, чтобы запустить Kubernetes.

#### Шаг 2: Обновление файла kubeconfig

**Windows и Mac**

1. Скопируйте контекст `docker-desktop` из конфигурации Kubernetes и сохраните его в папку `/include/.kube/` вашего проекта Astro. В файле `config` хранится информация, которую KubernetesPodOperator использует для подключения к кластеру.

```bash
kubectl config use-context docker-desktop
kubectl config view --minify --raw > <каталог проекта Astro>/include/.kube
```

После выполнения команд в папке `/include/.kube/` появится файл `config`, по структуре похожий на пример ниже:

```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: <certificate-authority-data>
    server: https://kubernetes.docker.internal:6443/
  name: docker-desktop
contexts:
- context:
    cluster: docker-desktop
    user: docker-desktop
  name: docker-desktop
current-context: docker-desktop
kind: Config
preferences: {}
users:
- name: docker-desktop
  user:
    client-certificate-data: <client-certificate-data>
    client-key-data: <client-key-data>
```

2. (Опционально) Добавьте папку `.kube` в `.dockerignore`, чтобы исключить её из Docker-образа.
3. (Опционально) Добавьте папку `.kube` в `.gitignore`, если проект Astro в репозитории GitHub и вы не хотите, чтобы файл отслеживался системой контроля версий.
4. При проблемах с подключением проверьте настройку сервера в файле kubeconfig. Если указано `server: https://localhost:6445`, замените на `server: https://kubernetes.docker.internal:6443` для доступа к localhost с запущенными Pod’ами Kubernetes. Если не поможет, попробуйте `server: https://host.docker.internal:6445`.

**Linux**

В папке `.kube` проекта Astro создайте файл config:

```bash
microk8s.config > /include/.kube/config
```

#### Шаг 3: Создание подключения Kubernetes в UI Airflow

Чтобы запускать Pod Kubernetes локально, можно использовать JSON-шаблон ниже для создания строки подключения и создания подключения Kubernetes в локальном UI Airflow. Сначала подставьте в шаблон значения, полученные на предыдущих шагах:

```json
{
    "apiVersion": "v1",
    "clusters": [
        {
            "cluster": {
                "certificate-authority-data": "<certificate-authority-data>",
                "server": "https://kubernetes.docker.internal:6443"
            },
            "name": "docker-desktop"
        }
    ],
    "contexts": [
        {
            "context": {
                "cluster": "docker-desktop",
                "user": "docker-desktop"
            },
            "name": "docker-desktop"
        }
    ],
    "current-context": "docker-desktop",
    "kind": "Config",
    "preferences": {},
    "users": [
        {
            "name": "docker-desktop",
            "user": {
                "client-certificate-data": "<client-certificate-data>",
                "client-key-data": "<client-key-data>"
            }
        }
    ]
}
```

Затем выполните `astro dev start` в Astro CLI, чтобы поднять локальное окружение Airflow. После создания окружения откройте UI управления подключениями, создайте новое подключение типа **Kubernetes Cluster Connection**. В форме создания скопируйте созданный по шаблону JSON в поле **Kube config (JSON format)** и сохраните подключение с connection id `k8s_conn`. Если нужен другой connection id, измените его в примере DAG ниже.

#### Шаг 4: Запуск контейнера

Чтобы использовать KubernetesPodOperator, нужно задать конфигурацию каждой задачи и Pod Kubernetes, в котором она выполняется, включая namespace и Docker-образ.

В примере ниже DAG запускает образ `hello-world` через подключение `k8s_conn`, созданное на предыдущем шаге, в локальном кластере Kubernetes.

```python
from pendulum import datetime
from airflow import DAG
from airflow.providers.cncf.kubernetes.operators.kubernetes_pod import (
    KubernetesPodOperator,
)

with DAG(
    dag_id="example_kubernetes_pod",
    schedule="@once",
    start_date=datetime(2023, 3, 30),
) as dag:
    example_kpo = KubernetesPodOperator(
        kubernetes_conn_id="k8s_conn",
        image="hello-world",
        name="airflow-test-pod",
        task_id="task-one",
        is_delete_operator_pod=True,
        get_logs=True,
    )

    example_kpo
```

#### Шаг 4: Просмотр логов Kubernetes

(Опционально) Для проверки логов Pod’ов, созданных оператором, и отладки можно использовать утилиту `kubectl`. Если `kubectl` не установлена, см. [Install Tools](https://kubernetes.io/docs/tasks/tools/#kubectl).

**Windows и Mac**

Выполните `kubectl get pods -n $namespace` или `kubectl logs {pod_name} -n $namespace`, чтобы посмотреть логи только что запущенного Pod’а. По умолчанию `docker-for-desktop` запускает Pod’ы в namespace `default`.

**Linux**

Выполните `microk8s.kubectl get pods -n $namespace` или `microk8s.kubectl logs {pod_name} -n $namespace`, чтобы посмотреть логи только что запущенного Pod’а. По умолчанию `microk8s` запускает Pod’ы в namespace `default`.

## Когда использовать KubernetesPodOperator

KubernetesPodOperator запускает любой переданный ему Docker-образ. Типичные сценарии:

- Задачи с ограничениями по Node (виртуальная или физическая машина в Kubernetes), например только на узлах в ЕС.
- Задачи на версии Python, не поддерживаемой окружением Airflow.
- Выполнение задач в отдельном окружении со своими пакетами и зависимостями.
- Полный контроль над объёмом вычислительных ресурсов и памяти для одной задачи.
- Запуск задачи на языке, отличном от Python. В руководстве есть пример запуска скрипта на Haskell с помощью KubernetesPodOperator.

### Сравнение KubernetesPodOperator и Kubernetes executor

[Исполнители (executors)](https://www.astronomer.io/docs/learn/airflow-executors-explained) определяют, как выполняются задачи Airflow. И Kubernetes executor, и KubernetesPodOperator динамически создают и завершают Pod’ы для выполнения задач. Kubernetes executor влияет на выполнение **всех** задач в инстансе Airflow. KubernetesPodOperator запускает **только свою** задачу в отдельном Pod с собственной конфигурацией и не влияет на остальные задачи.

Основные отличия:

- Если в Kubernetes executor в контейнер `base` (через `pod_template_file` или ключ `pod_override` в словаре аргумента `executor_config`) передан кастомный Docker-образ, в нём должен быть установлен Airflow, иначе задача не выполнится. Кастомный образ может понадобиться, например, для другой версии пакетов. С KubernetesPodOperator такого требования нет — он может запускать любой валидный Docker-образ.
- У Kubernetes executor меньше абстракций над конфигурацией Pod: все настройки на уровне задачи передаются исполнителю словарём через аргумент `executor_config` у `BaseOperator` (доступен всем операторам).
- KubernetesPodOperator описывает одну изолированную задачу Airflow. Kubernetes executor настраивается на уровне инстанса Airflow — тогда все задачи выполняются в своих Pod’ах. Это удобно при автоскейлинге, но не всегда оптимально при большом количестве коротких задач.
- Для KubernetesPodOperator нужно указывать Docker-образ; для Kubernetes executor — нет.

И KubernetesPodOperator, и Kubernetes executor могут использовать Kubernetes API для создания Pod’ов. Обычно KubernetesPodOperator удобен для контроля окружения задачи, а Kubernetes executor — для оптимизации ресурсов. Часто оба используются в одном инстансе Airflow: все задачи идут в Kubernetes, а дополнительная конфигурация окружения нужна только части задач.

## Настройка KubernetesPodOperator

KubernetesPodOperator запускает любой переданный ему валидный Docker-образ в отдельном Pod в кластере Kubernetes. Оператор поддерживает аргументы для типичных настроек Pod. Для сложных сценариев можно указать [Pod template file](https://kubernetes.io/docs/concepts/workloads/pods/#pod-templates) со всеми возможными настройками Pod.

KubernetesPodOperator создаётся как любой другой оператор в контексте DAG.

### Обязательные аргументы

- **`image`**: Docker-образ для запуска. Образы с [hub.docker.com](https://hub.docker.com/) можно передавать по имени; для кастомных репозиториев нужен полный URL.
- **`name`**: имя создаваемого Pod’а. В рамках namespace имя должно быть уникальным.
- **`namespace`**: namespace в кластере Kubernetes, в который помещается новый Pod.
- **`task_id`**: уникальная строка, идентифицирующая задачу в Airflow.

### Необязательные аргументы

- **`container_resources`**: объект [`k8s.V1ResourceRequirements`](https://github.com/kubernetes-client/python/blob/master/kubernetes/docs/V1ResourceRequirements) с запросами и/или лимитами ресурсов для Pod’а.
- **`env_vars`**: словарь переменных окружения для Pod’а.
- **`log_events_on_failure`**: логировать ли события при падении Pod’а. По умолчанию `False`.
- **`get_logs`**: использовать ли stdout контейнера как логи задачи в системе логирования Airflow.
- **`is_delete_operator_pod`**: удалять ли Pod по достижении финального состояния или при прерывании выполнения. По умолчанию `True`.
- **`reattach_on_restart`**: поведение при потере воркера во время работы Pod’а. При `True` существующий Pod переподключается к воркеру при следующей попытке. При `False` при каждой попытке создаётся новый Pod. По умолчанию `True`.
- **`ports`**: порты для Pod’а.
- **`labels`**: список пар ключ–значение для логической группировки объектов.
- **`random_name_suffix`**: при `True` к имени Pod’а добавляется случайный суффикс. Помогает избежать конфликтов имён при большом числе Pod’ов.

```python
# from airflow.providers.cncf.kubernetes.operators.kubernetes_pod import (
#     KubernetesPodOperator,
# )
# from kubernetes.client import CoreV1Api, V1Pod, models as k8s

KubernetesPodOperator(
    # другие аргументы
    container_resources=k8s.V1ResourceRequirements(
        requests={"cpu": "100m", "memory": "64Mi", "ephemeral-storage": "1Gi"},
        limits={"cpu": "200m", "memory": "420Mi", "ephemeral-storage": "2Gi"},
    )
)
```

Подробнее: [Kubernetes Documentation on Resource Management for Pods and Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/).

> Компоненты подключения можно задать или переопределить на уровне задачи аргументами `config_file` (путь к файлу KubeConfig) и `cluster_context`. Задание этих параметров в `airflow.cfg` устарело.
>
> Информация

### Запуск Pod’ов во внешних кластерах

На изображении ниже показано, как настроить подключение к кластеру Kubernetes в UI Airflow.

![Подключение к кластеру Kubernetes](images/kubernetes_cluster_connection.png)

Для работы подключения нужны:

- Контекст кластера из предоставленного файла KubeConfig.
- Файл KubeConfig — путь к файлу или данные в формате JSON.

Если Airflow не запущен в Kubernetes или нужно отправлять Pod в другой кластер, отличный от того, где работает Airflow, можно создать [подключение](https://www.astronomer.io/docs/learn/connections) типа Kubernetes Cluster. Оно использует [Kubernetes hook](https://registry.astronomer.io/providers/kubernetes/modules/kuberneteshook) для доступа к [Kubernetes API](https://kubernetes.io/docs/reference/kubernetes-api/) другого кластера. Подключение передаётся в KubernetesPodOperator через аргумент `kubernetes_conn_id`.

При `in_cluster=True` для подключения к кластеру достаточно указать аргумент `namespace` у KubernetesPodOperator. Pod будет запущен в том же кластере Kubernetes, что и инстанс Airflow.

### Настройка подключения Kubernetes

С шаблонами Jinja можно использовать аргументы KubernetesPodOperator: `image`, `cmds`, `arguments`, `env_vars`, `labels`, `config_file`, `pod_template_file`, `namespace`.

Дополнительные аргументы для настройки Pod и передачи данных в Docker-образ см. в [исходном коде KubernetesPodOperator](https://github.com/apache/airflow/blob/main/providers/cncf/kubernetes/src/airflow/providers/cncf/kubernetes/operators/pod.py):

- **`full_pod_spec`**: полная конфигурация Pod в виде Python-объекта `k8s`.
- **`pod_template_file`**: путь к файлу шаблона Pod.
- **`affinity`** и **`tolerations`**: правила [назначения Pod узлам (Pod to Node)](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/). Как и `volumes`, требуют объект `k8s`.
- **`volumes`**: список `k8s.V1Volumes`; пример: [Kubernetes example DAG](https://github.com/apache/airflow/blob/providers-cncf-kubernetes/10.0.0/providers/tests/system/cncf/kubernetes/example_kubernetes.py).

Клиенты Astronomer могут задать запросы и лимиты ресурсов по умолчанию для всех задач KPO в настройках деплоя: [Configure Kubernetes Pod resources](https://www.astronomer.io/docs/astro/deployment-resources#configure-kubernetes-pod-resources). Задание аргумента `container_resources` в задаче KPO переопределяет эти настройки. Использование `ephemeral-storage` для Astro Hosted сейчас в [Public Preview](https://astronomer.io/docs/astro/feature-previews).

> Информация

Если части задач нужны особые ресурсы (например, GPU), их можно запускать в кластере, отличном от того, где работает Airflow.

Способ подключения к внешнему кластеру зависит от того, где размещён кластер и где — окружение Airflow, но в общем случае для запуска Pod во внешнем кластере нужно:

- Передать конфигурацию кластера задачам KubernetesPodOperator через настройки на уровне задачи или через подключение Kubernetes.
- Дать окружению Airflow права на создание Pod’ов во внешнем кластере.
- Обеспечить сетевую связность окружения Airflow с внешним кластером.

Подробный пример настройки задачи KubernetesPodOperator для запуска Pod во внешнем кластере EKS: [документация Astro](https://www.astronomer.io/docs/astro/kubernetespodoperator).

## Использование декоратора @task.kubernetes

Декоратор `@task.kubernetes` — альтернатива классическому KubernetesPodOperator при запуске Python-скриптов в отдельном Pod Kubernetes. Docker-образ, передаваемый в `@task.kubernetes`, должен уметь выполнять Python-скрипты.

Как и у обычных функций с декоратором `@task`, в Python-скрипт, выполняющийся в выделенном Pod, можно передавать XCom. Если в параметрах декоратора задано `do_xcom_push=True`, возвращаемое значение функции попадает в XCom. Подробнее о декораторах: [Введение в декораторы Airflow](https://www.astronomer.io/docs/learn/airflow-decorators).

Astronomer рекомендует использовать декоратор `@task.kubernetes` вместо KubernetesPodOperator при работе с XCom и Python-скриптами в отдельном Pod Kubernetes.

```python
from pendulum import datetime
from airflow.configuration import conf
from airflow.decorators import dag, task
import random

# текущий namespace Kubernetes, в котором запущен Airflow
namespace = conf.get("kubernetes", "NAMESPACE")

@dag(
    start_date=datetime(2023, 1, 1),
    catchup=False,
    schedule="@daily",
)
def kubernetes_decorator_example_dag():
    @task
    def extract_data():
        # имитация запроса к БД
        data_point = random.randint(0, 100)
        return data_point

    @task.kubernetes(
        # Docker-образ для запуска, должен уметь выполнять Python-скрипт
        image="python",
        # запуск Pod в том же кластере, где работает Airflow
        in_cluster=True,
        # запуск Pod в том же namespace, где работает Airflow
        namespace=namespace,
        # конфигурация Pod
        name="my_pod",
        get_logs=True,
        log_events_on_failure=True,
        do_xcom_push=True,
    )
    def transform(data_point):
        multiplied_data_point = 23 * int(data_point)
        return multiplied_data_point

    @task
    def load_data(**context):
        transformed_data_point = context["ti"].xcom_pull(
            task_ids="transform", key="return_value"
        )
        print(transformed_data_point)

    load_data(transform(extract_data()))

kubernetes_decorator_example_dag()
```

## Пример: запуск скрипта на другом языке с KubernetesPodOperator

Частый сценарий — выполнение задачи на языке, отличном от Python. Для этого собирают кастомный Docker-образ со скриптом.

В примере ниже скрипт на Haskell выводит в консоль значение переменной `NAME_TO_GREET`:

```haskell
import System.Environment

main = do
        name <- getEnv "NAME_TO_GREET"
        putStrLn ("Hello, " ++ name)
```

Dockerfile подготавливает окружение и запускает скрипт командой `CMD`:

```dockerfile
FROM haskell
WORKDIR /opt/hello_name
RUN cabal update
COPY ./haskell_example.cabal /opt/hello_name/haskell_example.cabal
RUN cabal build --only-dependencies -j4
COPY . /opt/hello_name
RUN cabal install
CMD ["haskell_example"]
```

После публикации образа его можно запустить из KubernetesPodOperator через аргумент `image`. В примере DAG ниже показаны разные аргументы оператора, в том числе передача `NAME_TO_GREET` в код на Haskell.

```python
from airflow import DAG
from pendulum import datetime
from airflow.providers.cncf.kubernetes.operators.kubernetes_pod import (
    KubernetesPodOperator,
)
from airflow.configuration import conf

namespace = conf.get("kubernetes", "NAMESPACE")
name = "your_name"

with DAG(
    start_date=datetime(2022, 6, 1),
    catchup=False,
    schedule="@daily",
    dag_id="KPO_different_language_example_dag",
) as dag:
    say_hello_name_in_haskell = KubernetesPodOperator(
        task_id="say_hello_name_in_haskell",
        image="<image location>",
        in_cluster=True,
        namespace=namespace,
        name="my_pod",
        random_name_suffix=True,
        labels={"app": "backend", "env": "dev"},
        reattach_on_restart=True,
        is_delete_operator_pod=True,
        get_logs=True,
        log_events_on_failure=True,
        env_vars={"NAME_TO_GREET": f"{name}"},
    )
```

## Пример: KubernetesPodOperator и XCom

[XCom](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/xcoms.html) в Airflow часто используется для передачи небольших объёмов данных между задачами. KubernetesPodOperator может и получать значения из XCom, и записывать их в XCom.

В примере ниже — ETL-пайплайн: задача `extract_data` выполняет запрос к БД и возвращает значение. [TaskFlow API](https://airflow.apache.org/docs/apache-airflow/stable/tutorial_taskflow_api.html#tutorial-on-the-taskflow-api) автоматически отправляет возвращаемое значение в XCom.

Задача `transform` — KubernetesPodOperator: ей нужны данные из XCom от вышестоящей задачи, после чего запускается образ, собранный по следующему Dockerfile:

```dockerfile
FROM python

WORKDIR /

RUN mkdir -p airflow/xcom
RUN echo "" > airflow/xcom/return.json

COPY multiply_by_23.py ./

CMD ["python", "./multiply_by_23.py"]
```

При использовании XCom с KubernetesPodOperator в Docker-контейнере нужно создать файл `airflow/xcom/return.json` (лучше в Dockerfile), так как Airflow ищет данные для XCom только по этому пути. В примере ниже образ содержит простой Python-скрипт: умножает переменную окружения на 23, упаковывает результат в JSON и записывает в нужный файл для считывания как XCom. XCom от KubernetesPodOperator записывается только при успешном завершении задачи.

```python
import os

data_point = os.environ["DATA_POINT"]

multiplied_data_point = str(23 * int(data_point))
return_json = {"return_value": f"{multiplied_data_point}"}

f = open("./airflow/xcom/return.json", "w")
f.write(f"{return_json}")
f.close()
```

Задача `load_data` забирает XCom из задачи `transform` и выводит значение в консоль.

Полный код DAG — в примерах ниже. Чтобы задача не падала, включайте `do_xcom_push` только после создания файла `airflow/xcom/return.json` внутри Docker-контейнера, запускаемого KubernetesPodOperator.

**Taskflow**

```python
from pendulum import datetime
from airflow.providers.cncf.kubernetes.operators.kubernetes_pod import (
    KubernetesPodOperator,
)
from airflow.configuration import conf
from airflow.decorators import dag, task

import random

namespace = conf.get("kubernetes", "NAMESPACE")

@dag(
    start_date=datetime(2022, 6, 1),
    catchup=False,
    schedule="@daily",
)
def KPO_XComs_example_dag():
    @task
    def extract_data():
        data_point = random.randint(0, 100)
        return data_point

    transform = KubernetesPodOperator(
        task_id="transform",
        image="<image location>",
        in_cluster=True,
        namespace=namespace,
        name="my_pod",
        get_logs=True,
        log_events_on_failure=True,
        env_vars={
            "DATA_POINT": """{{ ti.xcom_pull(task_ids='extract_data',
                                                 key='return_value') }}"""
        },
        do_xcom_push=True,
    )

    @task
    def load_data(**context):
        transformed_data_point = context["ti"].xcom_pull(
            task_ids="transform", key="return_value"
        )
        print(transformed_data_point)

    extract_data() >> transform >> load_data()

KPO_XComs_example_dag()
```

**Traditional**

```python
from airflow import DAG
from pendulum import datetime
from airflow.providers.cncf.kubernetes.operators.kubernetes_pod import (
    KubernetesPodOperator,
)
from airflow.configuration import conf
from airflow.operators.python import PythonOperator

import random

namespace = conf.get("kubernetes", "NAMESPACE")

def extract_data_function():
    data_point = random.randint(0, 100)
    return data_point

def load_data_function(**context):
    transformed_data_point = context["ti"].xcom_pull(
        task_ids="transform", key="return_value"
    )
    print(transformed_data_point)

with DAG(
    dag_id="KPO_XComs_example_dag",
    start_date=datetime(2022, 6, 1),
    catchup=False,
    schedule="@daily",
):
    extract_data = PythonOperator(
        task_id="extract_data", python_callable=extract_data_function
    )

    transform = KubernetesPodOperator(
        task_id="transform",
        image="<image location>",
        in_cluster=True,
        namespace=namespace,
        name="my_pod",
        get_logs=True,
        log_events_on_failure=True,
        env_vars={
            "DATA_POINT": """{{ ti.xcom_pull(task_ids='extract_data',
                                                 key='return_value') }}"""
        },
        do_xcom_push=True,
    )

    load_data = PythonOperator(
        task_id="load_data",
        python_callable=load_data_function,
    )

    extract_data >> transform >> load_data
```

---

[← Изолированные окружения](isolated-environments.md) | [К содержанию](README.md) | [Логирование →](logging.md)
