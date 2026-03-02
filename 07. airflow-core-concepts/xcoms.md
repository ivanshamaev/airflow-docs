# XComs

**XComs** (сокращение от «cross-communications») — механизм обмена данными между [задачами](tasks.md). По умолчанию задачи изолированы и могут выполняться на разных машинах.

XCom идентифицируется по `key` (по сути имя), а также по `task_id` и `dag_id` задачи-источника. В качестве значения может быть любой сериализуемый объект (в том числе с декораторами `@dataclass` или `@attr.define`, см. [TaskFlow arguments](taskflow.md#передача-произвольных-объектов-в-качестве-аргументов)), но XCom рассчитан на небольшой объём данных; не используйте его для передачи больших значений, например датафреймов.

Работать с XCom нужно через контекст задачи и [get_current_context()](https://airflow.apache.org/docs/task-sdk/stable/api.html#airflow.sdk.get_current_context). Прямое изменение через модель XCom в БД недоступно.

Значения явно «пушатся» и «пулятся» в хранилище методами `xcom_push` и `xcom_pull` у Task Instance.

Чтобы отправить значение из задачи «task-1» для использования в другой задаче:

```python
# помещает any_serializable_value в XCom с ключом "identifier as string"
task_instance.xcom_push(key="identifier as a string", value=any_serializable_value)
```

Чтобы получить это значение в другой задаче:

```python
# получает XCom с ключом "identifier as string", записанный в задаче task-1
task_instance.xcom_pull(key="identifier as string", task_ids="task-1")
```

Многие операторы по умолчанию автоматически пушат результат в XCom с ключом `return_value`, если задан аргумент `do_xcom_push=True` (он включён по умолчанию). То же делают функции с декоратором `@task`. У `xcom_pull` ключ по умолчанию — `return_value`, если ключ не передан, поэтому можно писать так:

```python
# Получает XCom return_value из задачи "pushing_task"
value = task_instance.xcom_pull(task_ids='pushing_task')
```

XCom можно использовать в [шаблонах](operators.md#jinja-templating-шаблонизация-jinja):

```python
SELECT * FROM {{ task_instance.xcom_pull(task_ids='foo', key='table_name') }}
```

XCom родственен [Variables](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/variables.html): XCom привязан к экземпляру задачи и предназначен для обмена данными внутри одного Dag run, а Variables глобальны и служат для общей конфигурации и общих значений.

Чтобы отправить несколько XCom за раз, задайте аргументы `do_xcom_push` и `multiple_outputs` в `True` и верните словарь значений.

Пример отправки нескольких XCom и получения их по отдельности:

```python
# Задача возвращает словарь
@task(do_xcom_push=True, multiple_outputs=True)
def push_multiple(**context):
    return {"key1": "value1", "key2": "value2"}


@task
def xcom_pull_with_multiple_outputs(**context):
    # Получение конкретного ключа из нескольких выходов
    key1 = context["ti"].xcom_pull(task_ids="push_multiple", key="key1")  # key1
    key2 = context["ti"].xcom_pull(task_ids="push_multiple", key="key2")  # key2

    # Получение всех данных XCom из задачи push_multiple
    data = context["ti"].xcom_pull(task_ids="push_multiple", key="return_value")
```

> **Примечание.** Если первый запуск задачи не завершился успехом, при каждой повторной попытке (retry) XCom этой задачи очищаются, чтобы запуск оставался идемпотентным.

## Object Storage XCom Backend

Стандартный бэкенд XCom, BaseXCom, хранит данные в БД Airflow. Это удобно для небольших значений, но может вызывать проблемы при больших объёмах или большом числе XCom. Для эффективной работы с крупными данными рекомендуется бэкенд на объектном хранилище. Подробнее: [документация](https://airflow.apache.org/docs/apache-airflow-providers-common-io/stable/xcom_backend.html).

## Кастомные бэкенды XCom

У системы XCom сменяемые бэкенды; нужный задаётся опцией конфигурации `xcom_backend`.

Чтобы реализовать свой бэкенд, создайте подкласс `BaseXCom` и переопределите методы `serialize_value` и `deserialize_value`.

Метод `purge` в классе `BaseXCom` можно переопределить, чтобы управлять удалением данных XCom из кастомного бэкенда. Он вызывается в рамках операции `delete`.

## Проверка использования кастомного бэкенда XCom в контейнерах

В зависимости от окружения развёртывания (локально, Docker, K8s и т.д.) бывает важно убедиться, что кастомный бэкенд XCom действительно инициализируется. В контейнерной среде сложнее проверить, что бэкенд загружается при старте. Ниже — способ проверить, какой класс XCom используется.

Если есть доступ к терминалу внутри контейнера Airflow, можно вывести фактический класс XCom:

```python
from airflow.sdk.execution_time.xcom import XCom

print(XCom.__name__)
```

---

*Источник: [Airflow 3.1.7 — XComs](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/xcoms.html). Перевод неофициальный.*
