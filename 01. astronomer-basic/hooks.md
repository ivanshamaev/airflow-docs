# Хуки (Hooks) в Airflow

**Хук** — абстракция над конкретным API, позволяющая Airflow взаимодействовать с внешней системой. Хуки встроены во многие операторы, но их можно использовать и напрямую в коде DAG.

В этом руководстве вы узнаете, как использовать хуки в Airflow и когда имеет смысл вызывать их прямо в DAG. Также вы реализуете два разных хука в одном DAG.

В [Astronomer Registry](https://registry.astronomer.io/modules/?typeName=Hooks) доступно [более 300 хуков](https://registry.astronomer.io/modules/?typeName=Hooks). Если подходящего хука нет, можно написать свой и опубликовать его в сообществе.

> **Инфо.** Подробнее о написании кастомных хуков и операторов см. в руководстве [Custom hooks and operators](https://www.astronomer.io/docs/learn/airflow-importing-custom-hooks-operators).

## Необходимая база

Чтобы получить максимум от руководства, нужно понимать:

- Основы Python. См. [Python Documentation](https://docs.python.org/3/tutorial/index.html).
- Основные концепции Airflow. См. [Введение в Apache Airflow](https://www.astronomer.io/docs/learn/intro-to-airflow).

## Основы хуков

Хуки оборачивают API и предоставляют методы для взаимодействия с внешними системами. Они унифицируют способ работы с внешними системами и делают код DAG чище, понятнее и менее подверженным ошибкам.

Чтобы использовать хук, обычно достаточно **connection ID** для подключения к внешней системе. Подробнее о настройке подключений: [Управление подключениями в Apache Airflow](connections.md).

Все хуки наследуются от класса [BaseHook](https://github.com/apache/airflow/blob/main/airflow-core/src/airflow/hooks/base.py), в котором реализована установка внешнего подключения по connection ID. Помимо установки соединения, конкретные хуки могут содержать дополнительные методы для действий во внешней системе. Эти методы могут опираться на разные Python-библиотеки. Например, [S3Hook](https://registry.astronomer.io/providers/amazon/modules/s3hook) использует библиотеку [boto3](https://boto3.amazonaws.com/v1/documentation/api/latest/index.html) для работы с Amazon S3.

У `S3Hook` есть [более 20 методов](https://github.com/apache/airflow/blob/main/providers/amazon/src/airflow/providers/amazon/aws/hooks/s3.py) для работы с бакетами Amazon S3. Вот некоторые из них:

- **download_file**: загружает файл из S3 в локальную файловую систему.
- **load_file**: загружает локальный файл в S3.
- **list_keys**: перечисляет ключи в бакете по заданным параметрам.
- **list_prefixes**: перечисляет префиксы в бакете по заданным параметрам.
- **check_for_bucket**: проверяет существование бакета с указанным именем.

В следующем примере показано использование хука в DAG.

**TaskFlow API:**

```python
from airflow.sdk import dag, task

@dag
def my_dag():
    @task
    def my_task():
        from airflow.providers.amazon.aws.hooks.s3 import S3Hook

        s3_hook = S3Hook(aws_conn_id="my_aws_conn")
        # используйте методы хука здесь

    my_task()

my_dag()
```

**Традиционный синтаксис:**

```python
from airflow.sdk import DAG, task
from airflow.providers.standard.operators.python import PythonOperator

def _my_task():
    from airflow.providers.amazon.aws.hooks.s3 import S3Hook

    s3_hook = S3Hook(aws_conn_id="my_aws_conn")
    # используйте методы хука здесь

with DAG(dag_id="my_dag"):

    my_task = PythonOperator(task_id="my_task", python_callable=_my_task)
```

## Когда использовать хуки

Так как хуки — строительные блоки операторов, их использование в Airflow часто скрыто от автора DAG. Но в ряде случаев хуки стоит вызывать напрямую в Python-функции в DAG. Общие рекомендации:

- Если нужно регулярно подключаться к API и подходящего хука нет — напишите свой хук и опубликуйте его в сообществе.
- Если для вашей задачи уже есть оператор со встроенным хуком — используйте оператор, а не ручную настройку хука.
- Если вы пишете кастомный оператор для работы с внешней системой, он должен использовать хук.
- Всегда предпочтительнее использовать хук, а не ручное взаимодействие с API. Хуки часто используют в [декорированных функциях Airflow](../02.%20astronomer-dags/airflow-decorators.md) (например, с `@task`) и в DAG, заданных декоратором [@asset](assets.md).

## Пример: S3 и Slack

В примере ниже используются хуки [S3Hook](https://registry.astronomer.io/providers/amazon/modules/s3hook) и [SlackHook](https://registry.astronomer.io/providers/slack/modules/slackhook): чтение значений из файлов в бакете Amazon S3, проверка по ним, отправка результата в Slack и логирование ответа Slack API.

Хуки здесь вызываются напрямую в Python-функциях, потому что ни один из существующих операторов Amazon S3 не умеет читать данные из нескольких файлов в бакете, а операторы Slack не возвращают ответ вызова Slack API (который может понадобиться для логирования и мониторинга).

Исходный код хуков:

- [SlackHook](https://github.com/apache/airflow/blob/main/providers/slack/src/airflow/providers/slack/hooks/slack.py)
- [S3Hook](https://github.com/apache/airflow/blob/main/providers/amazon/src/airflow/providers/amazon/aws/hooks/s3.py)

### Подготовка

Перед запуском примера установите нужные провайдеры Airflow. При использовании Astro CLI добавьте в `requirements.txt`:

```text
apache-airflow-providers-amazon
apache-airflow-providers-slack
```

### Создание подключений

1. В Airflow UI откройте Admin → Connections и нажмите + Add Connection.
2. В поле Connection ID введите уникальное имя подключения.
3. В списке Connection Type выберите **aws** для бакета Amazon S3. Если тип aws недоступен, проверьте установку провайдера.
4. В поле Login введите AWS Access Key ID.
5. В поле Password введите AWS Secret Access Key. Получить ключи: [AWS Account and Access Keys](https://docs.aws.amazon.com/powershell/latest/userguide/pstools-appendix-sign-up.html).
6. Нажмите Save.
7. Повторите шаги 1–6 для нового подключения к Slack: выберите тип **slack**, в поле Password укажите [Bot User OAuth Token](https://api.slack.com/authentication/oauth-v2). Токен: api.slack.com/apps → Features → OAuth & Permissions. Нажмите Save.

### Запуск примера DAG

В примере DAG используются [декораторы Airflow](../02.%20astronomer-dags/airflow-decorators.md) для задач и [XCom](../02.%20astronomer-dags/passing-data-between-tasks.md) для передачи данных между задачами. Имя бакета S3 и имена файлов, которые читает первая задача, заданы переменными окружения.

DAG выполняет следующие шаги:

- Задача с **S3Hook** читает три заданных ключа из S3 методом `read_key` и возвращает словарь с содержимым файлов, преобразованным в целые числа.
- Вторая задача выполняет простую проверку суммы по результатам первой задачи.
- Метод **SlackHook** `call` отправляет результат проверки в канал Slack и возвращает ответ Slack API.

**TaskFlow API:**

```python
from datetime import datetime
from airflow.decorators import dag, task
from airflow.providers.slack.hooks.slack import SlackHook
from airflow.providers.amazon.aws.hooks.s3 import S3Hook

S3BUCKET_NAME = "myhooktutorial"
S3_EXAMPLE_FILE_NAME_1 = "file1.txt"
S3_EXAMPLE_FILE_NAME_2 = "file2.txt"
S3_EXAMPLE_FILE_NAME_3 = "file3.txt"


@task
def read_keys_from_s3():
    s3_hook = S3Hook(aws_conn_id="aws_conn")
    response_file_1 = s3_hook.read_key(
        key=S3_EXAMPLE_FILE_NAME_1, bucket_name=S3BUCKET_NAME
    )
    response_file_2 = s3_hook.read_key(
        key=S3_EXAMPLE_FILE_NAME_2, bucket_name=S3BUCKET_NAME
    )
    response_file_3 = s3_hook.read_key(
        key=S3_EXAMPLE_FILE_NAME_3, bucket_name=S3BUCKET_NAME
    )

    response = {
        "num1": int(response_file_1),
        "num2": int(response_file_2),
        "num3": int(response_file_3),
    }

    return response


@task
def run_sum_check(response):
    if response["num1"] + response["num2"] == response["num3"]:
        return (True, response["num3"])
    return (False, response["num3"])


@task
def post_to_slack(sum_check_result):
    slack_hook = SlackHook(slack_conn_id="hook_tutorial_slack_conn")

    if sum_check_result[0] is True:
        server_response = slack_hook.call(
            api_method="chat.postMessage",
            json={
                "channel": "#test-airflow",
                "text": f"""All is well in your bucket!
                        Correct sum: {sum_check_result[1]}!""",
            },
        )
    else:
        server_response = slack_hook.call(
            api_method="chat.postMessage",
            json={
                "channel": "#test-airflow",
                "text": f"""A test on your bucket contents failed!
                        Target sum not reached: {sum_check_result[1]}""",
            },
        )

    return server_response


@dag(
    dag_id="hook_tutorial",
    start_date=datetime(2022, 5, 20),
    schedule="@daily",
    catchup=False,
)
def hook_tutorial():
    response = read_keys_from_s3()
    sum_check_result = run_sum_check(response)
    post_to_slack(sum_check_result)


hook_tutorial()
```

**Традиционный синтаксис:**

```python
from datetime import datetime
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.providers.slack.hooks.slack import SlackHook
from airflow.providers.amazon.aws.hooks.s3 import S3Hook

S3BUCKET_NAME = "myhooktutorial"
S3_EXAMPLE_FILE_NAME_1 = "file1.txt"
S3_EXAMPLE_FILE_NAME_2 = "file2.txt"
S3_EXAMPLE_FILE_NAME_3 = "file3.txt"


def read_keys_from_s3_function():
    s3_hook = S3Hook(aws_conn_id="aws_conn")
    response_file_1 = s3_hook.read_key(
        key=S3_EXAMPLE_FILE_NAME_1, bucket_name=S3BUCKET_NAME
    )
    response_file_2 = s3_hook.read_key(
        key=S3_EXAMPLE_FILE_NAME_2, bucket_name=S3BUCKET_NAME
    )
    response_file_3 = s3_hook.read_key(
        key=S3_EXAMPLE_FILE_NAME_3, bucket_name=S3BUCKET_NAME
    )

    response = {
        "num1": int(response_file_1),
        "num2": int(response_file_2),
        "num3": int(response_file_3),
    }

    return response


def run_sum_check_function(response):
    if response["num1"] + response["num2"] == response["num3"]:
        return (True, response["num3"])
    return (False, response["num3"])


def post_to_slack_function(sum_check_result):
    slack_hook = SlackHook(slack_conn_id="hook_tutorial_slack_conn")

    if sum_check_result[0] is True:
        server_response = slack_hook.call(
            api_method="chat.postMessage",
            json={
                "channel": "#test-airflow",
                "text": f"""All is well in your bucket!
                        Correct sum: {sum_check_result[1]}!""",
            },
        )
    else:
        server_response = slack_hook.call(
            api_method="chat.postMessage",
            json={
                "channel": "#test-airflow",
                "text": f"""A test on your bucket contents failed!
                        Target sum not reached: {sum_check_result[1]}""",
            },
        )

    return server_response


with DAG(
    dag_id="hook_tutorial",
    start_date=datetime(2022, 5, 20),
    schedule="@daily",
    catchup=False,
    render_template_as_native_obj=True,
):
    read_keys_form_s3 = PythonOperator(
        task_id="read_keys_form_s3", python_callable=read_keys_from_s3_function
    )

    run_sum_check = PythonOperator(
        task_id="run_sum_check",
        python_callable=run_sum_check_function,
        op_kwargs={
            "response": "{{ ti.xcom_pull(task_ids='read_keys_form_s3', \
                key='return_value') }}"
        },
    )

    post_to_slack = PythonOperator(
        task_id="post_to_slack",
        python_callable=post_to_slack_function,
        op_kwargs={
            "sum_check_result": "{{ ti.xcom_pull(task_ids='run_sum_check', \
                key='return_value') }}"
        },
    )

    read_keys_form_s3 >> run_sum_check >> post_to_slack
```

---

[← Выполнение SQL](execute-sql.md) | [К содержанию](README.md) | [Управление кодом →](managing-airflow-code.md)
