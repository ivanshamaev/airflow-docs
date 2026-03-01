# Уведомления об ошибках в Airflow (Error notifications)

Как понять, что в оркестрации данных что-то пошло не так? Пользователи [Apache Airflow®](https://airflow.apache.org/) могут смотреть статус DAG в UI, но для систематического управления ошибками это неудобно, особенно если сбои нужно обрабатывать быстро или нескольким людям. В Airflow есть несколько механизмов уведомлений, с помощью которых можно настроить оповещения под вашу организацию.

В этом руководстве описана настройка основных способов уведомлений: email (SMTP), callback’и Airflow и нотификаторы (notifiers).

## Необходимая база

Полезно понимать:

- Подключения в Airflow. См. [Управление подключениями](../01.%20astronomer-basic/connections.md).
- Декораторы Airflow. См. [TaskFlow API и декораторы](airflow-decorators.md).
- Операторы Airflow. См. [Что такое оператор?](../01.%20astronomer-basic/operators.md).
- DAG в Airflow. См. [Введение в DAG](../01.%20astronomer-basic/dags.md).

## Типы уведомлений

При настройке уведомлений в Airflow нужно выбрать: встроенная система Airflow, внешний мониторинг или их комбинация. При работе Airflow на Astro доступны три типа:

**Уведомления Airflow** — встроены в open-source Airflow и задаются параметрами callback и/или конфигурационными переменными для email и SMTP.

**Astro Alerts** — функция Astro для настройки алертов сразу для многих DAG и Deployments.

**Astro Observe** — продукт Astronomer с возможностью задавать data products для нескольких DAG и Deployments и определять для них SLA (Service Level Agreements).

Плюс уведомлений Airflow — их можно задавать прямо в коде DAG. Минус — для отправки нужен работающий Airflow, поэтому при проблемах с инфраструктурой уведомления могут не уйти. У уведомлений Airflow есть и другие ограничения (например, по SLA и таймаутам).

Когда возможностей Airflow недостаточно, [Astro Alerts](https://www.astronomer.io/docs/astro/alerts) и [Astro Observe](https://www.astronomer.io/docs/astro/astro-observe) дают дополнительный уровень наблюдаемости. Когда выбирать уведомления Airflow, а когда Astro Alerts: [When to use Airflow or Astro alerts](https://www.astronomer.io/docs/astro/best-practices/airflow-vs-astro-alerts).

## Концепции уведомлений в Airflow

При задании уведомлений в Airflow важно понимать следующее:

**Email (SMTP)** — Airflow может отправлять письма через внешний SMTP-сервер. Email-уведомления настраиваются разными способами.

**Airflow Callbacks** — параметры уровня DAG и задачи, в которых задаётся код, выполняемый при достижении DAG или задачей определённого состояния. Можно использовать обычные Python-функции или нотификаторы.

**Airflow Notifiers** — классы в Airflow (как операторы или хуки), позволяющие унифицировать код уведомлений. У каждого нотификатора есть метод `.notify()` для отправки.

**Airflow Listeners** — продвинутый механизм Airflow: код выполняется в фоне при определённых событиях во всём окружении Airflow.

### Выбор способа уведомлений в Airflow

По возможности лучше использовать готовые решения: меньше своего кода и единообразие уведомлений в разных окружениях.

- **Email:** используйте SmtpNotifier или EmailOperator (разделы ниже). Для SendGrid, Amazon SES и т.п. см. [документацию Airflow](https://airflow.apache.org/docs/apache-airflow/stable/howto/email-config.html).
- **Другие каналы:** проверьте, есть ли готовый нотификатор под вашу систему (раздел «Готовые нотификаторы»). Список нотификаторов: [документация Airflow](https://airflow.apache.org/docs/apache-airflow-providers/core-extensions/notifications.html); сервисы для [AppriseNotifier](https://airflow.apache.org/docs/apache-airflow-providers-apprise/stable/_api/airflow/providers/apprise/notifications/apprise/index.html): [Apprise wiki](https://github.com/caronc/apprise/wiki).
- **Callback-функции** — используйте только если подходящего нотификатора нет. Имеет смысл оформить отправку в кастомный нотификатор (раздел ниже).
- **События по всему окружению** (обновление ассета, падение DAG run, ошибка импорта и т.д.) — [Airflow Listeners](https://airflow.apache.org/docs/apache-airflow/stable/administration-and-deployment/listeners.html#listeners).

## Email (SMTP) уведомления

Настроить email-уведомления в Airflow можно тремя способами:

- **(Устаревший)** Параметр задачи `email` в сочетании с конфигурационными переменными в секции SMTP. Подход ограничен и будет удалён в будущей версии. См. раздел «(Legacy) Email через конфигурационные переменные» ниже.
- **(Рекомендуется)** EmailOperator — отдельные задачи в DAG для отправки писем.
- **(Рекомендуется)** SmtpNotifier в любом параметре callback — отправка email при достижении DAG или задачей нужного состояния.

Для любых email-уведомлений нужен [SMTP provider](https://registry.astronomer.io/providers/apache-airflow-providers-smtp/versions/latest): добавьте в `requirements.txt`:

```text
apache-airflow-providers-smtp
```

### Использование SmtpNotifier

**SmtpNotifier** — готовый нотификатор, который можно передать в любой [параметр callback](#airflow-callbacks), чтобы отправлять письма при срабатывании callback’а.

Чтобы подключить нотификатор к SMTP-серверу, создайте [подключение Airflow](../01.%20astronomer-basic/connections.md), например через переменную окружения для соединения `smtp_default`:

```text
AIRFLOW_CONN_SMTP_DEFAULT='{
   "conn_type":"smtp",
   "host":"smtp.yourdomain.com",
   "port":<your-port>,
   "login":"<your-username>",
   "password":"<your-password>",
   "extra":{
      "disable_ssl":<your-setting>,
      "disable_tls":<your-setting>
   }
}'
```

Основные параметры [SmtpNotifier](https://registry.astronomer.io/providers/apache-airflow-providers-smtp/versions/latest/modules/SmtpNotifier):

- **custom_headers** — словарь дополнительных заголовков письма. По умолчанию: `None`.
- **files** — список путей к файлам для вложений. По умолчанию: `None`.
- **html_content** — HTML-тело письма. По умолчанию: `None`.
- **subject** — тема письма. По умолчанию: `None`.
- **from_email** — адрес отправителя. По умолчанию: `None`.
- **bcc** — скрытая копия (строка или список). По умолчанию: `None`.
- **cc** — копия (строка или список). По умолчанию: `None`.
- **to** — адрес или список адресов получателей. **Обязательный параметр.** По умолчанию: `None`.
- **smtp_conn_id** — ID подключения Airflow к SMTP-серверу. По умолчанию: `smtp_default`.

Экземпляр нотификатора передаётся в любой [параметр callback](#airflow-callbacks); при срабатывании callback’а будет отправлено письмо. Чтобы подставить данные DAG run, используйте [Jinja-шаблоны](jinja-templating.md). Все перечисленные параметры, кроме `smtp_conn_id`, поддерживают шаблоны.

Пример: отправить email при падении задачи с информацией о задаче, текстом ошибки (`{{ exception }}`) и ссылкой на лог (`{{ ti.log_url }}`):

```python
from airflow.sdk import task
from airflow.providers.smtp.notifications.smtp import SmtpNotifier


@task(
    on_failure_callback=SmtpNotifier(
        from_email="testnotifier@test.com",
        to=["primary@test.com"],
        cc=["manager@test.com", "team-lead@test.com"],
        bcc=["audit@test.com", "monitoring@test.com"],
        subject="{{ ti.task_id }} failed in {{ dag.dag_id }}",
        html_content="""
            <html>
                <body>
                    <h2 style="color: red;">Task Failure Alert</h2>
                    <p><strong>Task:</strong> {{ ti.task_id }}</p>
                    <p><strong>DAG:</strong> {{ dag.dag_id }}</p>
                    <p><strong>Execution Date:</strong> {{ ts }}</p>
                    <p><strong>Log URL:</strong> {{ ti.log_url }}</p>
                    <hr>
                    <h3>Error Details:</h3>
                    <pre>{{ exception }}</pre>
                </body>
            </html>
        """,
        files=["include/debug_info.json"],
        custom_headers={
            "X-Priority": "1",
            "X-Airflow-DAG": "{{ dag.dag_id }}",
            "X-Airflow-Task": "{{ ti.task_id }}",
            "Reply-To": "airflow-support@test.com"
        }
    )
)
def test_notifier_advanced():
    raise Exception("Oops, too much vibe coding!")
```

Для локальной проверки формата писем без реального SMTP можно использовать [MailHog](https://github.com/mailhog/MailHog) в Docker: `docker run -d -p 1025:1025 -p 8025:8025 mailhog/mailhog`, затем подключение:

```text
AIRFLOW_CONN_SMTP_DEFAULT='{
   "conn_type":"smtp",
   "host":"localhost",
   "port":1025,
   "login":"",
   "password":"",
   "extra":{
      "disable_ssl":true,
      "disable_tls":true
   }
}'
```

### Использование EmailOperator

**EmailOperator** создаёт в DAG отдельные задачи для отправки писем. Как и для SmtpNotifier, нужен [SMTP provider](https://registry.astronomer.io/providers/apache-airflow-providers-smtp/versions/latest) и [подключение Airflow](../01.%20astronomer-basic/connections.md) к SMTP. Основные параметры совпадают с SmtpNotifier.

```python
from airflow.providers.smtp.operators.smtp import EmailOperator

EmailOperator(
    task_id="send_email",
    conn_id="smtp_default",
    from_email="caller@mydomain.io",
    to="receiver@mydomain.io",
    subject="Test Email",
    html_content="This is a test email"
)
```

### (Legacy) Email через конфигурационные переменные

В старых версиях Airflow email часто настраивали через конфигурационные переменные и параметр задачи `email`. Такой способ устаревает в Airflow 3.0 и будет удалён.

Для настройки нужны и переменные [SMTP](https://airflow.apache.org/docs/apache-airflow/stable/configurations-ref.html#smtp), и параметр задачи `email`. Подключение `AIRFLOW_CONN_` с параметрами email в Airflow 3 использовать нельзя.

Переменные SMTP задают подключение к серверу:

```text
AIRFLOW__SMTP__SMTP_HOST=<your-smtp-host>
AIRFLOW__SMTP__SMTP_PORT=<your-port>
AIRFLOW__SMTP__SMTP_USER=<your-username>
AIRFLOW__SMTP__SMTP_PASSWORD=<your-password>
AIRFLOW__SMTP__SMTP_SSL=<your-setting>
AIRFLOW__SMTP__SMTP_TLS=<your-setting>
AIRFLOW__SMTP__SMTP_MAIL_FROM=<your-from-email>
```

Чтобы задача отправляла письмо при падении или повторе, у задачи нужно указать параметр `email` — кому отправлять. Обычно его задают в `default_args` DAG, чтобы применить ко всем задачам:

```python
from airflow.sdk import dag


@dag(
    default_args={
        "email": ["myname@mydomain.com"],
    }
)
def my_dag():
    ...
```

Либо у отдельной задачи, чтобы переопределить значение по умолчанию:

```python
from airflow.sdk import task


@task(email=["myfriend@mydomain.com"])
def test():
    print("Test")
    raise Exception("Test Exception")
```

Чтобы письма уходили только при падении, установите `email_on_retry=False`; только при повторе — `email_on_failure=False`.

Большинство переменных `AIRFLOW__EMAIL__` в Airflow 3.0 для SMTP больше не поддерживаются. Часть из них по-прежнему используется при SendGrid, Amazon SES и т.д. — см. [Email Configuration](https://airflow.apache.org/docs/apache-airflow/stable/howto/email-config.html).

## Airflow callbacks

В Airflow действия по разным состояниям DAG или задачи задаются параметрами `*_callback`:

- **on_retry_callback** — вызывается при повторе задачи. Только на уровне задачи.
- **on_execute_callback** — вызывается непосредственно перед выполнением задачи. Только на уровне задачи.
- **on_skipped_callback** — вызывается при пропуске задачи. Только на уровне задачи; только при выбросе `AirflowSkipException`, не при пропуске из-за trigger rule и т.п.
- **on_failure_callback** — при падении задачи или DAG.
- **on_success_callback** — при успешном завершении задачи или DAG.

В параметры `*_callback` можно передать любой Python callable или нотификатор Airflow (см. раздел «Готовые нотификаторы» ниже). Чтобы выполнить несколько функций, передайте список callback’ов в один параметр.

### Callback’и на уровне DAG

Чтобы задать уведомление на уровне DAG, укажите параметр `*_callback` при создании DAG. Callback’и уровня DAG срабатывают по конечному состоянию всего DAG run. В примере ниже при успехе DAG вызывается одна функция, при падении — две (своя функция и SlackNotifier):

```python
from airflow.sdk import dag
from airflow.providers.slack.notifications.slack_notifier import SlackNotifier


def my_success_callback_function(context):
    pass


def my_failure_callback_function(context):
    pass


@dag(
    on_success_callback=my_success_callback_function,
    on_failure_callback=[
        my_failure_callback_function,
        SlackNotifier(
            slack_conn_id="slack_conn",
            text="Dag failed",
            channel="alerts"
        )
    ],
)
def my_dag():
    ...
```

В Airflow 3.1 добавлены deadline alerts (экспериментально) — срабатывают, когда DAG run превышает заданный порог по времени; они заменяют удалённую SLA-функцию с параметрами `sla` и `sla_miss_callback`. Подробнее: [Deadline alerts](https://airflow.apache.org/docs/apache-airflow/stable/howto/deadline-alerts.html).

Клиентам Astronomer для SLA по актуальности и свежести данных рекомендуется использовать [Astro Alerts](https://www.astronomer.io/docs/astro/alerts) и [Astro Observe](https://www.astronomer.io/docs/astro/astro-observe).

### Callback’и на уровне задачи

Чтобы применить callback ко всем задачам DAG, передайте функцию в `default_args`. Элементы словаря `default_args` задаются для каждой задачи. В примере у каждого типа callback по одной функции; в один параметр можно передать список из нескольких callback’ов и/или нотификаторов.

```python
from airflow.sdk import dag


def my_execute_callback_function(context):
    pass


def my_retry_callback_function(context):
    pass


def my_success_callback_function(context):
    pass


def my_failure_callback_function(context):
    pass


def my_skipped_callback_function(context):
    pass


@dag(
    default_args={
        "on_execute_callback": my_execute_callback_function,
        "on_retry_callback": my_retry_callback_function,
        "on_success_callback": my_success_callback_function,
        "on_failure_callback": my_failure_callback_function,
        "on_skipped_callback": my_skipped_callback_function,
    }
)
def my_dag():
    ...
```

Если callback нужен только одной задаче, задайте параметры callback при создании этой задачи. Callback’и на уровне задачи переопределяют переданные через `default_args`.

**TaskFlow:**

```python
from airflow.sdk import task


def my_execute_callback_function(context):
    pass


def my_retry_callback_function(context):
    pass


def my_success_callback_function(context):
    pass


def my_failure_callback_function(context):
    pass


def my_skipped_callback_function(context):
    pass


@task(
    on_execute_callback=my_execute_callback_function,
    on_retry_callback=my_retry_callback_function,
    on_success_callback=my_success_callback_function,
    on_failure_callback=my_failure_callback_function,
    on_skipped_callback=my_skipped_callback_function,
)
def t1():
    return "hello"
```

**Традиционный вариант:**

```python
from airflow.providers.standard.operators.python import PythonOperator


def my_execute_callback_function(context):
    pass


def my_retry_callback_function(context):
    pass


def my_success_callback_function(context):
    pass


def my_failure_callback_function(context):
    pass


def my_skipped_callback_function(context):
    pass


def say_hello():
    return "hello"


t1 = PythonOperator(
    task_id="t1",
    python_callable=say_hello,
    on_execute_callback=my_execute_callback_function,
    on_retry_callback=my_retry_callback_function,
    on_success_callback=my_success_callback_function,
    on_failure_callback=my_failure_callback_function,
    on_skipped_callback=my_skipped_callback_function,
)
```

### Готовые нотификаторы

[Нотификаторы Airflow](https://airflow.apache.org/docs/apache-airflow/stable/howto/notifications.html) — готовые или кастомные классы для унификации кода уведомлений. Их можно передавать в нужный параметр `*_callback` DAG в зависимости от события.

Полный список готовых нотификаторов провайдеров: [здесь](https://airflow.apache.org/docs/apache-airflow-providers/core-extensions/notifications.html). К [многим другим сервисам](https://github.com/caronc/apprise/wiki) можно подключаться через [AppriseNotifier](https://airflow.apache.org/docs/apache-airflow-providers-apprise/stable/_api/airflow/providers/apprise/notifications/apprise/index.html).

Нотификаторы определяются в пакетах провайдеров или импортируются из папки `include` и могут использоваться в любых DAG. Плюс подхода — общие сценарии можно оформить как модули Airflow и переиспользовать.

#### Пример: SlackNotifier

Пример готового нотификатора от сообщества — [SlackNotifier](https://airflow.apache.org/docs/apache-airflow-providers-slack/stable/_api/airflow/providers/slack/notifications/slack/index.html#module-airflow.providers.slack.notifications.slack). Он импортируется из провайдера Slack и подставляется в любой `*_callback`:

```python
"""
Пример использования SlackNotifier. Нужно подключение Slack с API Token бота (начинается с 'xoxb-...').
"""

from airflow.sdk import dag, task
from pendulum import datetime
from airflow.providers.slack.notifications.slack_notifier import SlackNotifier

SLACK_CONNECTION_ID = "slack_conn"
SLACK_CHANNEL = "alerts"
SLACK_MESSAGE = """
Hello! The {{ ti.task_id }} task is saying hi :wave:
Today is the {{ ds }} and this task finished with the state: {{ ti.state }} :tada:.
"""


@dag
def slack_notifier_example_dag():
    @task(
        on_success_callback=SlackNotifier(
            slack_conn_id=SLACK_CONNECTION_ID,
            text=SLACK_MESSAGE,
            channel=SLACK_CHANNEL,
        ),
    )
    def post_to_slack():
        return 10

    post_to_slack()


slack_notifier_example_dag()
```

В DAG одна задача отправляет уведомление в Slack. Используется [подключение Airflow](../01.%20astronomer-basic/connections.md) с ID `slack_conn`.

### Кастомные нотификаторы

Если под вашу задачу нет готового нотификатора, можно написать свой. Нотификатор Airflow — класс, наследующийся от `BaseNotifier`, с реализацией действия в методе `.notify()`:

```python
from airflow.sdk import BaseNotifier


class MyNotifier(BaseNotifier):
    """
    Простой нотификатор: выводит task_id, состояние и сообщение.
    """

    template_fields = ("message",)

    def __init__(self, message):
        self.message = message

    def notify(self, context):
        t_id = context["ti"].task_id
        t_state = context["ti"].state
        print(
            f"Hi from MyNotifier! {t_id} finished as: {t_state} and says {self.message}"
        )
```

Использование в DAG — передать экземпляр нотификатора в любой параметр callback.

**TaskFlow:**

```python
from airflow.sdk import task


def say_hello():
    return "hello"


@task(
    on_failure_callback=MyNotifier(message="Hello failed!"),
)
def t1():
    return "hello"
```

**Традиционный вариант:**

```python
from airflow.providers.standard.operators.python import PythonOperator


def say_hello():
    return "hello"


t1 = PythonOperator(
    task_id="t1",
    python_callable=say_hello,
    on_failure_callback=MyNotifier(message="Hello failed!"),
)
```

---

[← Декораторы](airflow-decorators.md) | [К содержанию](README.md) | [Параметры →](airflow-params.md)
