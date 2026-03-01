# Airflow Docs (RU)

Неофициальный русский перевод документации **Apache Airflow®** и [Astronomer Learn](https://www.astronomer.io/docs/learn/) в Markdown. Для личного использования и GitHub.

---

## Содержание сайта

| Раздел | Описание |
|--------|----------|
| **[01. Basics](01.%20astronomer-basic/README.md)** | Интерфейс, операторы, DAG, планирование, сенсоры, Connections, Variables, trigger rules |
| **[02. DAGs](02.%20astronomer-dags/README.md)** | Контекст, декораторы, params, BranchOperator, XCom, task groups, Jinja, отладка, best practices |
| **[03. Инфраструктура](03.%20astronomer-infra/README.md)** | Компоненты Airflow, БД, executors (Astro, K8s, Celery), масштабирование |
| **[04. Продвинутое](04.%20astronomer-advanced/README.md)** | Политики кластера, пуллы, deferrable, event-driven, Human-in-the-loop, K8s Pod Operator, setup/teardown |
| **[05. Написание DAG](05.%20astronomer-write-dags/README.md)** | Object Storage, DAG Docs, DAG Factory, SQL checks, разработка в PyCharm/VS Code |

В каждом разделе — оглавление и ссылки на все страницы.

---

## Телеграм

[@data_engineer_path](https://t.me/data_engineer_path)

---

## Сайт (GitHub Pages)

Сборка: **MkDocs** + тема **Material** (боковое меню, поиск). Деплой при пуше в `main` через GitHub Actions.

- **URL:** `https://<владелец>.github.io/airflow-docs/` (включите **Settings → Pages → Source: GitHub Actions**).
- **Локально:** `pip install mkdocs-material && mkdocs serve` → http://127.0.0.1:8000

---

## Использование

Читать в Markdown-редакторе или на GitHub; навигация по ссылкам между файлами. Оригинал: [Airflow Docs](https://airflow.apache.org/docs/), [Astronomer Learn](https://www.astronomer.io/docs/learn/).

**Лицензия:** Apache Airflow — Apache Software Foundation. Репозиторий — неофициальный перевод для личного использования.
