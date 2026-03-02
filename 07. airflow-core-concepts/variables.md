# Variables (переменные)

**Variables** — механизм runtime-конфигурации в Airflow: общее глобальное хранилище «ключ — значение», к которому можно обращаться из задач, задавать значения через UI Airflow или загружать пачкой из JSON-файла.

Чтобы использовать переменные, импортируйте модель Variable и вызовите метод `get`:

```python
from airflow.sdk import Variable

# Обычный вызов
foo = Variable.get("foo")

# Автоматическая десериализация JSON-значения
bar = Variable.get("bar", deserialize_json=True)

# Возвращает default (None), если переменная не задана
baz = Variable.get("baz", default=None)
```

Доступ к переменным возможен и через контекст задачи с помощью [get_current_context()](https://airflow.apache.org/docs/task-sdk/stable/api.html#airflow.sdk.get_current_context):

```python
from airflow.sdk import get_current_context


def my_task():
    context = get_current_context()
    var = context["var"]
    my_variable = var.get("my_variable_name")
    return my_variable
```

Их можно использовать в [шаблонах](operators.md#jinja-templating-шаблонизация-jinja):

```python
# Сырое значение
echo {{ var.value.<variable_name> }}

# Автодесериализация JSON
echo {{ var.json.<variable_name> }}
```

Variables глобальны и предназначены только для общей конфигурации всей установки. Для передачи данных между задачами/операторами используйте [XComs](xcoms.md).

Рекомендуется по возможности хранить настройки и конфигурацию в файлах DAG, чтобы они версионировались в системе контроля версий. Variables уместны лишь для значений, которые действительно зависят от runtime.

Подробнее о создании и управлении переменными: [Managing Variables](https://airflow.apache.org/docs/apache-airflow/stable/howto/variable.html).

---

*Источник: [Airflow 3.1.7 — Variables](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/variables.html). Перевод неофициальный.*
