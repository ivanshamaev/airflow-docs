# BashOperator

[BashOperator](https://registry.astronomer.io/providers/apache-airflow/modules/bashoperator) — один из самых часто используемых операторов в Airflow. Он выполняет bash-команды или bash-скрипт из вашего DAG Airflow.

В этом руководстве вы узнаете:

- Как запускать скрипты на языках, отличных от Python, с помощью BashOperator.
- Как использовать BashOperator: выполнение bash-команд и bash-скриптов.
- Как использовать BashOperator и декоратор `@task.bash`.
- Когда использовать BashOperator.

## Необходимая база

Чтобы получить максимум от руководства, нужно понимать:

- Основные bash-команды. См. [Bash Reference Manual](https://www.gnu.org/software/bash/manual/bash.html).
- Декораторы Airflow. См. [Введение в TaskFlow API и декораторы Airflow](../02.%20astronomer-dags/airflow-decorators.md).
- Операторы Airflow. См. [Операторы](operators.md).

## Как использовать BashOperator и декоратор `@task.bash`

BashOperator входит в ядро Airflow и может выполнять одну bash-команду, набор bash-команд или bash-скрипт с расширением `.sh`. Декоратор `@task.bash` позволяет формировать bash-команды в виде Python-функций и доступен начиная с Airflow 2.9.

```python
from airflow.providers.standard.operators.bash import BashOperator

bash_task = BashOperator(
    task_id="bash_task",
    bash_command="echo $MY_VAR",
    env={"MY_VAR": "Hello World"}
)
```

```python
from airflow.sdk import task

@task.bash(env={"MY_VAR": "Hello World"})
def bash_task():
    return "echo $MY_VAR"  # возвращаемая строка выполняется как bash-команда

bash_task()
```

Оператору и декоратору можно передавать следующие параметры:

- **cwd**: рабочая директория для выполнения bash-команды. По умолчанию `None` — команда выполняется во временной директории.
- **skip_on_exit_code**: код выхода bash, при котором задача переходит в состояние `skipped`. По умолчанию `99`.
- **output_encoding**: кодировка вывода bash-команды. По умолчанию `utf-8`.
- **append_env**: меняет поведение параметра `env`. При `True` переменные из `env` дополняют существующие переменные окружения, а не перезаписывают их. По умолчанию `False`.
- **env**: словарь переменных окружения для bash-процесса. По умолчанию этот словарь **полностью заменяет** все переменные окружения Airflow, в том числе не указанные в словаре. Чтобы дополнять, а не заменять, используйте параметр `append_env`. Если `env` не задан, BashOperator наследует переменные окружения Airflow.
- **bash_command**: одна bash-команда, набор команд или путь к bash-скрипту для выполнения. Параметр обязателен.

Поведение задачи BashOperator зависит от кода выхода bash-оболочки:

- Задача завершается **успешно**, если оболочка выходит с кодом 0.
- Задача переходит в **skipped**, если код выхода 99 (или значение из `skip_on_exit_code`).
- Задача завершается **ошибкой** при любом другом коде выхода.

> **Совет.** Если подкоманда может вернуть ненулевой код выхода, добавьте в начало команды префикс `set -e;`, чтобы этот код воспринимался как сбой задачи.

Параметры `bash_command` и `env` поддерживают [Jinja-шаблоны](../02.%20astronomer-dags/jinja-templating.md). Ввод, подставляемый через Jinja в `bash_command`, не экранируется и не санитизируется. Если возможен вредоносный пользовательский ввод, см. [документацию BashOperator](https://airflow.apache.org/docs/apache-airflow/stable/howto/operator/bash.html).

## Когда использовать BashOperator

Типичные сценарии для BashOperator и декоратора `@task.bash` в DAG Airflow:

- Запуск команд для инициализации инструментов, для которых нет отдельного оператора (например, [Soda Core](https://www.astronomer.io/docs/learn/soda-data-quality)).
- Запуск скриптов на языках, отличных от Python.
- Запуск заранее подготовленного bash-скрипта.
- Выполнение одной или нескольких bash-команд в окружении Airflow.
- Формирование и выполнение bash-команд на основе сложной Python-логики.

## Пример: формирование bash-команд с помощью Python

Декоратор `@task.bash` позволяет формировать bash-команды в Python-функциях. Это удобно, когда команда зависит от сложной логики или от выхода вышестоящих задач. В примере ниже показано, как выполнять разные bash-команды в зависимости от результата upstream-задачи.

```python
from airflow.sdk import task

@task
def upstream_task():
    dog_owner_data = {
        "names": ["Trevor", "Grant", "Marcy", "Carly", "Philip"],
        "dogs": [1, 2, 2, 0, 4],
    }

    return dog_owner_data

@task.bash
def bash_task(dog_owner_data):
    names_of_dogless_people = []
    for name, dog in zip(dog_owner_data["names"], dog_owner_data["dogs"]):
        if dog < 1:
            names_of_dogless_people.append(name)

    if names_of_dogless_people:
        if len(names_of_dogless_people) == 1:
            # эта bash-команда выполняется, если без собаки только один человек
            return f'echo "{names_of_dogless_people[0]} urgently needs a dog!"'
        else:
            names_of_dogless_people_str = " and ".join(names_of_dogless_people)
            # эта bash-команда выполняется, если без собаки больше одного человека
            return f'echo "{names_of_dogless_people_str} urgently need a dog!"'
    else:
        # эта bash-команда выполняется, если у всех есть хотя бы одна собака
        return f'echo "All good, everyone has at least one dog!"'

bash_task(dog_owner_data=upstream_task())
```

## Пример: выполнение двух bash-команд одним BashOperator

BashOperator может выполнять любое количество bash-команд, разделённых `&&`.

В примере в одной задаче выполняются две команды:

- `echo $A_LARGE_NUMBER | rev 2>&1 | tee $AIRFLOW_HOME/include/my_secret_number.txt` — берёт переменную окружения `A_LARGE_NUMBER`, передаёт её в `rev` (переворот строки) и сохраняет результат в файл `my_secret_number.txt` в каталоге `/include`. Перевёрнутое число также выводится в консоль.
- `echo Hello $MY_NAME!` — выводит переменную окружения `MY_NAME` в консоль.

Вторая команда использует переменную окружения Airflow `AIRFLOW_HOME`. Это возможно только потому, что задано `append_env=True`.

```python
from airflow.decorators import dag
from airflow.operators.bash import BashOperator
from pendulum import datetime


@dag(start_date=datetime(2022, 8, 1), schedule=None, catchup=False)
def bash_two_commands_example_dag():
    say_hello_and_create_a_secret_number = BashOperator(
        task_id="say_hello_and_create_a_secret_number",
        bash_command="echo Hello $MY_NAME! && echo $A_LARGE_NUMBER | rev  2>&1\
                     | tee $AIRFLOW_HOME/include/my_secret_number.txt",
        env={"MY_NAME": "<my name>", "A_LARGE_NUMBER": "231942"},
        append_env=True,
    )

    say_hello_and_create_a_secret_number


bash_two_commands_example_dag()
```

Можно также использовать два отдельных BashOperator для двух команд — это полезно, если нужно задать разным задачам разные зависимости.

## Пример: выполнение bash-скрипта

BashOperator может выполнять bash-скрипт (файл с расширением `.sh`).

В примере запускается скрипт, который перебирает все файлы в каталоге `/include` и выводит их имена в консоль.

```bash
#!/bin/bash

echo "The script is starting!"
echo "The current user is $(whoami)"
files = $AIRFLOW_HOME/include/*

for file in $files
do
    echo "The include folder contains $(basename $file)"
done

echo "The script has run. Have an amazing day!"
```

Убедитесь, что bash-скрипт (в примере `my_bash_script.sh`) доступен окружению Airflow. При использовании Astro CLI поместите файл в каталог `/include` вашего проекта Astro.

Важно сделать скрипт исполняемым перед тем, как он станет доступен Airflow:

```bash
chmod +x my_bash_script.sh
```

При использовании Astro CLI эту команду можно выполнить перед `astro dev start` или добавить в Dockerfile проекта:

```dockerfile
RUN chmod +x /usr/local/airflow/include/my_bash_script.sh
```

Astronomer рекомендует выполнять эту команду в Dockerfile для production-сборок (Astro Deployments или production CI/CD).

После того как скрипт доступен Airflow, в параметр `bash_command` передаётся только путь к скрипту. **Обязательно добавьте пробел в конце пути**, иначе задача упадёт с ошибкой Jinja!

```python
from airflow.decorators import dag
from airflow.operators.bash import BashOperator
from pendulum import datetime


@dag(start_date=datetime(2022, 8, 1), schedule=None, catchup=False)
def bash_script_example_dag():
    execute_my_script = BashOperator(
        task_id="execute_my_script",
        # Обратите внимание на пробел в конце команды!
        bash_command="$AIRFLOW_HOME/include/my_bash_script.sh ",
        # так как env не задан, этот BashOperator наследует переменные
        # окружения экземпляра Airflow, в том числе AIRFLOW_HOME
    )

    execute_my_script


bash_script_example_dag()
```

## Пример: запуск скрипта на другом языке программирования

BashOperator — простой способ запустить в Airflow скрипт на языке, отличном от Python. Можно запускать скрипт на любом языке, который вызывается bash-командой.

В примере выполняется JavaScript, запрашивающий публичный API [текущего местоположения МКС](http://open-notify.org/Open-Notify-API/ISS-Location-Now/). Результат запроса передаётся в XCom, чтобы вторая задача извлекла широту и долготу в скрипте на R и вывела данные в консоль.

Что нужно подготовить:

- Выполнять файлы из DAG через BashOperator.
- Сделать скрипты доступными окружению Airflow.
- Написать R-скрипт.
- Написать JavaScript-файл.
- Установить пакеты JavaScript и R на уровне ОС.

При использовании Astro CLI пакеты языков можно установить на уровне ОС, добавив их в файл `packages.txt` проекта Astro:

```text
r-base
nodejs
```

Следующий JavaScript-файл отправляет GET-запрос к `/iss-now` на `api.open-notify.org` и возвращает результат в `stdout`; вывод будет и выведен в консоль, и передан в XCom через BashOperator.

```javascript
// указание на HTTP API
const http = require('http');

const options = {
  hostname: 'api.open-notify.org',
  port: 80,
  path: '/iss-now',
  method: 'GET',
};

const req = http.request(options, res => {
  console.log(`statusCode: ${res.statusCode}`);

  res.on('data', d => {
    process.stdout.write(d);
  });
});

req.on('error', error => {
  console.error(error);
});

req.end();
```

Вторая задача запускает R-скрипт, который с помощью regex извлекает и выводит долготу и широту из ответа API.

```r
# Вывод в консоль
options(echo = TRUE)

# Читаем все аргументы командной строки
myargs <- commandArgs(trailingOnly = TRUE)

# Собираем в одну JSON-строку
json_string <- paste(myargs, collapse = " ")

# Извлекаем latitude и longitude через regex
latitude_str <- sub('.*"latitude"\\s*:\\s*"([^"]+)".*', '\\1', json_string)
longitude_str <- sub('.*"longitude"\\s*:\\s*"([^"]+)".*', '\\1', json_string)

latitude <- as.numeric(latitude_str)
longitude <- as.numeric(longitude_str)

sprintf("The current ISS location: lat: %s / long: %s.", latitude, longitude)
```

Чтобы запустить эти скрипты через BashOperator, они должны быть доступны окружению Airflow. При использовании Astro CLI поместите файлы в каталог `/include` проекта Astro.

DAG использует BashOperator для последовательного выполнения обоих скриптов.

```python
from airflow.decorators import dag
from airflow.operators.bash import BashOperator
from pendulum import datetime


@dag(
    dag_id="print_ISS_info_dag",
    start_date=datetime(2022, 8, 1),
    schedule=None,
    catchup=False,
)
def print_ISS_info_dag():
    # Команда node выполняет JavaScript-файл из командной строки
    get_ISS_coordinates = BashOperator(
        task_id="get_ISS_coordinates",
        bash_command="node $AIRFLOW_HOME/include/my_java_script.js",
    )

    # Команда Rscript выполняет R-файл; результат первой задачи передаётся
    # через переменную окружения (XCom)
    print_ISS_coordinates = BashOperator(
        task_id="print_ISS_coordinates",
        bash_command="Rscript $AIRFLOW_HOME/include/my_R_script.R $ISS_COORDINATES",
        env={
            "ISS_COORDINATES": "{{ task_instance.xcom_pull(\
                               task_ids='get_ISS_coordinates', \
                               key='return_value') }}"
        },
        append_env=True,  # чтобы использовать переменные окружения Airflow, например AIRFLOW_HOME
    )

    get_ISS_coordinates >> print_ISS_coordinates


print_ISS_info_dag()
```

---

[← Операторы](operators.md) | [К содержанию](README.md) | [Connections →](connections.md)
