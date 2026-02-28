# Быстрый старт

Краткие шаги, чтобы поднять Airflow локально и запустить первый DAG.

## 1. Установка

Рекомендуемый способ — виртуальное окружение и ограничение версии поставщиков (constraints):

```bash
python -m venv .venv
source .venv/bin/activate  # Linux/macOS
# или: .venv\Scripts\activate  # Windows

pip install "apache-airflow==3.1.7" --constraint "https://raw.githubusercontent.com/apache/airflow/constraints-3.1.7/constraints-3.10.txt"
```

Подставьте свою версию Python в URL constraints (например, `3.10` → `3.11`). Актуальные ссылки: [Installation of Airflow](https://airflow.apache.org/docs/apache-airflow/stable/installation/index.html).

## 2. Инициализация БД и пользователя

```bash
airflow db init
airflow users create --username admin --firstname Admin --lastname User --role Admin --email admin@example.com --password admin
```

## 3. Запуск

В отдельных терминалах:

```bash
airflow webserver --port 8080
airflow scheduler
```

Интерфейс: http://localhost:8080 (логин/пароль из шага 2).

## 4. Первый DAG

Положите Python-файл с DAG в каталог `dags/` (путь задаётся в `airflow.cfg`, по умолчанию `~/airflow/dags`). Пример минимального DAG см. на [главной странице](index.md). После появления DAG в списке включите его переключателем и запустите вручную или дождитесь расписания.

## Дальше

- [Что такое Airflow?](index.md)
- [DAG и задачи](core-concepts/README.md)
- [Официальная документация: Quick Start](https://airflow.apache.org/docs/apache-airflow/stable/quickstart.html)

---

[← К содержанию](toc.md)
