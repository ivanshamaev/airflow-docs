# Параметры DAG (DAG parameters)

В Airflow момент и способ запуска DAG настраиваются параметрами объекта DAG. Параметры уровня DAG влияют на поведение всего DAG в отличие от параметров уровня задачи (одна задача) и [конфигурации Airflow](https://airflow.apache.org/docs/apache-airflow/stable/configurations-ref.html) (весь инстанс).

В этом руководстве перечислены все актуальные для пользователя параметры DAG.

## Базовые параметры DAG

Есть четыре базовых параметра. Рекомендуется задавать их в каждом DAG:

| Параметр | Описание |
| --- | --- |
| **dag_id** | Имя DAG. Должно быть уникальным в окружении Airflow. При использовании декоратора `@dag` без явного `dag_id` в качестве имени берётся имя функции. При использовании класса `DAG` параметр обязателен. |
| **start_date** | Дата и время, после которых DAG начинает планироваться. Фактический первый запуск может быть позже в зависимости от расписания. Подробнее: [Планирование и timetables в Airflow](../01.%20astronomer-basic/scheduling.md). По умолчанию: `None`. |
| **schedule** | Расписание DAG. Варианты задания: [Планирование в Airflow](../01.%20astronomer-basic/scheduling.md). По умолчанию: `None`. |
| **catchup** | Нужно ли планировщику создавать пропущенные DAG run между текущей датой и start_date при включении DAG. По умолчанию: `False`. Подробнее: [Catchup](rerunning-dags.md). |

## Параметры отображения в UI

Часть параметров [добавляет описание](../05.%20astronomer-write-dags/dag-documentation.md) DAG или меняет его вид в Airflow UI:

| Параметр | Описание |
| --- | --- |
| **description** | Короткая строка, отображаемая в UI рядом с именем DAG. |
| **doc_md** | Строка, которая показывается как [документация DAG](../05.%20astronomer-write-dags/dag-documentation.md) в UI. Совет: можно использовать `__doc__`, чтобы подставить docstring файла. Рекомендуется задавать осмысленную документацию для всех DAG. |
| **tags** | Список тегов для фильтрации DAG в UI. |

## Параметры Jinja-шаблонов

Параметры, связанные с [Jinja-шаблонированием](jinja-templating.md):

| Параметр | Описание |
| --- | --- |
| **template_searchpath** | Список каталогов, в которых Jinja ищет шаблоны. По умолчанию учитывается каталог с файлом DAG. |
| **template_undefined** | Поведение Jinja при неопределённой переменной. По умолчанию: [StrictUndefined](https://jinja.palletsprojects.com/en/3.0.x/api/#jinja2.StrictUndefined). |
| **render_template_as_native_obj** | Рендерить ли шаблоны Jinja как нативные объекты Python вместо строк. По умолчанию: `False`. |
| **user_defined_macros** | Словарь макросов, доступных в Jinja-шаблонах DAG. Для фильтров используйте `user_defined_filters`, для дополнительной настройки Jinja — `jinja_environment_kwargs`. См. [Макросы: свои функции и переменные в шаблонах](jinja-templating.md). |

## Масштабирование

Следующие параметры влияют на использование ресурсов DAG. Подробнее: [Scaling Airflow](../03.%20astronomer-infra/scaling-airflow.md).

| Параметр | Описание |
| --- | --- |
| **max_active_tasks** | Максимальное число экземпляров задач, которые могут выполняться одновременно в рамках одного запуска этого DAG. |
| **max_active_runs** | Максимальное число одновременных активных DAG run для этого DAG. |
| **max_consecutive_failed_dag_runs** | (экспериментальный) После какого числа подряд неудачных DAG run планировщик отключает этот DAG. |

## Параметры callback'ов

Эти параметры настраивают [callback'и Airflow](airflow-notifications.md#airflow-callbacks):

| Параметр | Описание |
| --- | --- |
| **on_success_callback** | Функция, вызываемая после успешного завершения DAG run. |
| **on_failure_callback** | Функция, вызываемая после неудачного завершения DAG run. |

> В Astro вместо или вместе с callback'ами Airflow можно использовать Astro Alerts. См. [When to use Airflow or Astro alerts](https://www.astronomer.io/docs/astro/best-practices/airflow-vs-astro-alerts).

## Прочие параметры

Другие параметры DAG:

| Параметр | Описание |
| --- | --- |
| **end_date** | Дата, после которой новые DAG run для этого DAG не создаются. По умолчанию: `None`. |
| **default_args** | Словарь параметров, применяемых ко всем задачам DAG. Параметры передаются операторам, поэтому должны быть допустимыми для BaseOperator. Значения можно переопределять на уровне задачи. |
| **params** | Словарь параметров уровня DAG (Airflow params). Подробнее: [Airflow params](airflow-params.md). |
| **dagrun_timeout** | Время, по истечении которого DAG run этого DAG считается просроченным и помечается как `failed`. |
| **access_control** | Права доступа по ролям для данного DAG. См. [DAG-level permissions](https://airflow.apache.org/docs/apache-airflow/stable/security/access-control.html#dag-level-permissions). На Astro не поддерживается; рекомендуется [RBAC в Astro](https://www.astronomer.io/docs/astro/user-permissions). |
| **is_paused_upon_creation** | Создавать ли DAG в состоянии паузы. Если не задано, используется конфиг `core.dags_are_paused_at_creation` (по умолчанию `True`). |
| **auto_register** | По умолчанию `True`. При `False` DAG, созданные через контекст `with`, не регистрируются автоматически; используется в сценариях [динамической генерации DAG](https://airflow.apache.org/docs/apache-airflow/stable/howto/dynamic-dag-generation.html#registering-dynamic-dags). |
| **fail_fast** | При `True` выполнение DAG останавливается при первой упавшей задаче. Все ещё выполняющиеся задачи помечаются как `failed`, ещё не запущенные — как `skipped`. В DAG с `fail_fast=True` нельзя использовать [trigger rule](../01.%20astronomer-basic/trigger-rules.md), отличные от `all_success`. |
| **dag_display_name** | Имя DAG в Airflow UI (переопределяет `dag_id`). Допускает спецсимволы. |

---

[← Лучшие практики](dag-best-practices.md) | [К содержанию](README.md) | [Params →](airflow-params.md)
