# Переменные (Variables) в Airflow

**Переменная Airflow** — это пара ключ–значение для хранения информации в окружении Airflow. Обычно переменные используют для данных уровня инстанса, которые редко меняются: секреты (API-ключи), пути к конфигурационным файлам и т.п.

В Airflow есть два типа переменных: обычные значения и значения в виде сериализованного JSON.

В этом руководстве описано, как создавать переменные Airflow и обращаться к ним из кода.

## Необходимая база

Полезно понимать:

- Операторы Airflow. См. [Что такое оператор?](operators.md).
- DAG в Airflow. См. [Введение в DAG](dags.md).

## Рекомендации по хранению информации в Airflow

[Переменные Airflow](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/variables.html#variables) хранят пары ключ–значение или короткие JSON-объекты, к которым нужен доступ во всём инстансе Airflow. Это механизм конфигурации в рантайме; переменные задаются через объект [`airflow.sdk.definitions.variable.Variable`](https://github.com/apache/airflow/blob/main/task-sdk/src/airflow/sdk/definitions/variable.py).

Рекомендации при использовании переменных:

- Переменные шифруются с помощью [Fernet](https://github.com/fernet/spec/) при записи в metastore Airflow. Чтобы маскировать переменные в UI и логах, включите в имя переменной подстроку, указывающую на чувствительное значение. См. раздел «Скрытие чувствительной информации в переменных Airflow» ниже.
- Если переменные используются в коде DAG на верхнем уровне, обращайтесь к ним через [Jinja-шаблоны](../02.%20astronomer-dags/jinja-templating.md), чтобы значение подставлялось только при выполнении задачи.
- По возможности не обращайтесь к переменным Airflow вне задач, в коде DAG на верхнем уровне: при каждом парсинге DAG это создаёт подключение к metastore и может ухудшать производительность. См. [Рекомендации по написанию DAG](../02.%20astronomer-dags/dag-best-practices.md).
- Переменные подходят для данных, зависящих от окружения в рантайме, но не меняющихся слишком часто.

Примеры правильного и неправильного обращения к переменным в DAG: [документация Airflow](https://airflow.apache.org/docs/apache-airflow/stable/best-practices.html#airflow-variables).

Кроме переменных Airflow есть и другие способы хранения данных. Выбор зависит от типа данных и от того, где и как к ним нужен доступ:

- **[XCom](../02.%20astronomer-dags/passing-data-between-tasks.md)** — передача небольших данных между задачами. Удобен, когда данные могут меняться от запуска к запуску DAG и нужны в основном отдельным задачам. XCom по умолчанию не шифруется; не используйте его для секретов.
- **[Params](../02.%20astronomer-dags/airflow-params.md)** — данные, специфичные для DAG или для DAG run. Значения по умолчанию задаются на уровне DAG или задачи и могут переопределяться в рантайме. Params не шифруются; не используйте их для секретов.
- **Переменные окружения** — небольшие данные, доступные во всём окружении Airflow. В UI Airflow их не видно; в DAG и задачах доступ через `os.getenv("MY_ENV_VAR")`. Удобны и для произвольных данных, и для [настройки Airflow](https://airflow.apache.org/docs/apache-airflow/stable/howto/set-config.html). Плюс — можно задавать их в CI/CD. Часто используются для учётных данных при локальной разработке.

## Создание переменной Airflow

Переменные можно создавать так:

- программно из задачи Airflow;
- через переменную окружения;
- через Airflow CLI;
- через Airflow UI.

### Через Airflow UI

В UI откройте вкладку **Admin** → **Variables**. Нажмите **+**, введите ключ, значение и при необходимости описание. Доступен также импорт переменных из файла.

### Через Airflow CLI

В CLI есть команды для установки, чтения и удаления [переменных](https://airflow.apache.org/docs/apache-airflow/stable/cli-and-env-variables-ref.html#variables). Создание переменной:

**Astro:**

```sh
astro dev run variables set my_var my_value
astro dev run variables set -j my_json_var '{"key": "value"}'
```

Команда [`astro dev run`](https://www.astronomer.io/docs/astro/cli/astro-dev-run) выполняется только в локальном окружении Airflow и не применяется к Astro Deployments. Для переменных в Astro Deployment используйте [Astro Environment Manager](https://www.astronomer.io/docs/astro/manage-connections-variables) или [переменные окружения](https://www.astronomer.io/docs/learn/airflow-variables#using-environment-variables).

**Airflow:**

```sh
airflow variables set my_var my_value
airflow variables set -j my_json_var '{"key": "value"}'
```

### Через переменные окружения

Чтобы задать переменную Airflow через окружение, создайте переменную окружения с префиксом `AIRFLOW_VAR_` и именем переменной Airflow. Пример:

```text
AIRFLOW_VAR_MYREGULARVAR='my_value'
AIRFLOW_VAR_MYJSONVAR='{"hello":"world"}'
```

Получить значение в DAG можно двумя способами:

- **`os.getenv('AIRFLOW_VAR_<ИМЯ>', '')`** — быстрее, меньше обращений к БД метаданных. Менее безопасно: Astronomer не рекомендует использовать `os.getenv` для секретов, так как при таком способе значения могут попасть в логи.
- **`Variable.get('<имя>', '')`** — предпочтительный способ для секретов. Но вызов на верхнем уровне DAG или в аргументах оператора при каждом парсинге DAG (например, каждые 30 секунд) обращается к БД и может влиять на производительность. Альтернатива — Jinja-шаблон `{{ var.value.get('<имя>', '') }}`, который вычисляется только при выполнении задачи. См. [Рекомендации по написанию DAG](../02.%20astronomer-dags/dag-best-practices.md).

Если переменная окружения не задана, вторым аргументом укажите значение по умолчанию.

Настройка переменных окружения в Astro: [Environment Variables](https://www.astronomer.io/docs/astro/manage-env-vars).

### Программно из DAG или задачи

Переменные можно задавать из кода задач через [модель `Variable`](https://airflow.apache.org/docs/apache-airflow/stable/_api/airflow/models/variable/index.html#module-airflow.models.variable). Для JSON нужно передать `serialize_json=True`.

**TaskFlow:**

```python
from airflow.sdk import task


@task
def set_var():
    from airflow.sdk import Variable
    Variable.set(key="my_regular_var", value="Hello!")
    Variable.set(
        key="my_json_var",
        value={"num1": 23, "num2": 42},
        serialize_json=True,
    )
```

**Традиционный вариант:**

```python
from airflow.providers.standard.operators.python import PythonOperator


def set_var_func():
    from airflow.sdk import Variable
    Variable.set(key="my_regular_var", value="Hello!")
    Variable.set(
        key="my_json_var",
        value={"num1": 23, "num2": 42},
        serialize_json=True,
    )


PythonOperator(
    task_id="set_var",
    python_callable=set_var_func,
)
```

Обновление переменной делается так же через метод `.update()`.

## Получение переменной Airflow

Получить переменную из кода можно методом `.get()` модели Variable или из [контекста Airflow](../02.%20astronomer-dags/airflow-context.md).

Для переменной, сохранённой как JSON, в `.get()` укажите `deserialize_json=True` либо обращайтесь к ключу `json` в словаре `var` в контексте.

**TaskFlow:**

```python
from airflow.sdk import task


@task
def get_var_regular():
    from airflow.sdk import Variable
    my_regular_var = Variable.get("my_regular_var", default=None)
    my_json_var = Variable.get(
        "my_json_var", deserialize_json=True, default=None
    )["num1"]
    print(my_regular_var)
    print(my_json_var)


@task
def get_var_from_context(**context):
    my_regular_var = context["var"]["value"].get("my_regular_var")
    my_json_var = context["var"]["json"].get("my_json_var")["num2"]
    print(my_regular_var)
    print(my_json_var)
```

**Традиционный вариант:**

```python
from airflow.providers.standard.operators.python import PythonOperator


def get_var_regular_func():
    from airflow.sdk import Variable
    my_regular_var = Variable.get("my_regular_var", default=None)
    my_json_var = Variable.get(
        "my_json_var", deserialize_json=True, default=None
    )["num1"]
    print(my_regular_var)
    print(my_json_var)


def get_var_from_context_func(**context):
    my_regular_var = context["var"]["value"].get("my_regular_var")
    my_json_var = context["var"]["json"].get("my_json_var")["num2"]
    print(my_regular_var)
    print(my_json_var)


PythonOperator(
    task_id="get_var_regular",
    python_callable=get_var_regular_func,
)

PythonOperator(
    task_id="get_var_from_context",
    python_callable=get_var_from_context_func,
)
```

С традиционными операторами часто удобнее получать переменные через [Jinja-шаблон](../02.%20astronomer-dags/jinja-templating.md). См. [Переменные в шаблонах Airflow](https://airflow.apache.org/docs/apache-airflow/stable/templates-ref.html#airflow-variables-in-templates).

```python
from airflow.providers.standard.operators.bash import BashOperator

get_var_jinja = BashOperator(
    task_id="get_var_jinja",
    bash_command='echo "{{ var.value.my_regular_var }} {{ var.json.my_json_var.num2 }}"',
)
```

Читать переменные можно и через CLI: команды [`get`](https://airflow.apache.org/docs/apache-airflow/stable/cli-and-env-variables-ref.html#get_repeat3) и [`list`](https://airflow.apache.org/docs/apache-airflow/stable/cli-and-env-variables-ref.html#list_repeat8).

## Скрытие чувствительной информации в переменных Airflow

Переменные в metastore Airflow хранятся в зашифрованном виде ([Fernet](https://github.com/fernet/spec/)).

Часть переменных дополнительно маскируется в UI и логах. По умолчанию опция [`hide_sensitive_var_conn_fields`](https://airflow.apache.org/docs/apache-airflow/stable/configurations-ref.html#hide-sensitive-var-conn-fields) включена (`True`), и автоматически маскируются все переменные, в имени которых встречаются подстроки:

- `token`
- `secret`
- `private_key`
- `password`
- `passwd`
- `passphrase`
- `authorization`
- `apikey`
- `api_key`
- `access_token`

Список можно расширить, добавив через запятую строки в конфигурацию [`sensitive_var_conn_names`](https://airflow.apache.org/docs/apache-airflow/stable/configurations-ref.html#sensitive-var-conn-names). См. [Маскирование чувствительных данных](https://airflow.apache.org/docs/apache-airflow/stable/administration-and-deployment/security/secrets/mask-sensitive-values.html).

В Astro переменные можно вручную помечать как секреты при создании через переменные окружения. См. [Set environment variables on Astro](https://www.astronomer.io/docs/astro/environment-variables).

> **Инфо.** Если один и тот же секрет нужен в нескольких инстансах Airflow, рассмотрите использование [Secrets Backend](https://airflow.apache.org/docs/apache-airflow/stable/security/secrets/secrets-backend/index.html).
---

[← Trigger rules](trigger-rules.md) | [К содержанию](README.md) | [Connections →](connections.md)
