# Linter (Ruff)

Раздел с переводом документации линтера и форматтера [Ruff](https://docs.astral.sh/ruff/) (Astral).

## Содержание раздела

- **[Установка Ruff](installation.md)** — PyPI, uvx, uv, pip, pipx, автономные установщики, Homebrew, Conda, pkgx, Arch, Alpine, openSUSE, Docker.
- **[Туториал Ruff](ruff-tutorial.md)** — интеграция линтера и форматтера Ruff в проект (uv), конфигурация, выбор правил, игнорирование ошибок, pre-commit и редакторы.
- **[Линтер Ruff](ruff-linter.md)** — ruff check, выбор правил, исправления (safe/unsafe), подавление ошибок (noqa, block, file-level), RUF100, exit codes.
- **[Форматтер Ruff](ruff-formatter.md)** — ruff format, философия (совместимость с Black), конфигурация, форматирование docstring и Markdown, подавление форматирования, конфликтующие правила линтера, коды выхода, руководство по стилю (f-строки, fluent layout), сортировка импортов.
- **[Правила Airflow (AIR)](airflow-rules.md)** — правила линтера Ruff для Apache Airflow: AIR001–AIR002 (стиль DAG/задач), AIR301–AIR303, AIR311–AIR312, AIR321 (миграция на Airflow 3.x и 3.1).
