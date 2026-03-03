# Black 101: финальный гайд для enterprise-проектов

Единый туториал по внедрению и использованию [Black](https://black.readthedocs.io/) в Python-проектах корпоративного уровня: теория, установка, конфигурация, CI/CD, миграция, совместная работа с линтерами и типичные ошибки.

## Содержание

1. [Введение](#1-введение)
2. [Теория: принципы Black](#2-теория-принципы-black)
3. [Установка и окружение](#3-установка-и-окружение)
4. [Первый запуск и базовые команды](#4-первый-запуск-и-базовые-команды)
5. [Конфигурация проекта](#5-конфигурация-проекта)
6. [Игнорирование кода и исключение файлов](#6-игнорирование-кода-и-исключение-файлов)
7. [Интеграция в CI/CD](#7-интеграция-в-cicd)
8. [Интеграция в IDE](#8-интеграция-в-ide)
9. [Миграция репозитория и git blame](#9-миграция-репозитория-и-git-blame)
10. [Совместная работа с другими инструментами](#10-совместная-работа-с-другими-инструментами)
11. [Практика: примеры до/после](#11-практика-примеры-допосле)
12. [Типичные ошибки и решения](#12-типичные-ошибки-и-решения)
13. [Enterprise: структура проекта и автоматизация](#13-enterprise-структура-проекта-и-автоматизация)
14. [FAQ и антипаттерны](#14-faq-и-антипаттерны)
15. [Чек-лист для enterprise](#15-чек-лист-для-enterprise)

## 1. Введение

### Для кого этот гайд

Материал для разработчиков, тимлидов и platform-инженеров, которые хотят внедрить единый стиль кода в Python-проекте без бесконечных споров о форматировании.

### Что такое Black

Black — **opinionated** и **deterministic** форматтер кода для Python. Один и тот же код всегда форматируется одинаково; кастомизации стиля почти нет. Код, отформатированный одной версией Black с одними опциями, при повторном прогоне не изменится.

**Ключевая идея:** вы пишете логику, форматирование делает инструмент.

### Зачем в корпоративном проекте

| Проблема | Как решает Black |
|----------|------------------|
| Разный стиль у команд и контрибьюторов | Единый стиль для всего репозитория |
| Ревью забито правками форматирования | Форматирование вынесено в отдельный шаг/коммит |
| «У меня так отформатировалось в IDE» | Один инструмент и одна конфигурация в `pyproject.toml` |
| CI падает из-за «forgot to run black» | `black --check` в CI гарантирует, что коммиты уже отформатированы |
| Онбординг: «какой стиль у вас?» | Ответ: «Black, конфиг в корне» |

**Почему это важно на enterprise-уровне:** снижение стоимости code review, единый стиль в десятках сервисов, проще онбординг, меньше конфликтов при мерже. Black используется в масштабных проектах (Instagram, Google, Microsoft, Spotify, Mozilla и др.).

### Ограничения

- Black **не** линтер: не ищет баги и не заменяет flake8/ruff/mypy.
- Opinionated: почти нет опций стиля (длина строки — одна из немногих).
- Только Python (и .pyi, Jupyter при установке `black[jupyter]`).
- Конфигурация **только** через TOML (`pyproject.toml`); `.black`, `setup.cfg` для Black не поддерживаются.

---

## 2. Теория: принципы Black

### Ширина строки (Line Length)

Black по умолчанию использует **88 символов** (а не 79 из PEP 8). Это баланс между читаемостью и практичностью; на современных мониторах меньше переносов — меньше визуального шума. В конфиге можно задать другое значение (`line-length`).

### Кавычки (Quotes)

Black предпочитает двойные кавычки `"`. Если в строке уже есть двойные кавычки, использует одиночные, чтобы избежать экранирования:

```python
string = "hello world"
string_with_quotes = 'He said "hello"'
```

### Magic Trailing Comma

Если в конце многострочной структуры (список, вызов функции) стоит запятая, Black **не** упаковывает её в одну строку:

```python
# С trailing comma — Black оставит в нескольких строках
data = [
    "item1",
    "item2",
]

# Без trailing comma — Black может упаковать в одну строку (если влезает)
data = ["item1", "item2"]
```

### Пустые строки

Две пустые строки между определениями функций/классов верхнего уровня; одна пустая строка между методами класса (согласно PEP 8).

### Скобки и операторы

Black переносит длинные выражения, размещая бинарный оператор в конце строки (в соответствии с PEP 8). При переносе не ломает семантику.

### AST Safety

Black работает с AST (Abstract Syntax Tree) Python и **никогда** не меняет семантику кода — только формат. Логических ошибок от форматирования не будет.

### Preview и Unstable

Новые возможности стиля сначала попадают в режим `--preview`, затем в стабильный. Для экспериментов: `black --preview .` или в конфиге `preview = true`. Режим `--unstable` ещё менее стабилен. В production обычно используют стабильный Black без этих флагов.

### Нормализация строк

По умолчанию Black нормализует кавычки и переносы строк. Отключить: `--skip-string-normalization` (или `skip-string-normalization = true` в конфиге).

### Расширения

- `black[jupyter]` — поддержка Jupyter Notebooks (`.ipynb`).
- [blacken-docs](https://github.com/psf/black/tree/main/blacken-docs) — форматирование кода в docstrings и Markdown.

---

## 3. Установка и окружение

### Базовая установка

```bash
pip install black
# или
python -m pip install black

# С поддержкой Jupyter
pip install black[jupyter]

black --version
```

### Рекомендуемый способ для enterprise: dev-зависимость

Все участники и CI используют одну версию Black из зависимостей проекта.

**pip + requirements-dev.txt:**

```text
black>=24.0,<25
# или зафиксировать: black==24.10.0
```

**pyproject.toml (PEP 621):**

```toml
[project]
name = "my-enterprise-service"
requires-python = ">=3.11"

[project.optional-dependencies]
dev = [
    "black>=24.0,<25",
    "pytest>=8.0",
]
```

```bash
pip install -e ".[dev]"
# или
uv pip install -e ".[dev]"
```

**Poetry:**

```toml
[tool.poetry.group.dev.dependencies]
black = {version = "^24.0", optional = false}
```

```bash
poetry install
```

Фиксация версии (pinned) в проекте обязательна, чтобы результат в CI и локально совпадал. В `pyproject.toml` дополнительно задают `required-version` (см. раздел 5).

---

## 4. Первый запуск и базовые команды

### Минимальный пример

Файл `app.py`:

```python
def  add(a,b): return a+b

class   User:
    def __init__(self,name,age): self.name=name; self.age=age
```

```bash
black app.py
```

Результат:

```python
def add(a, b):
    return a + b


class User:
    def __init__(self, name, age):
        self.name = name
        self.age = age
```

### Основные команды

| Задача | Команда |
|--------|--------|
| Форматировать файл/каталог | `black path/to/file.py` или `black src/` |
| Весь проект | `black .` или `black src tests` |
| Проверить без изменений (CI) | `black --check .` |
| Показать diff без записи | `black --diff .` |
| Проверка + diff (отладка CI) | `black --check --diff .` |
| Другая длина строки | `black --line-length 100 src` |
| Версия | `black --version` |

### Форматирование из stdin

```bash
echo "print(  'hello' )" | black -
# print("hello")
```

Для редакторов часто передают `--stdin-filename`. Black по умолчанию рекурсивно обходит каталоги и учитывает `.gitignore` и типичные исключения (`.venv`, `build`, `__pycache__` и т.д.).

---

## 5. Конфигурация проекта

Black читает настройки **только** из TOML (обычно `pyproject.toml` в корне). Файлы `.black`, `setup.cfg` для конфигурации Black не поддерживаются.

### Минимальный пример

```toml
[tool.black]
line-length = 88
target-version = ["py311", "py312"]
include = '\.pyi?$'
extend-exclude = '''
/(
    migrations
  | _generated
  | \.venv
)/
'''
```

### Основные опции

| Опция | По умолчанию | Описание |
|-------|--------------|----------|
| `line-length` | 88 | Максимальная длина строки |
| `target-version` | из `requires-python` | Список версий Python, напр. `["py311"]` |
| `exclude` | см. док. | Регулярное выражение: что исключать (переопределяет дефолты) |
| `extend-exclude` | — | Добавить к стандартным исключениям |
| `force-exclude` | — | Исключать даже при явной передаче пути (удобно для pre-commit) |
| `include` | `\.pyi?$` | Регулярное выражение для включаемых файлов |
| `skip-string-normalization` | false | Не менять кавычки в строках |
| `required-version` | — | Требуемая версия Black, напр. `"24"` или `"24.0"` |
| `preview` | false | Включить стиль из следующего мажорного релиза |

На Windows в путях в конфиге используйте прямые слэши `/`.

### Фиксация версии Black (рекомендуется)

```toml
[tool.black]
line-length = 88
required-version = "24"
```

При несовпадении версии Black выведет ошибку и завершится с ненулевым кодом.

### Полный пример для команды

```toml
[tool.black]
line-length = 88
target-version = ["py311", "py312"]
required-version = "24"
include = '\.pyi?$'
extend-exclude = '''
/(
    \.git
  | \.venv
  | venv
  | build
  | dist
  | migrations
  | __pycache__
  | node_modules
)/
'''
```

---

## 6. Игнорирование кода и исключение файлов

### Участки кода

- **Строка:** комментарий `# fmt: skip` — не форматировать эту строку.
- **Блок:** между `# fmt: off` и `# fmt: on` — не форматировать блок.

Правила: `# fmt: off` и `# fmt: on` должны быть на одном уровне отступа и в одном логическом блоке.

Пример:

```python
# fmt: off
matrix = [
    [1, 2, 3],
    [4, 5, 6],
]
# fmt: on

def solve():
    x = 1  # эту строку Black отформатирует
```

Одна строка:

```python
VERY_LONG_LINE = "leave me alone"  # fmt: skip
```

Использовать редко (например, выравнивание в data-table). Black не поддерживает отключение форматирования для всего файла через специальный комментарий.

### Исключение файлов и каталогов

Через `extend-exclude` / `exclude` в `pyproject.toml` (см. раздел 5) или не передавать такие пути в аргументах. В pre-commit можно задать `exclude` в конфиге хука.

---

## 7. Интеграция в CI/CD

### pre-commit (рекомендуется)

**.pre-commit-config.yaml:**

```yaml
repos:
  - repo: https://github.com/psf/black
    rev: 24.10.0
    hooks:
      - id: black
        language_version: python3
        args: ["--config", "pyproject.toml"]

  - repo: https://github.com/PyCQA/isort
    rev: 5.13.2
    hooks:
      - id: isort
        args: ["--profile", "black", "--line-length", "88"]
```

```bash
pre-commit install
pre-commit run black --all-files   # первый прогон по всему репо
```

Порядок: сначала isort, потом Black.

### GitHub Actions

```yaml
name: Code Style

on: [push, pull_request]

jobs:
  black:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - name: Install dependencies
        run: pip install black
      - name: Run black
        run: black --check .
```

С установкой из проекта (для совпадения с `required-version`):

```yaml
- name: Install deps
  run: pip install -e ".[dev]"
- name: Check formatting
  run: black --check --diff .
```

**Почему в CI только `--check`:** CI не должен молча переписывать код; он должен падать и показывать diff, чтобы разработчик поправил форматирование локально.

### GitLab CI

```yaml
black:
  image: python:3.12
  before_script:
    - pip install black
  script:
    - black --check .
```

### Локальный скрипт

```bash
#!/bin/bash
# scripts/check-black.sh
set -e
black --check .
echo "Black: OK"
```

---

## 8. Интеграция в IDE

### VS Code

1. Установить расширение **Black Formatter** (например, `ms-python.black-formatter` или Black Formatter от Microsoft).
2. В `.vscode/settings.json`:

```json
{
  "[python]": {
    "editor.defaultFormatter": "ms-python.black-formatter",
    "editor.formatOnSave": true
  },
  "black-formatter.args": ["--line-length", "88"]
}
```

Или указать путь к конфигу проекта: `"black-formatter.args": ["--config", "pyproject.toml"]`.

### PyCharm

1. Установить Black: `pip install black`.
2. **Settings → Tools → Black** (или Python Integrated Tools): указать путь к интерпретатору с установленным Black и при необходимости аргументы (например, `--config pyproject.toml`).
3. Включить форматирование при сохранении или использовать Format File вручную.

---

## 9. Миграция репозитория и git blame

### План миграции

1. Добавить Black в dev-зависимости и `pyproject.toml` (опции + `required-version`).
2. Сухая проверка: `black --check --diff .`
3. Запустить `black .` по всему репозиторию.
4. Закоммитить изменения **одним коммитом** (например, «style: apply Black code formatting»).
5. Добавить файл для игнорирования этого коммита в `git blame`.

### Сохранение осмысленного git blame

Создайте в корне `.git-blame-ignore-revs`:

```text
# Migrate code style to Black
5b4ab991dede475d393e9d69ec388fd6bd949699
```

Подставьте реальный хеш коммита. Затем:

```bash
git blame src/main.py --ignore-revs-file .git-blame-ignore-revs
```

Чтобы не указывать файл каждый раз:

```bash
git config blame.ignoreRevsFile .git-blame-ignore-revs
```

GitHub и GitLab поддерживают `.git-blame-ignore-revs` в UI blame.

### Стратегии внедрения

**Big-bang (одним PR):** один большой PR только с форматированием, быстрый merge, сразу pre-commit и CI-check. Подходит для средних проектов и при согласии команды на большой diff.

**Поэтапно (по модулям):** форматировать по пакетам/директориям. Меньше merge-конфликтов при долгоживущих ветках, но переходный период с mixed-style. Удобно в монорепо.

Рекомендация: для большинства проектов — big-bang + быстрый merge + обязательные pre-commit и CI.

---

## 10. Совместная работа с другими инструментами

### Роли в pipeline

- **Black** — форматирование.
- **isort** — порядок импортов (Black не сортирует импорты).
- **Ruff / Flake8 / Pylint** — линтинг.

Рекомендуемый порядок: isort → black → ruff (или flake8).

### isort

В **pyproject.toml** (isort 5+):

```toml
[tool.isort]
profile = "black"
line_length = 88
```

Или в **.isort.cfg**:

```ini
[settings]
profile = black
line_length = 88
```

### Ruff

Ruff может и линтить, и форматировать (в стиле Black). Если форматирование оставляете за Black, в Ruff задайте ту же `line-length` (88) и не дублируйте форматтер на тех же файлах. Пример порядка в pre-commit: isort → black → ruff с `args: ["--fix"]` только для линтинга.

### Flake8 / Pylint

- **Flake8:** `max-line-length = 88`, `extend-ignore = E203,E501,E701` (и при необходимости W503 не включать). Рекомендуется плагин Bugbear и B950 вместо E501 — см. [Внедрение в проект и другие инструменты](introducing_black_to_your_project.md).
- **Pylint:** `max-line-length = 88`.

Подробные совместимые конфигурации — в [Внедрение в проект и использование с другими инструментами](introducing_black_to_your_project.md).

### mypy

Black не конфликтует с mypy. Используйте mypy для проверки типов отдельно; конфиг в `[tool.mypy]` в `pyproject.toml` или в `mypy.ini`.

---

## 11. Практика: примеры до/после

### Простое форматирование

**До:**

```python
import os,sys
from datetime import datetime

def my_function(x,y,z):
    result=x+y+z
    return result
```

**После:**

```python
import os
import sys
from datetime import datetime


def my_function(x, y, z):
    result = x + y + z
    return result
```

### Длинные вызовы и коллекции

**До:**

```python
def configure_database(connection_string, timeout=30, max_retries=5, debug=False, enable_ssl=True):
    config = {"connection_string": connection_string, "timeout": timeout, "max_retries": max_retries}
    return config

users = [u for u in get_users() if u.active and u.created_at > datetime(2024, 1, 1) and u.subscription_type == "premium"]
```

**После:**

```python
def configure_database(
    connection_string, timeout=30, max_retries=5, debug=False, enable_ssl=True
):
    config = {
        "connection_string": connection_string,
        "timeout": timeout,
        "max_retries": max_retries,
    }
    return config


users = [
    u
    for u in get_users()
    if u.active
    and u.created_at > datetime(2024, 1, 1)
    and u.subscription_type == "premium"
]
```

### Строки и кавычки

**До:**

```python
message = 'Hello world'
quote = "He said 'hello' to me"
```

**После:**

```python
message = "Hello world"
quote = 'He said "hello" to me'
```

### Цепочка вызовов (line-length=88)

**До:**

```python
return data.transform(option_one).filter(option_two).aggregate(option_three).export(option_four)
```

**После:**

```python
return (
    data.transform(option_one)
    .filter(option_two)
    .aggregate(option_three)
    .export(option_four)
)
```

### Trailing comma

**До (с запятой в конце):**

```python
items = [
    "item1",
    "item2",
    "item3",
]
```

**После:** Black оставит многострочный вид. Без trailing comma и при короткой строке Black может упаковать в одну строку.

### Реальный пример: DAG (Airflow-подобный код)

**До:**

```python
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime,timedelta

default_args = {'owner':'data-team','retries':3,'retry_delay':timedelta(minutes=5)}

with DAG(dag_id='data_pipeline', default_args=default_args, schedule_interval='0 2 * * *', start_date=datetime(2024,1,1), catchup=False) as dag:
    extract_task = PythonOperator(task_id='extract', python_callable=extract_data)
    transform_task = PythonOperator(task_id='transform', python_callable=transform_data, op_kwargs={'extracted_data': extract_task.output})
    extract_task >> transform_task
```

**После:**

```python
from datetime import datetime, timedelta

from airflow import DAG
from airflow.operators.python import PythonOperator

default_args = {
    "owner": "data-team",
    "retries": 3,
    "retry_delay": timedelta(minutes=5),
}

with DAG(
    dag_id="data_pipeline",
    default_args=default_args,
    schedule_interval="0 2 * * *",
    start_date=datetime(2024, 1, 1),
    catchup=False,
) as dag:
    extract_task = PythonOperator(task_id="extract", python_callable=extract_data)
    transform_task = PythonOperator(
        task_id="transform",
        python_callable=transform_data,
        op_kwargs={"extracted_data": extract_task.output},
    )
    extract_task >> transform_task
```

(Импорты полностью отсортирует isort; Black выровняет переносы и кавычки.)

---

## 12. Типичные ошибки и решения

### Ошибка 1: В CI другой результат, чем локально

**Причина:** разные версии Black.  
**Решение:** зафиксировать версию в dev-зависимостях и в CI (`black==24.10.0` или `required-version` в `pyproject.toml`).

### Ошибка 2: Конфликт с Flake8 (E501, E203)

**Причина:** Flake8 ругается на строки, отформатированные Black.  
**Решение:** `max-line-length = 88`, `extend-ignore = E203,E501,E701`. Лучше использовать Bugbear и B950 вместо E501 — см. [Внедрение в проект](introducing_black_to_your_project.md).

### Ошибка 3: Конфликт с isort

**Причина:** разный порядок/формат импортов.  
**Решение:** `[tool.isort]` с `profile = "black"` и `line_length = 88`; в pipeline сначала isort, потом Black.

### Ошибка 4: Black не форматирует файл

**Причина:** файл в exclude или уже отформатирован.  
**Решение:** `black --check --verbose .` — посмотреть, какие файлы игнорируются; проверить `extend-exclude` и путь запуска.

### Ошибка 5: Black трогает слишком много файлов

**Причина:** запуск из корня без корректного exclude или нужны только часть путей.  
**Решение:** ограничить пути (`black src tests`) и настроить `extend-exclude` в `pyproject.toml`.

### Ошибка 6: Pre-commit hook не срабатывает

**Решение:** `pre-commit uninstall`, затем `pre-commit install`; проверить `pre-commit run --all-files`.

### Ошибка 7: Слишком шумный PR

**Решение:** отдельный PR только на форматирование, без логических изменений; после merge включить pre-commit и CI-check.

---

## 13. Enterprise: структура проекта и автоматизация

### Пример структуры

```
my_project/
├── .pre-commit-config.yaml
├── .git-blame-ignore-revs
├── .github/workflows/code-quality.yml   # или .gitlab-ci.yml
├── pyproject.toml
├── src/
│   └── my_package/
├── tests/
├── scripts/
│   └── check-black.sh
├── Makefile
└── README.md
```

### Makefile

```makefile
.PHONY: format lint check

format:
	isort . --profile black
	black .

lint:
	isort --check-only . --profile black
	black --check .

check: lint
	pytest tests/ -v
```

Использование: `make format`, `make lint`, `make check`.

### Форматирование только изменённых файлов

```bash
# Только файлы, изменённые относительно main
black $(git diff --name-only main -- "*.py")

# Только staged-файлы
black $(git diff --cached --name-only -- "*.py")
```

### Docker

В образе для CI можно установить Black и запускать проверку:

```dockerfile
RUN pip install black
RUN black --check .
```

### Документирование в README

Указать: «Форматирование: Black. Конфиг в `pyproject.toml`. Перед коммитом: `make format` или pre-commit. В CI выполняется `black --check .`.»

---

## 14. FAQ и антипаттерны

### FAQ

**Нужно ли обсуждать стиль после внедрения Black?**  
Почти нет. Обсуждайте архитектуру, доменные решения, контракты API и тесты.

**Можно ли гибко настраивать Black под любой вкус?**  
Нет. Мало опций — меньше расхождений между командами; это плюс.

**Стоит ли запускать Black в runtime (на сервере)?**  
Нет. Форматирование — часть developer workflow и CI, не production.

**Что делать с legacy-кодом на сотни тысяч строк?**  
Один форматирующий PR + pre-commit + CI; в монорепо — по пакетам.

**Black поддерживает setup.cfg?**  
Нет. Только `pyproject.toml` (TOML).

### Антипаттерны

- Включить Black в проекте, но не фиксировать версию.
- Смешивать логические изменения и массовое форматирование в одном PR.
- Не использовать pre-commit, полагаясь на ручную дисциплину.
- Игнорировать CI-проверку `black --check`.
- Уходить в бесконечную кастомизацию форматирования (обход через `# fmt: off` везде).

---

## 15. Чек-лист для enterprise

- [ ] Black добавлен в dev-зависимости и версия зафиксирована (requirements-dev.txt или pyproject.toml).
- [ ] В корне есть `pyproject.toml` с секцией `[tool.black]` (line-length, target-version, extend-exclude).
- [ ] Задана `required-version` для Black.
- [ ] В CI выполняется `black --check .` (или выбранные пути).
- [ ] Включен pre-commit хук для Black (и при необходимости isort перед ним).
- [ ] При первой миграции: один коммит с применением Black, его хеш добавлен в `.git-blame-ignore-revs`.
- [ ] Линтеры и isort настроены под Black (line-length 88, совместимые правила).
- [ ] В README или CONTRIBUTING указано: «Форматирование: Black, конфиг в pyproject.toml».

После этого Black становится единственным источником правды по стилю кода в репозитории.

---

*См. также: [Основы](the_basics.md), [Внедрение в проект и другие инструменты](introducing_black_to_your_project.md), [Стиль кода Black](code-style.md).*
