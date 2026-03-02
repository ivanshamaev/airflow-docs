# ty

Сверхбыстрый Python-типчекер и language server, написанный на Rust.

Проверка типов проекта [home-assistant](https://github.com/home-assistant/core) без кэширования.

ty разработан [Astral](https://astral.sh/), создателями [uv](https://github.com/astral-sh/uv) и [Ruff](https://github.com/astral-sh/ruff).

## Основные возможности

- Продвинутые возможности типизации: полноценные [пересечения типов (intersection types)](https://docs.astral.sh/ty/features/type-system/#intersection-types), расширенное [сужение типов (type narrowing)](https://docs.astral.sh/ty/features/type-system/#top-and-bottom-materializations) и [анализ достижимости](https://docs.astral.sh/ty/features/type-system/#reachability-based-on-types)
- Интеграция с редакторами: [VS Code](https://docs.astral.sh/ty/editors/#vs-code), [PyCharm](https://docs.astral.sh/ty/editors/#pycharm), [Neovim](https://docs.astral.sh/ty/editors/#neovim) и другие
- [Пошаговый инкрементальный анализ](https://docs.astral.sh/ty/features/language-server/#fine-grained-incrementality) для быстрых обновлений при редактировании файлов в IDE
- [Language server](https://docs.astral.sh/ty/features/language-server/) с навигацией по коду, автодополнением, code actions, auto-import, inlay hints, подсказками при наведении и т. п.
- Ориентирован на внедрение: поддержка [повторных объявлений (redeclarations)](https://docs.astral.sh/ty/features/type-system/#redeclarations) и [частично типизированного кода](https://docs.astral.sh/ty/features/type-system/#gradual-guarantee)
- Настраиваемые [уровни правил](https://docs.astral.sh/ty/rules/), [переопределения по файлам](https://docs.astral.sh/ty/reference/configuration/#overrides), [комментарии подавления](https://docs.astral.sh/ty/suppression/) и полноценная поддержка проектов
- Подробные [диагностики](https://docs.astral.sh/ty/features/diagnostics/) с богатым контекстом
- В 10–100 раз быстрее mypy и Pyright

## Быстрый старт

Запустите ty через [uvx](https://docs.astral.sh/uv/guides/tools/#running-tools):

```
uvx ty check

```

По умолчанию ty проверяет все Python-файлы в рабочем каталоге или проекте.

Подробнее см. в документации по [проверке типов](https://docs.astral.sh/ty/type-checking/).

## Установка

Инструкции по установке ty см. в [документации по установке](https://docs.astral.sh/ty/installation/).

Чтобы добавить language server ty в редактор, см. руководство по [интеграции с редакторами](https://docs.astral.sh/ty/editors/).

## Playground

У ty есть [онлайн playground](https://play.ty.dev/), где можно попробовать его на фрагментах кода или небольших проектах.

Tip

Playground удобен для обмена фрагментами с другими, например при отправке отчёта об ошибке.
