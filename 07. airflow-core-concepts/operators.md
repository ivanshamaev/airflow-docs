# Operators (операторы)

**Operator** — по сути шаблон предопределённой [задачи](tasks.md), которую можно описать декларативно внутри DAG:

```python
with DAG("my-dag") as dag:
    ping = HttpOperator(endpoint="http://example.com/update/")
    email = EmailOperator(to="admin@example.com", subject="Update complete")

    ping >> email
```

В Airflow доступен большой набор операторов: часть входит в ядро или предустановленные провайдеры. Из ядра часто используют:

- **[BashOperator](https://airflow.apache.org/docs/apache-airflow-providers-standard/stable/_api/airflow/providers/standard/operators/bash/index.html#airflow.providers.standard.operators.bash.BashOperator)** — выполняет bash-команду
- **[PythonOperator](https://airflow.apache.org/docs/apache-airflow-providers-standard/stable/_api/airflow/providers/standard/operators/python/index.html#airflow.providers.standard.operators.python.PythonOperator)** — вызывает произвольную Python-функцию

Для выполнения произвольной Python-функции рекомендуется декоратор `@task`. Он не поддерживает рендеринг Jinja-шаблонов в аргументах.

> **Примечание.** Для вызова Python-callable без шаблонизации аргументов рекомендуется декоратор `@task`, а не классический [PythonOperator](https://airflow.apache.org/docs/apache-airflow-providers-standard/stable/_api/airflow/providers/standard/operators/python/index.html#airflow.providers.standard.operators.python.PythonOperator).

Полный список операторов ядра: [Core Operators and Hooks Reference](https://airflow.apache.org/docs/apache-airflow/stable/operators-and-hooks-ref.html).

Если нужного оператора нет в поставке Airflow по умолчанию, его можно найти среди [провайдеров](https://airflow.apache.org/docs/apache-airflow-providers-index.html). Примеры популярных операторов из провайдеров:

- [EmailOperator](https://airflow.apache.org/docs/apache-airflow-providers-smtp/stable/_api/airflow/providers/smtp/operators/smtp/index.html#airflow.providers.smtp.operators.smtp.EmailOperator)
- [HttpOperator](https://airflow.apache.org/docs/apache-airflow-providers-http/stable/_api/airflow/providers/http/operators/http/index.html#airflow.providers.http.operators.http.HttpOperator)
- [SQLExecuteQueryOperator](https://airflow.apache.org/docs/apache-airflow-providers-common-sql/stable/_api/airflow/providers/common/sql/operators/sql/index.html#airflow.providers.common.sql.operators.sql.SQLExecuteQueryOperator)
- [DockerOperator](https://airflow.apache.org/docs/apache-airflow-providers-docker/stable/_api/airflow/providers/docker/operators/docker/index.html#airflow.providers.docker.operators.docker.DockerOperator)
- [HiveOperator](https://airflow.apache.org/docs/apache-airflow-providers-apache-hive/stable/_api/airflow/providers/apache/hive/operators/hive/index.html#airflow.providers.apache.hive.operators.hive.HiveOperator)
- [S3FileTransformOperator](https://airflow.apache.org/docs/apache-airflow-providers-amazon/stable/_api/airflow/providers/amazon/aws/operators/s3/index.html#airflow.providers.amazon.aws.operators.s3.S3FileTransformOperator)
- [PrestoToMySqlOperator](https://airflow.apache.org/docs/apache-airflow-providers-mysql/stable/_api/airflow/providers/mysql/transfers/presto_to_mysql/index.html#airflow.providers.mysql.transfers.presto_to_mysql.PrestoToMySqlOperator)
- [SlackAPIOperator](https://airflow.apache.org/docs/apache-airflow-providers-slack/stable/_api/airflow/providers/slack/operators/slack/index.html#airflow.providers.slack.operators.slack.SlackAPIOperator)

И многие другие — полный список операторов, хуков, сенсоров и трансферов из провайдеров: [providers packages](https://airflow.apache.org/docs/apache-airflow-providers/operators-and-hooks-ref/index.html).

> **Примечание.** В коде Airflow понятия [Tasks](tasks.md) и Operators часто смешивают, и в большинстве случаев они взаимозаменяемы. **Task** — общая «единица выполнения» в DAG; **Operator** — готовый переиспользуемый шаблон задачи с уже реализованной логикой, которому нужно лишь передать аргументы.

## Jinja Templating (шаблонизация Jinja)

Airflow использует [Jinja](http://jinja.pocoo.org/docs/dev/) в сочетании с [макросами](https://airflow.apache.org/docs/apache-airflow/stable/templates-ref.html#templates-ref).

Например, чтобы передать начало data interval в bash-скрипт как переменную окружения через `BashOperator`:

```python
# Начало data interval в формате YYYY-MM-DD
date = "{{ ds }}"
t = BashOperator(
    task_id="test_env",
    bash_command="/tmp/test.sh ",
    dag=dag,
    env={"DATA_INTERVAL_START": date},
)
```

Здесь `{{ ds }}` — шаблонная переменная; параметр `env` у `BashOperator` обрабатывается Jinja, поэтому дата начала data interval будет доступна в bash-скрипте как переменная окружения `DATA_INTERVAL_START`.

Вместо строки с шаблоном можно передать callable — когда на Python это читается проще. Callable должен принимать два именованных аргумента: `context` и `jinja_env`:

- **`context`** — объект `Context` Airflow с информацией о текущем выполнении задачи. Доступ — как к обычному [словарю](https://docs.python.org/3/library/stdtypes.html#mapping-types-dict). Содержит все переменные, доступные в Jinja-шаблонах; с точки зрения рендеринга шаблонов он только для чтения — изменения не влияют на среду выполнения задачи.

Полный список переменных контекста: [Templates reference](https://airflow.apache.org/docs/apache-airflow/stable/templates-ref.html#templates-variables).

```python
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    import jinja2
    from airflow.sdk import Context


def build_complex_command(context: Context, jinja_env: jinja2.Environment) -> str:
    # Доступ к данным выполнения из контекста
    task_id = context["ti"].task_id
    execution_date = context["ds"]
    with open("file.csv") as f:
        return do_complex_things(f, task_id, execution_date)


t = BashOperator(
    task_id="complex_templated_echo",
    bash_command=build_complex_command,
    dag=dag,
)
```

Каждое template-поле рендерится один раз, поэтому возвращаемое callable значение повторно не обрабатывается. При необходимости шаблоны внутри него нужно рендерить вручную, например через `render_template()` у текущей задачи:

```python
def build_complex_command(context: Context, jinja_env: jinja2.Environment) -> str:
    with open("file.csv") as f:
        data = do_complex_things(f)
    return context["task"].render_template(data, context, jinja_env)
```

Шаблонизация применима ко всем параметрам, помеченным в документации как «templated». Подстановка выполняется непосредственно перед вызовом функции `pre_execute` оператора.

Шаблонизацию можно использовать и для вложенных полей, если эти поля помечены как templated в своей структуре: поля из свойства `template_fields` проходят подстановку. Пример — поле `path`:

```python
class MyDataReader:
    template_fields: Sequence[str] = ("path",)

    def __init__(self, my_path):
        self.path = my_path

    # [дополнительный код...]


t = PythonOperator(
    task_id="transform_data",
    python_callable=transform_data,
    op_args=[MyDataReader("/tmp/{{ ds }}/my_file")],
    dag=dag,
)
```

> **Примечание.** Свойство `template_fields` — переменная класса и должно иметь тип `Sequence[str]` (список или кортеж строк).

Глубоко вложенные поля тоже подставляются, если все промежуточные поля помечены как template fields:

```python
class MyDataTransformer:
    template_fields: Sequence[str] = ("reader",)

    def __init__(self, my_reader):
        self.reader = my_reader

    # [дополнительный код...]


class MyDataReader:
    template_fields: Sequence[str] = ("path",)

    def __init__(self, my_path):
        self.path = my_path

    # [дополнительный код...]


t = PythonOperator(
    task_id="transform_data",
    python_callable=transform_data,
    op_args=[MyDataTransformer(MyDataReader("/tmp/{{ ds }}/my_file"))],
    dag=dag,
)
```

При создании DAG в Jinja `Environment` можно передать свои опции. Часто включают сохранение завершающего перевода строки в шаблоне:

```python
my_dag = DAG(
    dag_id="my-dag",
    jinja_environment_kwargs={
        "keep_trailing_newline": True,
        # другие опции jinja2 Environment
    },
)
```

Все доступные опции: [Jinja documentation](https://jinja.palletsprojects.com/en/2.11.x/api/#jinja2.Environment).

У части операторов строки с определёнными суффиксами (задаются в `template_ext`) при рендеринге считаются путями к файлам. Так можно подгружать скрипты или запросы из файлов, а не вставлять их в код DAG.

Пример: `BashOperator` с многострочным bash-скриптом — загружается файл `script.sh`, его содержимое подставляется в `bash_command`:

```python
run_script = BashOperator(
    task_id="run_script",
    bash_command="script.sh",
)
```

По умолчанию пути задаются относительно папки DAG (это путь поиска Jinja-шаблонов по умолчанию). Дополнительные пути можно добавить через аргумент `template_searchpath` у DAG.

Иногда нужно исключить строку из шаблонизации и использовать её как есть. Пример:

```python
print_script = BashOperator(
    task_id="print_script",
    bash_command="cat script.sh",
)
```

Так будет ошибка `TemplateNotFound: cat script.sh` — Airflow воспримет строку как путь к файлу, а не команду. Чтобы значение не считалось путём к файлу, его оборачивают в `literal()`. При этом отключаются и подстановка макросов, и загрузка из файла; можно применять к отдельным вложенным полям, оставляя обычную шаблонизацию для остального содержимого.

```python
from airflow.sdk import literal


fixed_print_script = BashOperator(
    task_id="fixed_print_script",
    bash_command=literal("cat script.sh"),
)
```

*Добавлено в версии 2.8:* функция `literal()`.

Либо можно переопределить `template_ext`, чтобы значение не трактовалось как путь к файлу:

```python
fixed_print_script = BashOperator(
    task_id="fixed_print_script",
    bash_command="cat script.sh",
)
fixed_print_script.template_ext = ()
```

### Рендеринг полей как нативных Python-объектов

По умолчанию все Jinja-шаблоны в `template_fields` рендерятся в строки. Иногда нужен другой тип. Например, задача `extract` пушит в [XCom](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/xcoms.html#concepts-xcom) словарь `{"1001": 301.27, "1002": 433.21, "1003": 502.22}`:

```python
@task(task_id="extract")
def extract():
    data_string = '{"1001": 301.27, "1002": 433.21, "1003": 502.22}'
    return json.loads(data_string)
```

Если задача зависит от `extract` и в `order_data` передаётся строка `"{'1001': 301.27, '1002': 433.21, '1003': 502.22}"`:

```python
def transform(order_data):
    total_order_value = sum(order_data.values())  # Ошибка: order_data — str
    return {"total_order_value": total_order_value}


transform = PythonOperator(
    task_id="transform",
    op_kwargs={"order_data": "{{ ti.xcom_pull('extract') }}"},
    python_callable=transform,
)

extract() >> transform
```

Чтобы получить именно словарь, есть два варианта. Первый — использовать callable:

```python
def render_transform_op_kwargs(context, jinja_env):
    order_data = context["ti"].xcom_pull("extract")
    return {"order_data": order_data}


transform = PythonOperator(
    task_id="transform",
    op_kwargs=render_transform_op_kwargs,
    python_callable=transform,
)
```

Второй — указать Jinja рендерить нативный Python-объект. Для этого в DAG передают `render_template_as_native_obj=True`. Тогда вместо стандартного `SandboxedEnvironment` используется [NativeEnvironment](https://jinja.palletsprojects.com/en/2.11.x/nativetypes/):

```python
with DAG(
    dag_id="example_template_as_python_object",
    schedule=None,
    start_date=pendulum.datetime(2021, 1, 1, tz="UTC"),
    catchup=False,
    render_template_as_native_obj=True,
):
    transform = PythonOperator(
        task_id="transform",
        op_kwargs={"order_data": "{{ ti.xcom_pull('extract') }}"},
        python_callable=transform,
    )
```

## Зарезервированное ключевое слово params

В Apache Airflow 2.2.0 переменная `params` используется при сериализации DAG. В сторонних операторах это имя использовать не следует. Если после обновления окружения появляется ошибка:

```
AttributeError: 'str' object has no attribute '__module__'
```

переименуйте `params` в своих операторах.

### Конфликт шаблонизации с f-строками

При формировании строк для template-полей (например, `bash_command` в `BashOperator`) через Python f-строки нужно учитывать взаимодействие подстановки f-строк и синтаксиса Jinja: в обоих случаях используются фигурные скобки `{}`.

В f-строках двойные фигурные скобки `{{` и `}}` интерпретируются как экранирование одной скобки `{` или `}`. В Jinja двойные скобки `{{ variable }}` обозначают переменную для подстановки.

Чтобы в строке, заданной f-строкой, оставить выражение Jinja (например, `{{ ds }}`) для последующей обработки движком Airflow, скобки для f-строки нужно экранировать ещё одним удвоением — то есть использовать четыре скобки:

```python
t1 = BashOperator(
    task_id="fstring_templating_correct",
    bash_command=f"echo Data interval start: {{{{ ds }}}}",
    dag=dag,
)

python_var = "echo Data interval start:"

t2 = BashOperator(
    task_id="fstring_templating_simple",
    bash_command=f"{python_var} {{{{ ds }}}}",
    dag=dag,
)
```

В результате после обработки f-строки получится строка с двойными скобками для Jinja, и Airflow корректно выполнит подстановку перед запуском. Частая ошибка у новичков — не удвоить скобки; из-за этого возможны ошибки при разборе DAG или неожиданное поведение при выполнении, когда подстановка не срабатывает.

---

*Источник: [Airflow 3.1.7 — Operators](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/operators.html). Перевод неофициальный.*
