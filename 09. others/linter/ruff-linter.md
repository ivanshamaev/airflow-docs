# Линтер Ruff (The Ruff Linter)

Линтер Ruff — очень быстрый линтер для Python, задуманный как замена [Flake8](https://pypi.org/project/flake8/) (плюс десятки плагинов), [isort](https://pypi.org/project/isort/), [pydocstyle](https://pypi.org/project/pydocstyle/), [pyupgrade](https://pypi.org/project/pyupgrade/), [autoflake](https://pypi.org/project/autoflake/) и других.

## ruff check

`ruff check` — основная команда для запуска линтера Ruff. Она принимает список файлов или каталогов и проверяет все найденные Python-файлы, при необходимости применяя исправления. При проверке каталога Ruff рекурсивно ищет Python-файлы в нём и во вложенных каталогах:

```bash
$ ruff check                  # Проверить файлы в текущем каталоге.
$ ruff check --fix            # Проверить и автоматически исправить то, что можно.
$ ruff check --watch          # Проверить и перезапускать проверку при изменениях.
$ ruff check path/to/code/    # Проверить файлы в path/to/code/.
```

Полный список опций: `ruff check --help`.

## Выбор правил (Rule selection)

Набор включённых правил задаётся настройками [lint.select](https://docs.astral.sh/ruff/settings/#lint_select), [lint.extend-select](https://docs.astral.sh/ruff/settings/#lint_extend-select) и [lint.ignore](https://docs.astral.sh/ruff/settings/#lint_ignore).

Линтер Ruff повторяет систему кодов правил Flake8: код состоит из префикса в 1–3 буквы и трёх цифр (например `F401`). Префикс указывает «источник» правила (например `F` — Pyflakes, `E` — pycodestyle, `ANN` — flake8-annotations).

Селекторы вроде [lint.select](https://docs.astral.sh/ruff/settings/#lint_select) и [lint.ignore](https://docs.astral.sh/ruff/settings/#lint_ignore) принимают полный код правила (например `F401`) или префикс (например `F`). Пример конфигурации:

**pyproject.toml**:

```toml
[tool.ruff.lint]
select = ["E", "F"]
ignore = ["F401"]
```

**ruff.toml**:

```toml
[lint]
select = ["E", "F"]
ignore = ["F401"]
```

В результате Ruff включит все правила с префиксами `E` (pycodestyle) и `F` (Pyflakes), кроме `F401`. Подробнее о настройке в `pyproject.toml`: [Configuring Ruff](https://docs.astral.sh/ruff/configuration/).

Особый случай — код `ALL`: он включает все правила. Некоторые правила pydocstyle конфликтуют (например `D203` и `D211` — разные форматы docstring). При включении `ALL` Ruff автоматически отключает конфликтующие правила.

Рекомендации по настройке:

- Начните с небольшого набора правил (`select = ["E", "F"]`) и добавляйте категории по одной. Например, расширьте до `select = ["E", "F", "B"]` для flake8-bugbear.
- Используйте `ALL` осторожно: при обновлении Ruff будут автоматически включаться новые правила.
- Лучше задавать явный набор через [lint.select](https://docs.astral.sh/ruff/settings/#lint_select), чем [lint.extend-select](https://docs.astral.sh/ruff/settings/#lint_extend-select).
- [Contributing](https://docs.astral.sh/ruff/contributing/)

Пример конфигурации с популярными правилами (без излишней строгости):

**pyproject.toml** / **ruff.toml**:

```toml
[tool.ruff.lint]
select = [
    "E",    # pycodestyle
    "F",    # Pyflakes
    "UP",   # pyupgrade
    "B",    # flake8-bugbear
    "SIM",  # flake8-simplify
    "I",    # isort
]
```

Итоговый набор правил может собираться из нескольких источников: текущий `pyproject.toml`, наследуемые `pyproject.toml`, опции CLI (например [--select](https://docs.astral.sh/ruff/settings/#lint_select)). Ruff берёт [select](https://docs.astral.sh/ruff/settings/#lint_select) с наивысшим приоритетом как основу, затем применяет [extend-select](https://docs.astral.sh/ruff/settings/#lint_extend-select) и [ignore](https://docs.astral.sh/ruff/settings/#lint_ignore). Опции CLI имеют приоритет над `pyproject.toml`, текущий `pyproject.toml` — над наследуемыми.

Пример: при конфигурации `select = ["E", "F"]`, `ignore = ["F401"]`:

- `ruff check --select F401` — будет проверяться только правило `F401`.
- `ruff check --extend-select B` — будут проверяться правила `E`, `F` и `B`, кроме `F401`.

## Исправления (Fixes)

Ruff умеет автоматически исправлять многие нарушения: удалять неиспользуемые импорты, переформатировать docstring, переписывать аннотации типов под новый синтаксис Python и т.д.

Чтобы включить исправления, передайте флаг `--fix` в `ruff check`:

```bash
$ ruff check --fix
```

По умолчанию Ruff применяет все «безопасные» исправления. Поддержка исправлений по правилам описана в [Rules](https://docs.astral.sh/ruff/rules/).

### Безопасность исправлений (Fix safety)

Исправления делятся на «безопасные» (safe) и «небезопасные» (unsafe). Безопасные сохраняют смысл кода; небезопасные могут его изменить.

Небезопасное исправление может изменить поведение в runtime, удалить комментарии или и то и другое. Безопасные призваны сохранять поведение и удаляют комментарии только при удалении целых операторов/выражений (например неиспользуемого импорта).

Пример: правило [unnecessary-iterable-allocation-for-first-element](https://docs.astral.sh/ruff/rules/unnecessary-iterable-allocation-for-first-element/) (`RUF015`) ищет неэффективное использование `list(...)[0]`. Исправление заменяет его на `next(iter(...))`, что может сильно ускорить код (в примере в документации — с ~1.69 с до ~70.8 нс на операцию). Но для пустой коллекции тип исключения меняется с `IndexError` на `StopIteration`, что может сломать обработку ошибок выше по коду, поэтому такое исправление считается небезопасным.

По умолчанию Ruff применяет только безопасные исправления. Небезопасные включаются настройкой [unsafe-fixes](https://docs.astral.sh/ruff/settings/#unsafe-fixes) в конфиге или флагом `--unsafe-fixes`:

```bash
# Показать небезопасные исправления
ruff check --unsafe-fixes

# Применить небезопасные исправления
ruff check --fix --unsafe-fixes
```

По умолчанию Ruff подсказывает, когда доступны небезопасные исправления. Подсказку можно отключить, задав [unsafe-fixes](https://docs.astral.sh/ruff/settings/#unsafe-fixes) в `false` или флагом `--no-unsafe-fixes`.

Безопасность исправлений можно менять по правилам через [lint.extend-safe-fixes](https://docs.astral.sh/ruff/settings/#lint_extend-safe-fixes) и [lint.extend-unsafe-fixes](https://docs.astral.sh/ruff/settings/#lint_extend-unsafe-fixes). Например, сделать исправления для `F601` безопасными, а для `UP034` — небезопасными:

**pyproject.toml** / **ruff.toml**:

```toml
[tool.ruff.lint]
extend-safe-fixes = ["F601"]
extend-unsafe-fixes = ["UP034"]
```

Можно использовать префиксы (например `F` для всех правил Pyflakes).

> При выводе в формате `json` Ruff всегда показывает все исправления; поле `applicability` указывает безопасность.
>
> Примечание

### Отключение исправлений (Disabling fixes)

Ограничить набор правил, для которых Ruff может применять исправления, можно настройками [lint.fixable](https://docs.astral.sh/ruff/settings/#lint_fixable) / [lint.extend-fixable](https://docs.astral.sh/ruff/settings/#lint_extend-fixable) и [lint.unfixable](https://docs.astral.sh/ruff/settings/#lint_unfixable).

Пример: включить исправления для всех правил, кроме [unused-imports](https://docs.astral.sh/ruff/rules/unused-import/) (`F401`):

```toml
[tool.ruff.lint]
fixable = ["ALL"]
unfixable = ["F401"]
```

Обратный пример — только исправления для `F401`:

```toml
[tool.ruff.lint]
fixable = ["F401"]
```

## Подавление ошибок (Error suppression)

Ruff поддерживает несколько способов подавления срабатываний: ложные срабатывания или допустимые нарушения.

### Конфигурация

Чтобы отключить правило везде, добавьте его в список `ignore` через [lint.ignore](https://docs.astral.sh/ruff/settings/#lint_ignore) в CLI, `pyproject.toml` или `ruff.toml`. Чтобы отключить по путям/шаблонам файлов — [lint.per-file-ignores](https://docs.astral.sh/ruff/settings/#lint_per-file-ignores).

Ruff поддерживает комментарии подавления: построчные и файловые `noqa`, а также подавление по диапазону.

#### Построчное (Line-level)

Поддерживается система `noqa`, похожая на [Flake8](https://flake8.pycqa.org/en/3.1.1/user/ignoring-errors.html). Чтобы игнорировать одно срабатывание, в конец строки добавьте `# noqa: {code}`:

```python
# Игнорировать F841.
x = 1  # noqa: F841

# Игнорировать E741 и F841.
i = 1  # noqa: E741, F841

# Игнорировать все срабатывания на строке.
x = 1  # noqa
```

Для многострочных строк (например docstring) директива `noqa` ставится в конце строки (после закрывающих тройных кавычек) и относится ко всей строке:

```python
"""Lorem ipsum dolor sit amet.
...
"""  # noqa: E501
```

Для сортировки импортов `noqa` ставится в конце первой строки блока импортов и действует на весь блок:

```python
import os  # noqa: I001
import abc
```

Спецификация inline-комментария: после совпадения (без учёта регистра) `#noqa` с опциональными пробелами после `#` и после `noqa` может идти `:`, затем список кодов правил (буквы и цифры, через пробелы или запятые). «Одеяльное» подавление — `#noqa` без `:` и кодов.

#### Блочное (Block-level)

Чтобы игнорировать срабатывания в диапазоне кода, используется комментарий «disable», затем парный «enable»:

```python
# ruff: disable[E501]
VALUE_1 = "Lorem ipsum ..."
VALUE_2 = "Lorem ipsum ..."
# ruff: enable[E501]
```

Коды в «disable» и «enable» должны совпадать и идти в одном порядке; уровень отступа должен быть в пределах одного логического блока. Если парный «enable» не найден, Ruff считает диапазон «неявным» — от «disable» до выхода из области с меньшим отступом. Рекомендуется использовать явные диапазоны, чтобы не подавить лишнее; при неявном диапазоне выдаётся диагностика `RUF104`. Подавления по диапазону не позволяют включать правила, не выбранные в конфигурации или CLI. «Enable» только завершает предшествующий «disable» с теми же кодами. В отличие от `noqa`, в диапазоне нельзя подавить «всё» — нужно указать хотя бы один код.

Спецификация: коды через запятые, опциональные пробелы; отдельная строка с точным `#ruff:`, затем `disable` или `enable`, затем `[`, коды, `]`.

#### Файловое (File-level)

Чтобы игнорировать все срабатывания в файле, добавьте строку `# ruff: noqa` (лучше в начало файла). Для одного правила — `# ruff: noqa: {code}`. Файловый `noqa` должен быть на отдельной строке. Ruff также понимает директиву Flake8 `# flake8: noqa` как аналог `# ruff: noqa`.

### Обнаружение неиспользуемых подавлений (Detecting unused suppressions)

Правило [unused-noqa](https://docs.astral.sh/ruff/rules/unused-noqa/) (код `RUF100`) проверяет, что подавления действительно относятся к срабатывающим нарушениям. Чтобы включить проверку: `ruff check --extend-select RUF100`. Удалить неиспользуемые подавления автоматически: `ruff check --extend-select RUF100 --fix`.

### Добавление директив noqa (Inserting necessary suppression comments)

Флаг `--add-noqa` автоматически добавляет директивы `noqa` ко всем строкам с нарушениями (удобно при миграции кодовой базы на Ruff): `ruff check /path/to/file.py --add-noqa`.

### Комментарии действий isort (isort action comments)

Ruff учитывает [action comments](https://pycqa.github.io/isort/docs/configuration/action_comments.html) isort (`# isort: skip_file`, `# isort: on`, `# isort: off`, `# isort: skip`, `# isort: split`) для выборочного включения/отключения сортировки импортов. Поддерживаются варианты с префиксом `# ruff:` (например `# ruff: isort: skip_file`). В отличие от isort, Ruff не учитывает такие комментарии внутри docstring. Подробнее: [документация isort](https://pycqa.github.io/isort/docs/configuration/action_comments.html).

## Коды выхода (Exit codes)

По умолчанию `ruff check` завершается с кодами:

- **2** — аварийное завершение (неверная конфигурация, неверные опции CLI или внутренняя ошибка).
- **1** — найдены нарушения.
- **0** — нарушений нет или все нарушения были автоматически исправлены.

Аналогично ESLint, Prettier, RuboCop.

Два флага меняют поведение кода выхода:

- **`--exit-non-zero-on-fix`** — код выхода `1`, если нарушения были найдены, даже если все они исправлены автоматически (в итоге может быть не ноль при отсутствии оставшихся нарушений).
- **`--exit-zero`** — код выхода `0` даже при наличии нарушений. Код `2` при аварийном завершении сохраняется.
