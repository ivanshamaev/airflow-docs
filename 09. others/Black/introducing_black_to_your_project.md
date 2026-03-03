# Внедрение Black в проект

Note

Это руководство неполное. Предложения и правки приветствуются!

## Как не испортить git blame

Один из старых аргументов против перехода на автоматические форматтеры вроде Black — миграция засоряет вывод `git blame`. Раньше это было так, но начиная с Git 2.23 поддерживается [игнорирование ревизий в blame](https://git-scm.com/docs/git-blame#Documentation/git-blame.txt---ignore-revltrevgt) через опцию `--ignore-rev`. Можно также передать файл со списком ревизий через `--ignore-revs-file`. Изменения из указанных ревизий не будут учитываться при назначении авторства; строкам будет приписываться последняя ревизия до игнорируемой, которая их меняла.

При переходе стиля проекта на Black отформатируйте весь код и закоммитьте изменения (лучше одним большим коммитом). Затем запишите полный 40-символьный идентификатор коммита (или несколько) в файл, обычно называемый `.git-blame-ignore-revs`, в корне проекта.

```
# Migrate code style to Black
5b4ab991dede475d393e9d69ec388fd6bd949699
```

После этого передавайте этот файл в `git blame` — авторство будет актуальным и без шума от форматирования.

```bash
$ git blame important.py --ignore-revs-file .git-blame-ignore-revs
7a1ae265 (John Smith 2019-04-15 15:55:13 -0400 1) def very_important_function(text, file):
abdfd8b0 (Alice Doe  2019-09-23 11:39:32 -0400 2)     text = text.lstrip()
7a1ae265 (John Smith 2019-04-15 15:55:13 -0400 3)     with open(file, "r+") as f:
7a1ae265 (John Smith 2019-04-15 15:55:13 -0400 4)         f.write(formatted)
```

Можно настроить Git так, чтобы при каждом вызове `git blame` автоматически использовался файл с игнорируемыми ревизиями:

```bash
$ git config blame.ignoreRevsFile .git-blame-ignore-revs
```

Ограничение: не все веб-интерфейсы репозиториев умеют игнорировать ревизии в своём blame. На таких платформах коммит с переформатированием всё равно будет виден. [GitHub](https://docs.github.com/en/repositories/working-with-files/using-files/viewing-a-file#ignore-commits-in-the-blame-view) и [GitLab (с версии 17.10)](https://about.gitlab.com/releases/2025/03/20/gitlab-17-10-released/#ignore-specific-revisions-in-git-blame) по умолчанию поддерживают `.git-blame-ignore-revs` в представлении blame.

---

# Использование Black вместе с другими инструментами

## Совместимые с Black конфигурации

Изменения Black безвредны (или должны быть такими), но часть из них конфликтует с другими инструментами. Часто вместе с Black используют линтеры и проверки типов. Некоторые из них нужно слегка подстроить. Ниже — совместимые с Black конфигурации в разных форматах для распространённых инструментов.

Black поддерживает только конфигурацию в TOML (например, `pyproject.toml`). Приведённые примеры задают только настройки соответствующих инструментов в их собственных форматах файлов.

Готовые совместимые конфигурации: [compatible_configs](https://github.com/psf/black/blob/main/docs/compatible_configs/).

### isort

[isort](https://pypi.org/p/isort/) упорядочивает и форматирует импорты. Black тоже форматирует импорты, но иначе, чем isort по умолчанию, из-за чего возникают конфликты.

#### Профиль

С версии 5.0.0 isort поддерживает [профили](https://pycqa.github.io/isort/docs/configuration/profiles.html) для совместимости с распространёнными стилями. Профиль Black можно задать в любом [поддерживаемом isort конфиге](https://pycqa.github.io/isort/docs/configuration/config_files.html). Пример для `pyproject.toml`:

```toml
[tool.isort]
profile = "black"
```

#### Ручная настройка

Если используется isort старше 5.0.0 или нужна своя настройка под Black, можно задать совместимые опции. Пример для `.isort.cfg`:

```ini
multi_line_output = 3
include_trailing_comma = True
force_grid_wrap = 0
use_parentheses = True
ensure_newline_before_comments = True
line_length = 88
```

#### Зачем эти опции

Black переносит длинные импорты, вынося идентификаторы на отдельные строки и добавляя trailing comma. Подробнее: [How Black wraps lines](https://black.readthedocs.io/en/stable/the_black_code_style/current_style.html#how-black-wraps-lines).

Режим переноса импортов isort по умолчанию — «Grid»:

```python
from third_party import (lib1, lib2, lib3,
                         lib4, lib5, ...)
```

Он несовместим с Black. В isort можно включить режим «Vertical Hanging Indent»:

```python
from third_party import (
    lib1,
    lib2,
    lib3,
    lib4,
)
```

Это достигается опцией `multi_line_output = 3`. Black при переносе ставит trailing comma и использует скобки — для совпадения нужны `include_trailing_comma = True` и `use_parentheses = True`. `force_grid_wrap = 0` — переносить только когда импорт длиннее `line_length`. Задайте `line_length = 88` (как у Black) и `ensure_newline_before_comments = True`, чтобы секции импортов с комментариями совпадали с Black.

`ensure_newline_before_comments = True` работает с isort >= 5; в более старых версиях не ломает конфиг, его можно оставить.

#### Форматы конфигов

**.isort.cfg**

```ini
[settings]
profile = black
```

**setup.cfg**

```ini
[isort]
profile = black
```

**pyproject.toml**

```toml
[tool.isort]
profile = 'black'
```

**.editorconfig**

```ini
[*.py]
profile = black
```

### pycodestyle

[pycodestyle](https://pycodestyle.pycqa.org/) — линтер стиля кода, предупреждает о синтаксисе, возможных багах и отступлениях от [PEP 8](https://www.python.org/dev/peps/pep-0008/). Несколько отступлений дают конфликты с Black.

#### Конфигурация

```ini
max-line-length = 88
ignore = E203,E701
```

#### Зачем эти опции

**max-line-length** — как и для isort, разрешать строки до 88 символов (значение по умолчанию в Black).

**E203** — в срезах Black по правилам PEP 8 выравнивает пробелы вокруг `:`, из-за чего pycodestyle выдаёт `E203 whitespace before ':'`. Это предупреждение не соответствует PEP 8, его нужно отключить.

**E701 / E704** — Black сворачивает в одну строку тела функций и классов из одного `...`. В остальных случаях Black не допускает несколько утверждений в одной строке. pycodestyle может выдавать `E701 multiple statements on one line (colon)`; правило `E704 multiple statements on one line (def)` по умолчанию выключено — его не включайте.

**W503** — при переносе строки Black переносит *перед* бинарным оператором, что соответствует PEP 8 (с [апреля 2016](https://github.com/python/peps/commit/c59c4376ad233a62ca4b3a6060c81368bd21e85b#diff-64ec08cc46db7540f18f2af46037f599)). В Flake8 есть выключенное по умолчанию предупреждение `W503 line break before binary operator`, противоречащее этой рекомендации — его не включайте. Можно использовать `W504 line break after binary operator`.

#### Форматы

**setup.cfg, .pycodestyle, tox.ini**

```ini
[pycodestyle]
max-line-length = 88
ignore = E203,E701
```

### Flake8

[Flake8](https://pypi.org/p/flake8/) объединяет несколько линтеров, в том числе pycodestyle, поэтому у него те же конфликты.

#### Bugbear

Рекомендуется использовать [плагин Bugbear](https://github.com/PyCQA/flake8-bugbear) и включить [проверку B950](https://github.com/PyCQA/flake8-bugbear#opinionated-warnings#:~:text=you%20expect%20it.-,B950,-%3A%20Line%20too%20long) вместо E501 во Flake8 — она согласована с [правилом 10% в Black](https://black.readthedocs.io/en/stable/the_black_code_style/current_style.html#labels-line-length).

Установите Bugbear и настройте так:

```ini
[flake8]
max-line-length = 80
extend-select = B950
extend-ignore = E203,E501,E701
```

#### Минимальная конфигурация

Если Bugbear установить нельзя или не хочется, можно ограничиться минимально совместимой конфигурацией:

```ini
[flake8]
max-line-length = 88
extend-ignore = E203,E701
```

#### Зачем эти опции

См. раздел про pycodestyle выше.

#### Форматы

**.flake8, setup.cfg, tox.ini**

```ini
[flake8]
max-line-length = 88
extend-ignore = E203,E701
```

### Pylint

[Pylint](https://pypi.org/p/pylint/) — линтер, во многом похожий на Flake8, с дополнительными проверками форматирования и стиля (например, именование переменных).

#### Конфигурация

```ini
max-line-length = 88
```

#### Зачем

Ограничить предупреждения о длине строк порогом в 88 символов.

Если используется `pylint<2.6.0`, отключите также `C0326` и `C0330` — они несовместимы с форматированием Black и в новых версиях удалены.

#### Форматы

**pylintrc**

```ini
[format]
max-line-length = 88
```

**setup.cfg**

```ini
[pylint]
max-line-length = 88
```

**pyproject.toml**

```toml
[tool.pylint.format]
max-line-length = "88"
```

---

*Источники: [Introducing Black to your project](https://black.readthedocs.io/en/stable/guides/introducing_black_to_your_project.html), [Using Black with other tools](https://black.readthedocs.io/en/stable/guides/using_black_with_other_tools.html). Перевод неофициальный.*
