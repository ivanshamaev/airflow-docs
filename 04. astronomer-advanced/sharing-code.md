# Общий код между проектами (Sharing code)

Три уровня переиспользования кода (от простого к более универсальному):

1. **Один файл** — общая функция в том же DAG-файле, вызов из нескольких задач. Подходит, когда код нужен только в одном скрипте.
2. **Папка `/include`** — вынести функцию в файл (например, `include/db.py`) и импортировать в нескольких DAG в **одном** репозитории. Образ Astro Runtime монтирует `include`, он доступен при выполнении задач.
3. **Отдельный Python-пакет** — код в отдельном Git-репозитории, собран в пакет (pyproject.toml, setuptools), публикуется в внутренний репозиторий (Artifactory, devpi) или ставится из пути. Проекты добавляют пакет в `requirements.txt` и импортируют (например, `from my_company_airflow.db import query_db`). Подходит для нескольких команд и репозиториев.

Рекомендации для пакета: сначала настроить CI/CD (сборка, тесты, публикация), ввести стандарты (lint, format), закрепить ответственных за репозиторий. Пример: [custom-package-demo](https://github.com/astronomer/custom-package-demo).

Подробнее: [Sharing code across projects](https://www.astronomer.io/docs/learn/sharing-code-multiple-projects).

---

[← Setup/teardown](setup-teardown.md) | [К содержанию](README.md) | [Синхронное выполнение →](synchronous-execution.md)
