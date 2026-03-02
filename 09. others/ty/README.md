# ty

Раздел с переводом ключевых страниц документации по сверхбыстрому типчекеру и language server’у [ty](https://docs.astral.sh/ty/), созданному командой Astral (авторы uv и Ruff).

## Содержание раздела

- **[Обзор ty](overview.md)** — что такое ty, основные возможности, производительность, playground.
- **[Установка ty](installation.md)** — запуск через `uvx`, установка в проект, глобальная установка (uv, pipx, pip, mise), standalone‑инсталлятор, Docker, Bazel, автодополнение команд.
- **[Проверка типов](type-checking.md)** — команды `ty check`, выбор окружения, выбор файлов, правила и уровни, watch‑режим, ссылка на описание type system.
- **[Интеграция с редакторами](editors.md)** — VS Code, Neovim, Zed, PyCharm, другие редакторы через LSP, базовые настройки.
- **[Конфигурация (общее)](configuration.md)** — файлы `pyproject.toml` и `ty.toml`, приоритет `ty.toml`, пользовательская конфигурация, слияние настроек проекта и пользователя.
- **[Обнаружение модулей](modules.md)** — first‑party / third‑party модули, поиск по `root`, виртуальные окружения, настройка `environment.python`.
- **[Версия Python](python-version.md)** — как `python-version` влияет на синтаксис и типы, `requires-python`, инференс версии по виртуальному окружению, значение по умолчанию и явная настройка.
- **[Исключения файлов](exclusions.md)** — `src.include`/`src.exclude`, паттерны, дефолтные исключения, игнор‑файлы, явные пути к проверке.
- **[Правила](rules.md)** — уровни правил (`ignore` / `warn` / `error`), настройка через CLI и конфигурацию, установка уровня `all`.
- **[Подавление (suppressions)](suppression.md)** — комментарии `# ty: ignore[...]`, `type: ignore`, несколько комментариев, `unused-ignore-comment`, декоратор `@no_type_check`.
- **[Система типов ty](type-system.md)** — redeclarations, пересечения типов (`Intersection`), top/bottom‑materializations, анализ достижимости по типам, gradual guarantee, fixpoint iteration.
- **[Справочник конфигурации](reference-configuration.md)** — детальное описание всех опций `rules`, `analysis`, `environment`, `overrides`, `src`, `terminal` в `pyproject.toml` и `ty.toml`.
- **[CLI справочник](reference-cli.md)** — команды `ty` (`check`, `server`, `version`, `generate-shell-completion`, `help`), аргументы, опции и флаги.

