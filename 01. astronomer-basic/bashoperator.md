# Использование BashOperator

[BashOperator](https://registry.astronomer.io/providers/apache-airflow/modules/bashoperator) выполняет bash-команды или bash-скрипты из DAG. Декоратор `@task.bash` (с Airflow 2.9) позволяет формировать команду в Python-функции.

## Использование

**Традиционный способ:**

```python
from airflow.providers.standard.operators.bash import BashOperator

bash_task = BashOperator(
    task_id="bash_task",
    bash_command="echo $MY_VAR",
    env={"MY_VAR": "Hello World"}
)
```

**TaskFlow:**

```python
from airflow.sdk import task

@task.bash(env={"MY_VAR": "Hello World"})
def bash_task():
    return "echo $MY_VAR"  # возвращаемая строка выполняется как bash-команда

bash_task()
```

## Параметры

- **bash_command** — одна команда, несколько команд или путь к скрипту `.sh` (обязателен).
- **env** — словарь переменных окружения. По умолчанию заменяет все переменные окружения; при **append_env=True** дополняет существующие.
- **append_env** — дополнять ли `env` к текущему окружению (по умолчанию False).
- **output_encoding** — кодировка вывода (по умолчанию utf-8).
- **skip_on_exit_code** — код выхода bash, при котором задача помечается как skipped (по умолчанию 99).
- **cwd** — рабочая директория (по умолчанию None — временная директория).

Поведение по коду выхода:

- Код 0 — успех.
- Код 99 (или заданный в skip_on_exit_code) — skipped.
- Остальные — failed.

Для перехвата ненулевого кода подкоманды можно добавить в начало команды `set -e;`.

Параметры `bash_command` и `env` поддерживают [Jinja-шаблоны](https://www.astronomer.io/docs/learn/templating).

## Когда использовать

- Формирование bash-команд на основе сложной Python-логики (в т.ч. выхода вышестоящих задач).
- Выполнение одной или нескольких bash-команд.
- Запуск подготовленного bash-скрипта.
- Запуск скриптов на других языках (через интерпретатор: `node`, `Rscript` и т.д.).
- Инициализация инструментов без отдельного оператора (например, Soda Core).

## Пример: две команды в одной задаче

Команды объединяются через `&&`:

```python
say_hello_and_create_a_secret_number = BashOperator(
    task_id="say_hello_and_create_a_secret_number",
    bash_command="echo Hello $MY_NAME! && echo $A_LARGE_NUMBER | rev 2>&1 | tee $AIRFLOW_HOME/include/my_secret_number.txt",
    env={"MY_NAME": "<my name>", "A_LARGE_NUMBER": "231942"},
    append_env=True,
)
```

## Пример: выполнение bash-скрипта

Скрипт должен быть доступен окружению Airflow (например, в `include/`) и исполняемым (`chmod +x script.sh`). В параметр передаётся путь; в конце пути рекомендуется пробел (из-за Jinja):

```python
execute_my_script = BashOperator(
    task_id="execute_my_script",
    bash_command="$AIRFLOW_HOME/include/my_bash_script.sh ",
)
```

---

[← Операторы](operators.md) | [К содержанию](README.md) | [Connections →](connections.md)
