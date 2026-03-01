# Форматтер Ruff (The Ruff Formatter)

Форматтер Ruff — чрезвычайно быстрый форматтер кода на Python, задуманный как замена [Black](https://pypi.org/project/black/), доступный в составе CLI `ruff` через команду `ruff format`.

## ruff format

`ruff format` — основная точка входа в форматтер. Команда принимает список файлов или каталогов и форматирует все обнаруженные Python-файлы:

```
ruff format                   # Форматировать все файлы в текущем каталоге.
ruff format path/to/code/     # Форматировать все файлы в path/to/code (и во вложенных каталогах).
ruff format path/to/file.py   # Форматировать один файл.
```

Как и в Black, запуск `ruff format /path/to/file.py` форматирует указанный файл или каталог на месте, а `ruff format --check /path/to/file.py` не записывает отформатированные файлы обратно, а завершается с ненулевым кодом при обнаружении неотформатированных файлов.

Полный список поддерживаемых опций: `ruff format --help`.

## Философия (Philosophy)

Изначальная цель форматтера Ruff — не менять стиль кода, а повысить производительность и обеспечить единый набор инструментов для линтера Ruff, форматтера и любых будущих инструментов.

Поэтому форматтер сделан как замена [Black](https://github.com/psf/black) с упором на производительность и прямую интеграцию с Ruff. Учитывая популярность Black в экосистеме Python, ориентация на совместимость с Black минимизирует последствия перехода для большинства проектов.

В частности, форматтер призван выдавать почти идентичный вывод при запуске над кодом, уже отформатированным Black. На крупных проектах вроде Django и Zulip совпадает > 99.9% строк. (См. Style Guide.)

Исходя из этой ориентации на совместимость с Black, форматтер следует [стабильному стилю кода Black](https://black.readthedocs.io/en/stable/the_black_code_style/current_style.html), который стремится к «единообразию, общности, читаемости и уменьшению git diff». Чтобы дать представление о применяемом стиле, пример:

```
# Вход
def _make_ssl_transport(
    rawsock, protocol, sslcontext, waiter=None,
    *, server_side=False, server_hostname=None,
    extra=None, server=None,
    ssl_handshake_timeout=None,
    call_connection_made=True):
    '''Make an SSL transport.'''
    if waiter is None:
      waiter = Future(loop=loop)

    if extra is None:
      extra = {}

    ...

# Ruff
def _make_ssl_transport(
    rawsock,
    protocol,
    sslcontext,
    waiter=None,
    *,
    server_side=False,
    server_hostname=None,
    extra=None,
    server=None,
    ssl_handshake_timeout=None,
    call_connection_made=True,
):
    """Make an SSL transport."""
    if waiter is None:
        waiter = Future(loop=loop)

    if extra is None:
        extra = {}

    ...

```

Как и Black, форматтер Ruff не поддерживает широкую настройку стиля кода; в отличие от Black, можно настроить стиль кавычек, отступов, окончаний строк и другое. (См. Configuration.)

Хотя форматтер рассчитан на замену Black, он не предназначен для постоянного чередования с Black, так как в нескольких осознанных моментах отличается от Black (см. [Known deviations](https://docs.astral.sh/ruff/formatter/black/)). В целом отличия ограничены случаями, где поведение Ruff сочли более последовательным или существенно проще в реализации (при незначительном влиянии на пользователя) из-за различий в реализации Black и Ruff.

В дальнейшем форматтер Ruff будет поддерживать preview-стиль Black в рамках собственного [режима preview](https://docs.astral.sh/ruff/preview/) Ruff.

## Конфигурация (Configuration)

Форматтер Ruff предоставляет небольшой набор опций конфигурации: часть общая с Black (например длина строки), часть уникальна для Ruff (кавычки, стиль отступов, форматирование примеров кода в docstring).

Например, чтобы настроить форматтер на одинарные кавычки, форматирование примеров кода в docstring, длину строки 100 и отступ табуляцией, добавьте в конфигурационный файл:

**pyproject.toml**:

```toml
[tool.ruff]
line-length = 100

[tool.ruff.format]
quote-style = "single"
indent-style = "tab"
docstring-code-format = true
```

**ruff.toml**:

```toml
line-length = 100

[format]
quote-style = "single"
indent-style = "tab"
docstring-code-format = true
```

Полный список поддерживаемых настроек: [Settings](https://docs.astral.sh/ruff/settings/#format). Подробнее о настройке Ruff через `pyproject.toml`: [Configuring Ruff](https://docs.astral.sh/ruff/configuration/).

С учётом ориентации на совместимость с Black (и в отличие от форматтеров вроде [YAPF](https://github.com/google/yapf)) Ruff пока не предоставляет других опций конфигурации.

## Форматирование docstring (Docstring formatting)

Форматтер Ruff предоставляет опциональную возможность автоматически форматировать примеры кода на Python в docstring. Сейчас распознаются примеры кода в форматах:

- reStructuredText: директивы [code-block и sourcecode](https://www.sphinx-doc.org/en/master/usage/restructuredtext/directives.html#directive-code-block). Как и в Markdown, для Python распознаются языки `python`, `py`, `python3` или `py3`.
- reStructuredText: [literal blocks](https://docutils.sourceforge.io/docs/ref/rst/restructuredtext.html#literal-blocks). Хотя literal blocks могут содержать не только Python, это отражает давнюю традицию в экосистеме Python, где literal blocks часто содержат код на Python.
- CommonMark [fenced code blocks](https://spec.commonmark.org/0.30/#fenced-code-blocks) с info string: `python`, `py`, `python3` или `py3`. Блоки без info string тоже считаются примерами кода на Python и форматируются.
- Формат [doctest](https://docs.python.org/3/library/doctest.html) в Python.

Если пример распознан и обрабатывается как Python, форматтер Ruff автоматически пропустит его, если код не парсится как корректный Python или если отформатированный код дал бы невалидную программу на Python.

Пользователь может также настроить лимит длины строки для переформатирования примеров кода на Python в docstring. По умолчанию используется специальное значение `dynamic`, которое предписывает форматтеру соблюдать лимит длины строки для окружающего кода на Python. Настройка `dynamic` обеспечивает, что даже при примерах кода во вложенных docstring лимит длины строки для окружающего кода не будет превышен. Можно задать и фиксированный лимит длины строки для примеров в docstring.

Пример конфигурации с фиксированным лимитом длины строки для примеров в docstring:

**pyproject.toml**:

```toml
[tool.ruff.format]
docstring-code-format = true
docstring-code-line-length = 20
```

**ruff.toml**:

```toml
[format]
docstring-code-format = true
docstring-code-line-length = 20
```

При такой конфигурации этот код:

```
def f(x):
    '''
    Something about `f`. And an example:

    .. code-block:: python

        foo, bar, quux = this_is_a_long_line(lion, hippo, lemur, bear)
    '''
    pass

```

… будет переформатирован (при остальных опциях по умолчанию) так:

```
def f(x):
    """
    Something about `f`. And an example:

    .. code-block:: python

        (
            foo,
            bar,
            quux,
        ) = this_is_a_long_line(
            lion,
            hippo,
            lemur,
            bear,
        )
    """
    pass

```

## Форматирование кода в Markdown (Markdown code formatting)

Эта возможность сейчас доступна только в [режиме preview](https://docs.astral.sh/ruff/preview/#preview).

Форматтер Ruff может также форматировать блоки кода на Python в файлах Markdown. В таких файлах Ruff форматирует любые CommonMark [fenced code blocks](https://spec.commonmark.org/0.30/#fenced-code-blocks) с info string: `python`, `py`, `python3`, `py3` или `pyi`. Блоки без info string считаются примерами кода на Python и тоже форматируются.

Если пример распознан и обрабатывается как Python, форматтер Ruff автоматически пропустит его, если код не парсится как корректный Python или если отформатированный код дал бы невалидную программу на Python.

Блоки с пометкой `python`, `py`, `python3` или `py3` форматируются в обычном стиле кода на Python, а блоки с пометкой `pyi` — как файлы type stub на Python:

```
```py
print("hello")
```

```pyi
def foo(): ...
def bar(): ...
```

```

Ruff также поддерживает исполняемые блоки кода в стиле [Quarto](https://quarto.org/) с фигурными скобками вокруг имени языка:

```
```{python}
print("hello")
```

```

Комментарии подавления форматирования внутри блоков кода обрабатываются как обычно; кроме того, форматтер пропустит любой блок кода, окружённый подходящими HTML-комментариями, например:

```
<!-- fmt: off -->
```py
print( 'hello' )
```
<!-- fmt: on -->
```

Любое количество блоков кода может находиться между парой комментариев `off` и `on`; комментарий `off` без парного `on` неявно действует до конца документа.

Форматтер Ruff также распознаёт HTML-комментарии из [blacken-docs](https://github.com/adamchainz/blacken-docs/): `<!-- blacken-docs:off -->` и `<!-- blacken-docs:on -->`, эквивалентные соответственно `<!-- fmt:off -->` и `<!-- fmt:on -->`.

Ruff не обнаруживает и не форматирует Markdown-файлы в проекте автоматически, но форматирует любые Markdown-файлы, явно переданные с расширением `.md`:

```
$ ruff format --preview --check docs/
warning: No Python files found under the given path(s)

$ ruff format --preview --check docs/*.md
13 files already formatted

```

Вероятно, это изменится в будущем релизе при стабилизации возможности. Подключение Markdown-файлов без включённого [режима preview](https://docs.astral.sh/ruff/preview/#preview) приведёт к сообщению об ошибке и ненулевому коду выхода.

Чтобы по умолчанию включать Markdown-файлы при запуске Ruff по проекту, добавьте их через [extend-include](https://docs.astral.sh/ruff/settings/#extend-include) в настройках проекта:

**pyproject.toml**:

```toml
[tool.ruff]
# Искать и форматировать блоки кода в Markdown-файлах
extend-include = ["*.md"]
# ИЛИ
extend-include = ["docs/*.md"]
```

**ruff.toml**:

```toml
# Искать и форматировать блоки кода в Markdown-файлах
extend-include = ["*.md"]
# ИЛИ
extend-include = ["docs/*.md"]
```

При запуске Ruff через [ruff-pre-commit](https://github.com/astral-sh/ruff-pre-commit) поддержку Markdown нужно явно включить, добавив её в `types_or`:

**.pre-commit-config.yaml**:

```yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.15.4
    hooks:
      - id: ruff-format
        types_or: [python, pyi, jupyter, markdown]
```

## Подавление форматирования (Format suppression)

Как и Black, Ruff поддерживает прагмы `# fmt: on`, `# fmt: off` и `# fmt: skip`, которыми можно временно отключить форматирование для заданного блока кода.

Комментарии `# fmt: on` и `# fmt: off` действуют на уровне оператора:

```
# fmt: off
not_formatted=3
also_not_formatted=4
# fmt: on

```

Поэтому добавление комментариев `# fmt: on` и `# fmt: off` внутри выражения не даёт эффекта. В следующем примере оба элемента списка будут отформатированы, несмотря на `# fmt: off`:

```
[
    # fmt: off
    '1',
    # fmt: on
    '2',
]
```

Вместо этого примените комментарий `# fmt: off` ко всему оператору:

```
# fmt: off
[
    '1',
    '2',
]
# fmt: on

```

Как и Black, Ruff также распознаёт прагмы [YAPF](https://github.com/google/yapf) `# yapf: disable` и `# yapf: enable`, которые обрабатываются так же, как `# fmt: off` и `# fmt: on` соответственно.

Комментарий `# fmt: skip` отключает форматирование для заголовка case, декоратора, определения функции, определения класса или предшествующих операторов на той же логической строке. Форматтер оставляет следующее без изменений:

```
if True:
    pass
elif False:  # fmt: skip
    pass

@Test
@Test2(a,b)  # fmt: skip
def test(): ...

a = [1,2,3,4,5]  # fmt: skip

def test(a,b,c,d,e,f) -> int:  # fmt: skip
    pass

x=1;x=2;x=3  # fmt: skip

```

Добавление комментария `# fmt: skip` в конце выражения не даёт эффекта. В следующем примере элемент списка `'1'` будет отформатирован, несмотря на `# fmt: skip`:

```
a = call(
    [
        '1',  # fmt: skip
        '2',
    ],
    b
)
```

Вместо этого примените комментарий `# fmt: skip` ко всему оператору:

```
a = call(
    [
        '1',
        '2',
    ],
    b
)  # fmt: skip

```

## Конфликтующие правила линтера (Conflicting lint rules)

Форматтер Ruff рассчитан на совместное использование с линтером. Однако в линтере есть правила, которые при включении могут конфликтовать с форматтером и приводить к неожиданному поведению. При корректной настройке цель совместимости форматтера и линтера Ruff — чтобы запуск форматтера никогда не вносил новых ошибок линтера.

При использовании Ruff как форматтера рекомендуется не включать следующие правила линтера:

- [multi-line-implicit-string-concatenation](https://docs.astral.sh/ruff/rules/multi-line-implicit-string-concatenation/) (ISC002), если используется без ISC001 и `flake8-implicit-str-concat.allow-multiline = false`
- [prohibited-trailing-comma](https://docs.astral.sh/ruff/rules/prohibited-trailing-comma/) (COM819)
- [missing-trailing-comma](https://docs.astral.sh/ruff/rules/missing-trailing-comma/) (COM812)
- [unnecessary-escaped-quote](https://docs.astral.sh/ruff/rules/unnecessary-escaped-quote/) (Q004)
- [avoidable-escaped-quote](https://docs.astral.sh/ruff/rules/avoidable-escaped-quote/) (Q003)
- [bad-quotes-docstring](https://docs.astral.sh/ruff/rules/bad-quotes-docstring/) (Q002)
- [bad-quotes-multiline-string](https://docs.astral.sh/ruff/rules/bad-quotes-multiline-string/) (Q001)
- [bad-quotes-inline-string](https://docs.astral.sh/ruff/rules/bad-quotes-inline-string/) (Q000)
- [triple-single-quotes](https://docs.astral.sh/ruff/rules/triple-single-quotes/) (D300)
- [docstring-tab-indentation](https://docs.astral.sh/ruff/rules/docstring-tab-indentation/) (D206)
- [over-indented](https://docs.astral.sh/ruff/rules/over-indented/) (E117)
- [indentation-with-invalid-multiple-comment](https://docs.astral.sh/ruff/rules/indentation-with-invalid-multiple-comment/) (E114)
- [indentation-with-invalid-multiple](https://docs.astral.sh/ruff/rules/indentation-with-invalid-multiple/) (E111)
- [tab-indentation](https://docs.astral.sh/ruff/rules/tab-indentation/) (W191)

Правило [line-too-long](https://docs.astral.sh/ruff/rules/line-too-long/) (E501) можно использовать вместе с форматтером, но форматтер лишь старается переносить строки в соответствии с настроенной [line-length](https://docs.astral.sh/ruff/settings/#line-length). Поэтому отформатированный код может превышать длину строки и приводить к ошибкам [line-too-long](https://docs.astral.sh/ruff/rules/line-too-long/) (E501).

Ни одно из перечисленных правил не входит в конфигурацию Ruff по умолчанию. Но если вы включили какие-либо из них или их родительские категории (например `Q`), рекомендуется отключить их через настройку [lint.ignore](https://docs.astral.sh/ruff/settings/#lint_ignore) линтера.

Аналогично не рекомендуются следующие настройки isort при нестандартных значениях — они несовместимы с обработкой импортов форматтером:

- [split-on-trailing-comma](https://docs.astral.sh/ruff/settings/#lint_isort_split-on-trailing-comma)
- [lines-between-types](https://docs.astral.sh/ruff/settings/#lint_isort_lines-between-types)
- [lines-after-imports](https://docs.astral.sh/ruff/settings/#lint_isort_lines-after-imports)
- [force-wrap-aliases](https://docs.astral.sh/ruff/settings/#lint_isort_force-wrap-aliases)
- [force-single-line](https://docs.astral.sh/ruff/settings/#lint_isort_force-single-line)

Если вы задали для этих настроек нестандартные значения, рекомендуется убрать их из конфигурации Ruff.

При включённом несовместимом правиле линтера или настройке `ruff format` выведет предупреждение. Если при запуске `ruff format` предупреждений нет — конфигурация в порядке.

## Коды выхода (Exit codes)

`ruff format` завершается со следующими кодами:

- **2** — если Ruff завершился аварийно из-за неверной конфигурации, неверных опций CLI или внутренней ошибки.
- **1** — если Ruff завершился успешно, один или более файлов были отформатированы и указан `--exit-non-zero-on-format`.
- **0** — если Ruff завершился успешно, независимо от того, были ли отформатированы файлы.

Команда `ruff format --check` завершается со следующими кодами:

- **2** — если Ruff завершился аварийно из-за неверной конфигурации, неверных опций CLI или внутренней ошибки.
- **1** — если Ruff завершился успешно и один или более файлов были бы отформатированы при отсутствии `--check`.
- **0** — если Ruff завершился успешно и ни один файл не требовал бы форматирования при отсутствии `--check`.

## Руководство по стилю (Style Guide)

Форматтер рассчитан на замену [Black](https://github.com/psf/black). В этом разделе описаны области, где форматтер Ruff расширяет стиль кода по сравнению с Black.

### Осознанные отличия (Intentional deviations)

Форматтер Ruff стремится быть заменой Black, но в нескольких известных моментах отличается от Black. Часть отличий — осознанные улучшения стиля Black, часть — следствие различий в реализации.

Полный перечень этих осознанных отличий: [Known deviations](https://docs.astral.sh/ruff/formatter/black/).

Непреднамеренные отличия от Black отслеживаются в [issue tracker](https://github.com/astral-sh/ruff/issues?q=is%3Aopen+is%3Aissue+label%3Aformatter). Если вы нашли новое отличиe, можно [создать issue](https://github.com/astral-sh/ruff/issues/new).

### Preview-стиль (Preview style)

По аналогии с [Black](https://black.readthedocs.io/en/stable/the_black_code_style/future_style.html#preview-style), Ruff вносит изменения форматирования под флагом [preview](https://docs.astral.sh/ruff/settings/#format_preview), переводя их в стабильные в минорных релизах в соответствии с [политикой версионирования](https://github.com/astral-sh/ruff/discussions/6998#discussioncomment-7016766).

### Форматирование f-строк (F-string formatting)

Стабилизировано в Ruff 0.9.0.

В отличие от Black, Ruff форматирует части выражений в f-строках — то, что внутри фигурных скобок `{...}`. Это [известное отличиe](https://docs.astral.sh/ruff/formatter/black/#f-strings) от Black.

Ruff использует несколько эвристик для форматирования f-строк, они описаны ниже.

#### Кавычки (Quotes)

Ruff будет использовать [настроенный стиль кавычек](https://docs.astral.sh/ruff/settings/#format_quote-style) для выражения в f-строке, если это не приведёт к невалидному синтаксису для целевой версии Python или не потребует больше экранирований обратным слэшем, чем в исходном выражении. В частности, Ruff сохранит исходный стиль кавычек в следующих случаях:

Когда целевая версия Python < 3.12 и [self-documenting f-string](https://realpython.com/python-f-strings/#self-documenting-expressions-for-debugging) содержит строковый литерал с [настроенным стилем кавычек](https://docs.astral.sh/ruff/settings/#format_quote-style):

```
# format.quote-style = "double"

f'{10 + len("hello")=}'
# Эта f-строка не может быть отформатирована так при целевой версии Python < 3.12
f"{10 + len("hello")=}"

```

Когда целевая версия Python < 3.12 и f-строка содержит любой тройной строковый, байтовый или f-строковый литерал с [настроенным стилем кавычек](https://docs.astral.sh/ruff/settings/#format_quote-style):

```
# format.quote-style = "double"

f'{"""nested " """}'
# Эта f-строка не может быть отформатирована так при целевой версии Python < 3.12
f"{'''nested " '''}"

```

Для всех целевых версий Python, когда [self-documenting f-string](https://realpython.com/python-f-strings/#self-documenting-expressions-for-debugging) содержит выражение в фигурных скобках (`{...}`) с format specifier, содержащим [настроенный стиль кавычек](https://docs.astral.sh/ruff/settings/#format_quote-style):

```
# format.quote-style = "double"

f'{1=:"foo}'
# Эта f-строка не может быть отформатирована так для всех целевых версий Python
f"{1=:"foo}"

```

Для вложенных f-строк Ruff чередует стили кавычек, начиная с [настроенного стиля кавычек](https://docs.astral.sh/ruff/settings/#format_quote-style) для самой внешней f-строки. Например:

```
# format.quote-style = "double"

f"outer f-string {f"nested f-string {f"another nested f-string"} end"} end"

```

Ruff форматирует это так:

```
f"outer f-string {f'nested f-string {f"another nested f-string"} end'} end"

```

#### Переносы строк (Line breaks)

Начиная с Python 3.12 ([PEP 701](https://peps.python.org/pep-0701/)), части выражений в f-строке могут занимать несколько строк. Ruff нужно решать, когда вводить перенос строки в выражении f-строки. Это зависит от семантики выражения — например, перенос в середине естественноязычного предложения нежелателен. Так как у Ruff недостаточно информации для такого решения, используется эвристика, похожая на [Prettier](https://prettier.io/docs/en/next/rationale.html#template-literals): части выражений f-строки переносятся на несколько строк только если внутри какой-либо из них уже был перенос строки.

Например, такой код:

```
f"this f-string has a multiline expression {
    ['red', 'green', 'blue', 'yellow',]
} and does not fit within the line length"

```

… форматируется так:

```
# Выражение-список разбито на несколько строк из-за завершающей запятой
f"this f-string has a multiline expression {
    [
        'red',
        'green',
        'blue',
        'yellow',
    ]
} and does not fit within the line length"

```

Но следующий код не будет разбит на несколько строк, хотя превышает длину строки:

```
f"this f-string has a multiline expression {['red', 'green', 'blue', 'yellow']} and does not fit within the line length"

```

Чтобы Ruff разбил f-строку на несколько строк, обеспечьте перенос строки где-нибудь внутри частей `{...}` f-строки.

### Fluent layout для цепочек методов (Fluent layout for method chains)

Иногда при длинных цепочках методов на объекте, например

```
x = df.filter(cond).agg(func).merge(other)

```

имеется в виду последовательность преобразований или операций над одним и тем же объектом — в примере это `df`. При превышении `line-length` этим preview-стилем выражение форматируется так:

```
x = (
    df
    .filter(cond)
    .agg(func)
    .merge(other)
)

```

Это отличается от стабильного форматирования и от Black, которые дают:

```
x = (
    df.filter(cond)
    .agg(func)
    .merge(other)
)

```

И стабильный, и preview-вариант — разновидности так называемого fluent layout.

В общем случае этот preview-стиль отличается от стабильного только у первого атрибута перед вызовом или подстановкой. В preview форматировании разрыв идёт перед этим атрибутом, в стабильном — после вызова или подстановки.

## Сортировка импортов (Sorting imports)

Сейчас форматтер Ruff не сортирует импорты. Чтобы и сортировать импорты, и форматировать код, вызовите линтер Ruff, затем форматтер:

```
ruff check --select I --fix
ruff format

```

Единая команда для линтинга и форматирования [запланирована](https://github.com/astral-sh/ruff/issues/8232).
