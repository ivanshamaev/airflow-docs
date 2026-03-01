# Синхронное выполнение DAG (Synchronous dag execution)

**Синхронное выполнение DAG** — возможность в Airflow 3.1+ запустить DAG run через API и дождаться его завершения, после чего получить значения [XCom](../02.%20astronomer-dags/passing-data-between-tasks.md), запушенные одной или несколькими задачами этого DAG run. Это удобно и для одиночного запуска DAG, и когда один и тот же DAG может запускаться много раз параллельно.

Синхронное выполнение DAG добавлено как [экспериментальная функция](https://airflow.apache.org/docs/apache-airflow/stable/release-process.html#experimental-features) в Airflow 3.1.

## Необходимая база

- Понимание основ XCom. См. [Передача данных между задачами](../02.%20astronomer-dags/passing-data-between-tasks.md).
- Умение пользоваться [REST API Airflow](https://airflow.apache.org/docs/apache-airflow/stable/stable-rest-api-ref.html).
- Основы Airflow. См. [Введение в Apache Airflow](../01.%20astronomer-basic/README.md).

## Когда использовать синхронное выполнение DAG

Синхронное выполнение DAG позволяет использовать Airflow как бэкенд для сервисов, обрабатывающих запросы пользователей от фронтенда: сайта, мобильного приложения или бота в Slack. Типичные сценарии:

- **Отправка данных:** нетехнические пользователи отправляют данные в DAG на обработку и сразу получают статус запроса и результат.
- **Ad-hoc запросы:** заинтересованные стороны запрашивают разовые анализы данных; DAG возвращает нужный результат.
- **Инференс:** пользователь передаёт входные данные в пайплайн, который обращается к одному или нескольким LLM и/или AI-агентам и формирует ответ. Ответ возвращается пользователю после завершения DAG.

## API-эндпоинт

Эндпоинт для ожидания завершения DAG run:

```text
GET api/v2/dags/{dag_id}/dagRuns/{dag_run_id}/wait
```

Параметры пути:

- **`dag_run_id`:** (обязательный) идентификатор DAG run, завершения которого ждём.
- **`dag_id`:** (обязательный) идентификатор DAG.

Параметры запроса (query):

- **`result`:** (опционально) массив строк или null. Список task_id, для которых нужно вернуть значение XCom по ключу `return_value`.
- **`interval`:** (обязательный) интервал в секундах между проверками состояния DAG run.

Вызов этого эндпоинта для запущенного DAG запускает процесс ожидания до завершения DAG run. Если в параметре `result` указаны XCom, они возвращаются в ответе после завершения DAG run.

## Пример скрипта

Скрипт ниже создаёт DAG run для DAG `my_dag`, дожидается его завершения и включает в ответ XCom, запушенные под ключом `return_value` задачи `my_task`.

```python
import requests
from datetime import datetime
import json

_USERNAME = "admin"
_PASSWORD = "admin"
_HOST = "http://localhost:8080/"  # Как отправлять API-запросы к Airflow на Astro: https://www.astronomer.io/docs/astro/airflow-api/

_DAG_ID = "my_dag"
_TASK_ID = "my_task"

def _get_jwt_token():
    token_url = f"{_HOST}/auth/token"
    payload = {"username": _USERNAME, "password": _PASSWORD}
    headers = {"Content-Type": "application/json"}
    response = requests.post(token_url, json=payload, headers=headers)

    token = response.json().get("access_token")
    return token

def _trigger_dag_run(dag_id: str):
    url = f"{_HOST}/api/v2/dags/{dag_id}/dagRuns"
    headers = {
        "Authorization": f"Bearer {_get_jwt_token()}",
        "Content-Type": "application/json",
    }
    payload = {
        "logical_date": None,
    }
    response = requests.post(url, headers=headers, json=payload)
    return response.json()["dag_run_id"]

def _wait_for_dag_run_completion(dag_id: str, dag_run_id: str):
    url = f"{_HOST}/api/v2/dags/{dag_id}/dagRuns/{dag_run_id}/wait"
    headers = {
        "Authorization": f"Bearer {_get_jwt_token()}",
    }
    params = {
        "interval": 1,
        "result": [_TASK_ID],
    }
    response = requests.get(url, headers=headers, params=params)
    print(f"Status Code: {response.status_code}")

    lines = response.text.strip().split("\n")
    json_objects = []

    for line in lines:
        if line.strip():
            json_obj = json.loads(line)
            json_objects.append(json_obj)
            print(f"Status: {json_obj.get('state', 'unknown')}")

    if json_objects:
        last_status_update = json_objects[-1]
        xcom_results = last_status_update.get("results", {})
        print("Last status update: ", last_status_update)
        print("XCom results: ", xcom_results)
        return xcom_results

if __name__ == "__main__":
    _dag_run_id = _trigger_dag_run(_DAG_ID)
    _wait_for_dag_run_completion(_DAG_ID, _dag_run_id)
```

При запуске скрипт выдаёт вывод примерно такого вида:

```text
Status Code: 200
Status: queued
Status: running
Status: running
Status: success
Last status update:  {'state': 'success', 'results': {'my_task': 'Hello World!'}}
XCom results:  {'my_task': 'Hello World!'}
```

---

[← Общий код](sharing-code.md) | [К содержанию](README.md) | [Тестирование →](testing-airflow.md)
