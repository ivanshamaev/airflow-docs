# airflow-docs

Русский перевод документации **Apache Airflow®** в формате Markdown. Структура ориентирована на разделы сайта [Astronomer.io Docs](https://www.astronomer.io/docs/learn/). Для личного использования и размещения на GitHub.

---

Подписывайтесь на телеграм канал t.me/data_engineer_path

---
## Структура документов

### Apache Airflow (официальная документация)

Краткий перевод основных разделов с [airflow.apache.org](https://airflow.apache.org/docs/):

- **[apache/airflow/](apache/airflow/)** — оглавление
  - [Что такое Airflow?](apache/airflow/index.md)
  - [Основные концепции](apache/airflow/core-concepts/README.md) — DAG, задачи (Tasks)
  - [Быстрый старт](apache/airflow/quick-start.md)

---

### 01. Airflow Concepts: Basics (Astronomer)

Основы Airflow: интерфейс, операторы, планирование, переменные, подключения.

- **[01. astronomer-basic/](01.%20astronomer-basic/)** — оглавление раздела
- Темы: Airflow UI, ассеты, BashOperator, Connections, DAG, выполнение SQL, Hooks, управление кодом, операторы, планирование, сенсоры, зависимости задач, trigger rules, переменные.

---

### 02. Airflow DAGs (Astronomer)

Написание и отладка DAG: контекст, декораторы, параметры, ветвление, XCom, тесты.

- **[02. astronomer-dags/](02.%20astronomer-dags/)** — оглавление раздела
- Темы: контекст Airflow, декораторы, уведомления, params, BranchOperator, cross-DAG зависимости, кастомные хуки и операторы, лучшие практики, параметры DAG, версионирование, отладка, динамические задачи, Jinja-шаблоны, передача данных между задачами, повторный запуск, task groups.

---

### 03. Airflow: инфраструктура (Astronomer)

Компоненты Airflow, метаданные БД, исполнители, масштабирование.

- **[03. astronomer-infra/](03.%20astronomer-infra/)** — оглавление раздела
- Темы: компоненты (Scheduler, API server, DAG processor, Triggerer, БД), метаданные БД, executors (Astro, Kubernetes, Celery, Local), масштабирование воркеров.

---

### 04. Airflow: продвинутые темы (Astronomer)

Политики кластера, MLOps, плагины, пуллы, XCom backend, deferrable, event-driven, Human-in-the-loop, изолированные окружения, KubernetesPodOperator, логирование, мультиязычность, динамические DAG, setup/teardown, общий код, синхронное выполнение, тестирование.

- **[04. astronomer-advanced/](04.%20astronomer-advanced/)** — оглавление раздела

---

### 05. Airflow: написание DAG (Astronomer)

Практики написания DAG и локальная разработка.

- **[05. astronomer-write-dags/](05.%20astronomer-write-dags/)** — оглавление раздела
- Темы: Airflow Object Storage, документирование DAG, DAG Factory (YAML), разработка в PyCharm, SQL check operators (data quality), разработка в VS Code.

---

## Использование

- Читать в любом Markdown-редакторе или на GitHub (навигация по ссылкам между файлами).
- Исходные тексты: [Apache Airflow Documentation](https://airflow.apache.org/docs/), [Astronomer Learn](https://www.astronomer.io/docs/learn/). Перевод неофициальный.

## Оригинальная документация

- [Documentation | Apache Airflow](https://airflow.apache.org/docs/)
- [Astronomer.io Docs — Learn](https://www.astronomer.io/docs/learn/)

## Лицензия

Apache Airflow — проект Apache Software Foundation. Репозиторий — неофициальный перевод документации для личного использования.
