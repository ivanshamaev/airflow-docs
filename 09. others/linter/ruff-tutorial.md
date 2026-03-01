# Туториал (Tutorial)

В этом туториале показано, как подключить линтер и форматтер Ruff к проекту. Подробнее о настройке: [Configuring Ruff](https://docs.astral.sh/ruff/configuration/).

## Начало работы (Getting Started)

Создаём проект с помощью [uv](https://docs.astral.sh/uv/):

```bash
$ uv init --lib numbers
```

Команда создаёт Python-проект со структурой:

```text
numbers
  ├── README.md
  ├── pyproject.toml
  └── src
      └── numbers
          ├── __init__.py
          └── py.typed
```

Очистим автогенерируемое содержимое в `src/numbers/__init__.py` и создадим `src/numbers/calculate.py` с таким кодом:

```python
from typing import Iterable

import os


def sum_even_numbers(numbers: Iterable[int]) -> int:
    """Given an iterable of integers, return the sum of all even numbers in the iterable."""
    return sum(
        num for num in numbers
        if num % 2 == 0
    )
```

Добавляем Ruff в проект:

```bash
$ uv add --dev ruff
```

Запуск линтера Ruff по проекту: `uv run ruff check`:

```bash
$ uv run ruff check
src/numbers/calculate.py:3:8: F401 [*] `os` imported but unused
Found 1 error.
[*] 1 fixable with the `--fix` option.
```

> Вместо `uv run` можно активировать виртуальное окружение проекта (`source .venv/bin/activate` на Linux и macOS или `.venv\Scripts\activate` на Windows) и запускать `ruff check` напрямую.
>
> Примечание

Ruff нашёл неиспользуемый импорт — типичную проблему в Python. Ошибка считается «исправляемой», поэтому её можно устранить автоматически: `ruff check --fix`:

```bash
$ uv run ruff check --fix
Found 1 error (1 fixed, 0 remaining).
```

Вывод `git diff`:

```diff
--- a/src/numbers/calculate.py
+++ b/src/numbers/calculate.py
@@ -1,7 +1,5 @@
 from typing import Iterable

-import os
-

 def sum_even_numbers(numbers: Iterable[int]) -> int:
```

> По умолчанию Ruff проверяет текущий каталог, но можно указать пути:
>
> Примечание

```bash
$ uv run ruff check src/numbers/calculate.py
```

Когда проект проходит `ruff check`, можно запустить форматтер: `ruff format`:

```bash
$ uv run ruff format
1 file reformatted
```

По `git diff` видно, что вызов `sum` переформатирован под лимит длины строки по умолчанию (88 символов):

```diff
--- a/src/numbers/calculate.py
+++ b/src/numbers/calculate.py
@@ -3,7 +3,4 @@ from typing import Iterable

 def sum_even_numbers(numbers: Iterable[int]) -> int:
     """Given an iterable of integers, return the sum of all even numbers in the iterable."""
-    return sum(
-        num for num in numbers
-        if num % 2 == 0
-    )
+    return sum(num for num in numbers if num % 2 == 0)
```

До этого использовалась конфигурация Ruff по умолчанию. Ниже — как её изменить.

## Конфигурация (Configuration)

Чтобы выбрать настройки для каждого файла, Ruff ищет первый файл `pyproject.toml`, `ruff.toml` или `.ruff.toml` в каталоге файла или в любом родительском каталоге.

Добавим в конфигурационный файл в корне проекта:

**pyproject.toml** / **ruff.toml**:

```toml
[tool.ruff]
# Максимальная длина строки — 79.
line-length = 79

[tool.ruff.lint]
# Включить правило `line-too-long`. По умолчанию Ruff не включает правила,
# пересекающиеся с форматтером (например Black), но это можно переопределить.
extend-select = ["E501"]
```

Вариант для `ruff.toml`:

```toml
line-length = 79

[lint]
extend-select = ["E501"]
```

После повторного запуска Ruff ограничивает длину строки (лимит 79):

```bash
$ uv run ruff check
src/numbers/calculate.py:5:80: E501 Line too long (90 > 79)
Found 1 error.
```

Полный список настроек: [Settings](https://docs.astral.sh/ruff/settings/). Для нашего проекта укажем минимальную версию Python:

**pyproject.toml**:

```toml
[project]
requires-python = ">=3.10"

[tool.ruff]
line-length = 79

[tool.ruff.lint]
extend-select = ["E501"]
```

**ruff.toml**:

```toml
target-version = "py310"
line-length = 79

[lint]
extend-select = ["E501"]
```

### Выбор правил (Rule Selection)

В Ruff [более 800 правил](https://docs.astral.sh/ruff/rules/) в более чем 50 встроенных плагинах. Набор правил зависит от проекта: часть может быть слишком строгой, часть заточена под конкретные фреймворки.

По умолчанию включены правила `F` из Flake8 и подмножество правил `E`; стилистические правила, пересекающиеся с форматтером (`ruff format` или [Black](https://github.com/psf/black)), по умолчанию отключены.

При первом подключении линтера разумно начать с набора по умолчанию: он узкий, но ловит много типичных ошибок (например неиспользуемые импорты) без настройки.

При переходе с другого линтера можно включить эквивалентные правила. Например, чтобы включить правила pyupgrade:

**pyproject.toml**:

```toml
[project]
requires-python = ">=3.10"

[tool.ruff.lint]
extend-select = ["UP"]  # pyupgrade
```

**ruff.toml**:

```toml
target-version = "py310"

[lint]
extend-select = ["UP"]  # pyupgrade
```

После запуска Ruff начнёт применять правила pyupgrade. В частности, помечает устаревший `typing.Iterable` вместо `collections.abc.Iterable`:

```bash
$ uv run ruff check
src/numbers/calculate.py:1:1: UP035 [*] Import from `collections.abc` instead: `Iterable`
Found 1 error.
[*] 1 fixable with the `--fix` option.
```

Со временем можно добавить правила. Например, требовать docstring у всех функций:

**pyproject.toml**:

```toml
[project]
requires-python = ">=3.10"

[tool.ruff.lint]
extend-select = ["UP", "D"]  # pyupgrade, pydocstyle

[tool.ruff.lint.pydocstyle]
convention = "google"
```

**ruff.toml**:

```toml
target-version = "py310"

[lint]
extend-select = ["UP", "D"]

[lint.pydocstyle]
convention = "google"
```

После запуска Ruff начнёт проверять по pydocstyle:

```bash
$ uv run ruff check
src/numbers/__init__.py:1:1: D104 Missing docstring in public package
src/numbers/calculate.py:1:1: UP035 [*] Import from `collections.abc` instead: `Iterable`
...
src/numbers/calculate.py:1:1: D100 Missing docstring in public module
Found 3 errors.
[*] 1 fixable with the `--fix` option.
```

### Игнорирование ошибок (Ignoring Errors)

Любое правило можно отключить для строки комментарием `# noqa`. Например, отключим правило UP035 для импорта `Iterable`:

```python
from typing import Iterable  # noqa: UP035


def sum_even_numbers(numbers: Iterable[int]) -> int:
    """Given an iterable of integers, return the sum of all even numbers in the iterable."""
    return sum(num for num in numbers if num % 2 == 0)
```

После `ruff check` импорт `Iterable` больше не помечается:

```bash
$ uv run ruff check
src/numbers/__init__.py:1:1: D104 Missing docstring in public package
src/numbers/calculate.py:1:1: D100 Missing docstring in public module
Found 2 errors.
```

Чтобы отключить правило для всего файла, добавьте в файл (лучше в начало) строку `# ruff: noqa: {code}`:

```python
# ruff: noqa: UP035
from typing import Iterable


def sum_even_numbers(numbers: Iterable[int]) -> int:
    """Given an iterable of integers, return the sum of all even numbers in the iterable."""
    return sum(num for num in numbers if num % 2 == 0)
```

Подробнее: [Error suppression](https://docs.astral.sh/ruff/linter/#error-suppression).

### Добавление правил (Adding Rules)

При включении нового правила в существующей кодовой базе можно не исправлять старые нарушения, а только следить за новыми.

Флаг `--add-noqa` добавляет директиву `# noqa` к каждой строке с текущими нарушениями. Вместе с `--select` можно добавить `# noqa` ко всем нарушениям UP035:

```bash
$ uv run ruff check --select UP035 --add-noqa .
Added 1 noqa directive.
```

Пример `git diff`:

```diff
diff --git a/numbers/src/numbers/calculate.py b/numbers/src/numbers/calculate.py
--- a/numbers/src/numbers/calculate.py
+++ b/numbers/src/numbers/calculate.py
@@ -1,4 +1,4 @@
-from typing import Iterable
+from typing import Iterable  # noqa: UP035
```

## Интеграции (Integrations)

В туториале использовался CLI Ruff, но Ruff можно подключить как хук [pre-commit](https://pre-commit.com/) через [ruff-pre-commit](https://github.com/astral-sh/ruff-pre-commit):

```yaml
- repo: https://github.com/astral-sh/ruff-pre-commit
  rev: v0.15.4  # версия Ruff
  hooks:
    - id: ruff-check   # линтер
    - id: ruff-format  # форматтер
```

Ruff также интегрируется с редакторами. Подробнее: [Editors](https://docs.astral.sh/ruff/editors/).

Другие варианты: [Integrations](https://docs.astral.sh/ruff/integrations/).
