# uv

Очень быстрый менеджер пакетов и проектов для Python, написанный на Rust.

*Установка зависимостей [Trio](https://trio.readthedocs.io/) с тёплым кэшем.*

## Основное

- Поддержка macOS, Linux и Windows.
- Установка без Rust и Python через `curl` или `pip`.
- Эффективное использование диска за счёт [глобального кэша](https://docs.astral.sh/uv/concepts/cache/) для дедупликации зависимостей.
- Поддержка [workspaces](https://docs.astral.sh/uv/concepts/projects/workspaces/) в стиле Cargo для масштабируемых проектов.
- Совместимый с pip интерфейс для ускорения работы с привычным CLI.
- Запуск и установка инструментов, распространяемых как Python-пакеты.
- Установка и управление версиями Python.
- Запуск скриптов с поддержкой [метаданных зависимостей в самом скрипте](https://docs.astral.sh/uv/guides/scripts/#declaring-script-dependencies).
- Управление проектами с [универсальным lockfile](https://docs.astral.sh/uv/concepts/projects/layout/#the-lockfile).
- [В 10–100 раз быстрее](https://github.com/astral-sh/uv/blob/main/BENCHMARKS.md), чем `pip`.
- Один инструмент вместо `pip`, `pip-tools`, `pipx`, `poetry`, `pyenv`, `twine`, `virtualenv` и других.

uv разработан [Astral](https://astral.sh/), создателями [Ruff](https://github.com/astral-sh/ruff).

## Установка

Установите uv с помощью официального автономного установщика:

**macOS и Linux:**

```bash
$ curl -LsSf https://astral.sh/uv/install.sh | sh
```

**Windows:**

```powershell
PS> powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

Затем перейдите к [первым шагам](https://docs.astral.sh/uv/getting-started/first-steps/) или читайте дальше для краткого обзора.

**Совет:** uv также можно установить через pip, Homebrew и другими способами. Все варианты описаны на [странице установки](https://docs.astral.sh/uv/getting-started/installation/).

## Проекты

uv управляет зависимостями и окружениями проектов с поддержкой lockfile, workspaces и др., по аналогии с `rye` или `poetry`:

```bash
$ uv init example
Initialized project `example` at `/home/user/example`

$ cd example

$ uv add ruff
Creating virtual environment at: .venv
Resolved 2 packages in 170ms
   Built example @ file:///home/user/example
Prepared 2 packages in 627ms
Installed 2 packages in 1ms
 + example==0.1.0 (from file:///home/user/example)
 + ruff==0.5.4

$ uv run ruff check
All checks passed!

$ uv lock
Resolved 2 packages in 0.33ms

$ uv sync
Resolved 2 packages in 0.70ms
Audited 1 package in 0.02ms
```

Подробнее в [руководстве по проектам](https://docs.astral.sh/uv/guides/projects/).

uv также умеет собирать и публиковать проекты, в том числе не управляемые uv. Подробнее в [руководстве по упаковке](https://docs.astral.sh/uv/guides/package/).

## Скрипты

uv управляет зависимостями и окружениями для однопоточных скриптов.

Создайте скрипт и добавьте в него метаданные с зависимостями:

```bash
$ echo 'import requests; print(requests.get("https://astral.sh"))' > example.py

$ uv add --script example.py requests
Updated `example.py`
```

Запуск скрипта в изолированном виртуальном окружении:

```bash
$ uv run example.py
Reading inline script metadata from: example.py
Installed 5 packages in 12ms
<Response [200]>
```

Подробнее в [руководстве по скриптам](https://docs.astral.sh/uv/guides/scripts/).

## Инструменты

uv запускает и устанавливает консольные инструменты из Python-пакетов, по аналогии с `pipx`.

Запуск инструмента во временном окружении через `uvx` (псевдоним для `uv tool run`):

```bash
$ uvx pycowsay 'hello world!'
Resolved 1 package in 167ms
Installed 1 package in 9ms
 + pycowsay==0.0.0.2
  """
  ------------
< hello world! >
  ------------
   \   ^__^
    \  (oo)\_______
       (__)\       )\/\
           ||----w |
           ||     ||
```

Установка инструмента через `uv tool install`:

```bash
$ uv tool install ruff
Resolved 1 package in 6ms
Installed 1 package in 2ms
 + ruff==0.5.4
Installed 1 executable: ruff

$ ruff --version
ruff 0.5.4
```

Подробнее в [руководстве по инструментам](https://docs.astral.sh/uv/guides/tools/).

## Версии Python

uv устанавливает Python и позволяет быстро переключаться между версиями.

Установка нескольких версий Python:

```bash
$ uv python install 3.10 3.11 3.12
Searching for Python versions matching: Python 3.10
Searching for Python versions matching: Python 3.11
Searching for Python versions matching: Python 3.12
Installed 3 versions in 3.42s
 + cpython-3.10.14-macos-aarch64-none
 + cpython-3.11.9-macos-aarch64-none
 + cpython-3.12.4-macos-aarch64-none
```

Загрузка нужных версий Python по мере необходимости:

```bash
$ uv venv --python 3.12.0
Using CPython 3.12.0
Creating virtual environment at: .venv
Activate with: source .venv/bin/activate

$ uv run --python pypy@3.8 -- python
Python 3.8.16 (a9dbdca6fc3286b0addd2240f11d97d8e8de187a, Dec 29 2022, 11:45:30)
[PyPy 7.3.11 with GCC Apple LLVM 13.1.6 (clang-1316.0.21.2.5)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
>>>
```

Фиксация версии Python для текущего каталога:

```bash
$ uv python pin 3.11
Pinned `.python-version` to `3.11`
```

Подробнее в [руководстве по установке Python](https://docs.astral.sh/uv/guides/install-python/).

## Интерфейс pip

uv предоставляет замену распространённым командам `pip`, `pip-tools` и `virtualenv`.

Интерфейс расширен возможностями вроде переопределения версий зависимостей, платформо-независимой резолвинга, воспроизводимой резолвинга, альтернативных стратегий разрешения зависимостей и др.

Переход на uv без смены привычного workflow — и ускорение в 10–100 раз — через интерфейс `uv pip`.

Компиляция requirements в платформо-независимый файл:

```bash
$ uv pip compile docs/requirements.in \
   --universal \
   --output-file docs/requirements.txt
Resolved 43 packages in 12ms
```

Создание виртуального окружения:

```bash
$ uv venv
Using CPython 3.12.3
Creating virtual environment at: .venv
Activate with: source .venv/bin/activate
```

Установка зафиксированных зависимостей:

```bash
$ uv pip sync docs/requirements.txt
Resolved 43 packages in 11ms
Installed 43 packages in 208ms
 + babel==2.15.0
 + black==24.4.2
 + certifi==2024.7.4
 ...
```

Подробнее в [документации по интерфейсу pip](https://docs.astral.sh/uv/pip/).

## Дополнительно

См. [первые шаги](https://docs.astral.sh/uv/getting-started/first-steps/) или перейдите к [руководствам](https://docs.astral.sh/uv/guides/), чтобы начать пользоваться uv.
