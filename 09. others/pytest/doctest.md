# Как запускать doctests

По умолчанию все файлы, соответствующие шаблону `test*.txt`, будут выполняться через стандартный модуль Python [doctest](https://docs.python.org/3/library/doctest.html#module-doctest). Шаблон можно изменить, выполнив:

```
pytest --doctest-glob="*.rst"

```

в командной строке. Опцию [--doctest-glob](https://docs.pytest.org/en/stable/reference/reference.html#cmdoption-doctest-glob) можно указывать несколько раз.

Если затем у вас есть текстовый файл, например такой:

```
# content of test_example.txt

hello this is a doctest
>>> x = 3
>>> x
3

```

то можно просто вызвать `pytest`:

```
$ pytest
=========================== test session starts ============================
platform linux -- Python 3.x.y, pytest-9.x.y, pluggy-1.x.y
rootdir: /home/sweet/project
collected 1 item

test_example.txt .                                                   [100%]

============================ 1 passed in 0.12s =============================

```

По умолчанию pytest собирает (collect) файлы `test*.txt` и ищет директивы doctest, но вы можете передать дополнительные шаблоны glob через опцию [--doctest-glob](https://docs.pytest.org/en/stable/reference/reference.html#cmdoption-doctest-glob) (можно указывать несколько раз).

Помимо текстовых файлов, можно выполнять doctest прямо из docstring ваших классов и функций, включая docstring из тестовых модулей:

```
# content of mymodule.py
def something():
    """a doctest in a docstring
    >>> something()
    42
    """
    return 42

```

```
$ pytest --doctest-modules
=========================== test session starts ============================
platform linux -- Python 3.x.y, pytest-9.x.y, pluggy-1.x.y
rootdir: /home/sweet/project
collected 2 items

mymodule.py .                                                        [ 50%]
test_example.txt .                                                   [100%]

============================ 2 passed in 0.12s =============================

```

Эти изменения можно сделать постоянными для проекта, добавив их в конфигурационный файл, например так:

```
# content of pytest.toml
[pytest]
addopts = ["--doctest-modules"]

```

## Кодировка

Кодировка по умолчанию — UTF-8, но можно указать кодировку, которая будет использоваться для doctest-файлов, через опцию конфигурации [doctest_encoding](https://docs.pytest.org/en/stable/reference/reference.html#confval-doctest_encoding):

toml

```
[pytest]
doctest_encoding = "latin1"

```

ini

```
[pytest]
doctest_encoding = latin1

```

## Использование опций ‘doctest’

Стандартный модуль Python [doctest](https://docs.python.org/3/library/doctest.html#module-doctest) предоставляет некоторые [опции](https://docs.python.org/3/library/doctest.html#option-flags-and-directives), которые задают «строгость» doctest-тестов. В pytest эти флаги можно включить через конфигурационный файл.

Например, чтобы игнорировать пробелы в конце строк и игнорировать длинные traceback исключений, можно написать:

toml

```
[pytest]
doctest_optionflags = ["NORMALIZE_WHITESPACE", "IGNORE_EXCEPTION_DETAIL"]

```

ini

```
[pytest]
doctest_optionflags = NORMALIZE_WHITESPACE IGNORE_EXCEPTION_DETAIL

```

Также опции можно включать inline-комментарием прямо в doctest:

```
>>> something_that_raises()  # doctest: +IGNORE_EXCEPTION_DETAIL
Traceback (most recent call last):
ValueError: ...

```

pytest также вводит новые опции:

`ALLOW_UNICODE`: если включено, префикс `u` удаляется из unicode-строк в ожидаемом выводе doctest. Это позволяет запускать doctest одинаково в Python 2 и Python 3.

`ALLOW_BYTES`: аналогично, префикс `b` удаляется из байтовых строк в ожидаемом выводе doctest.

`NUMBER`: если включено, числа с плавающей точкой должны совпадать только до точности, указанной вами в ожидаемом выводе doctest. Числа сравниваются через [pytest.approx()](https://docs.pytest.org/en/stable/reference/reference.html#pytest.approx) с относительным допуском, равным точности. Например, следующий вывод должен совпасть только до 2 десятичных знаков при сравнении `3.14` с `pytest.approx(math.pi, rel=10**-2)`:

```
>>> math.pi
3.14

```

Если бы вы написали `3.1416`, то фактический вывод должен был бы совпасть примерно до 4 десятичных знаков; и так далее.

Это помогает избежать ложных срабатываний из‑за ограниченной точности floating point, например:

```
Expected:
    0.233
Got:
    0.23300000000000001

```

`NUMBER` также поддерживает списки чисел с плавающей точкой — фактически он находит числа с плавающей точкой в любом месте вывода, даже внутри строки! Поэтому включать его глобально в `doctest_optionflags` в конфигурации может быть неуместно.

Добавлено в версии 5.1.

## Продолжение после падения

По умолчанию pytest сообщает только о первом падении для заданного doctest. Если вы хотите продолжить выполнение теста даже при ошибках, используйте:

```
pytest --doctest-modules --doctest-continue-on-failure

```

## Формат вывода

Можно изменить формат diff-вывода при падении doctest, используя один из стандартных форматов модуля doctest (см. [doctest.REPORT_UDIFF](https://docs.python.org/3/library/doctest.html#doctest.REPORT_UDIFF), [doctest.REPORT_CDIFF](https://docs.python.org/3/library/doctest.html#doctest.REPORT_CDIFF), [doctest.REPORT_NDIFF](https://docs.python.org/3/library/doctest.html#doctest.REPORT_NDIFF), [doctest.REPORT_ONLY_FIRST_FAILURE](https://docs.python.org/3/library/doctest.html#doctest.REPORT_ONLY_FIRST_FAILURE)):

```
pytest --doctest-modules --doctest-report none
pytest --doctest-modules --doctest-report udiff
pytest --doctest-modules --doctest-report cdiff
pytest --doctest-modules --doctest-report ndiff
pytest --doctest-modules --doctest-report only_first_failure

```

## Возможности, специфичные для pytest

Есть некоторые возможности, которые упрощают написание doctest или улучшают интеграцию с вашим набором тестов. Однако имейте в виду: используя эти возможности, вы сделаете doctest несовместимыми со стандартным модулем `doctests`.

### Использование фикстур

Можно использовать фикстуры через хелпер `getfixture`:

```
# content of example.rst
>>> tmp = getfixture('tmp_path')
>>> ...
>>>

```

Обратите внимание: фикстура должна быть определена в месте, доступном pytest, например в `conftest.py` или плагине. Обычные python-файлы с docstring по умолчанию не сканируются на фикстуры, если это явно не настроено опцией [python_files](https://docs.pytest.org/en/stable/reference/reference.html#confval-python_files).

Также маркер [usefixtures](https://docs.pytest.org/en/stable/how-to/fixtures.html#usefixtures) и фикстуры с опцией [autouse](https://docs.pytest.org/en/stable/how-to/fixtures.html#autouse) поддерживаются при выполнении текстовых doctest-файлов.

### Фикстура ‘doctest_namespace’

Фикстура `doctest_namespace` позволяет внедрять объекты в пространство имён, в котором выполняются doctest. Она предназначена для использования внутри ваших фикстур, чтобы давать тестам контекст.

`doctest_namespace` — это обычный объект `dict`, в который нужно поместить объекты, которые должны быть доступны в пространстве имён doctest:

```
# content of conftest.py
import pytest
import numpy


@pytest.fixture(autouse=True)
def add_np(doctest_namespace):
    doctest_namespace["np"] = numpy

```

После этого ими можно пользоваться в doctest напрямую:

```
# content of numpy.py
def arange():
    """
    >>> a = np.arange(10)
    >>> len(a)
    10
    """

```

Как и в обычном `conftest.py`, фикстуры обнаруживаются в дереве каталогов, где находится conftest. Это означает, что если doctest лежит рядом с исходным кодом, соответствующий `conftest.py` должен находиться в том же дереве каталогов. Фикстуры не будут обнаруживаться в «соседнем» дереве каталогов!

### Пропуск тестов

По тем же причинам, по которым пропускают обычные тесты, можно пропускать тесты внутри doctest.

Чтобы пропустить одну проверку внутри doctest, можно использовать стандартную директиву [doctest.SKIP](https://docs.python.org/3/library/doctest.html#doctest.SKIP):

```
def test_random(y):
    """
    >>> random.random()  # doctest: +SKIP
    0.156231223

    >>> 1 + 1
    2
    """

```

Это пропустит первую проверку, но не вторую.

pytest также позволяет использовать стандартные функции pytest [pytest.skip()](https://docs.pytest.org/en/stable/reference/reference.html#pytest.skip) и [pytest.xfail()](https://docs.pytest.org/en/stable/reference/reference.html#pytest.xfail) внутри doctest. Это может быть полезно, потому что можно пропускать/xfail тесты в зависимости от внешних условий:

```
>>> import sys, pytest
>>> if sys.platform.startswith('win'):
...     pytest.skip('this doctest does not work on Windows')
...
>>> import fcntl
>>> ...

```

Однако использование этих функций не рекомендуется, потому что оно снижает читаемость docstring.

Note

[pytest.skip()](https://docs.pytest.org/en/stable/reference/reference.html#pytest.skip) и [pytest.xfail()](https://docs.pytest.org/en/stable/reference/reference.html#pytest.xfail) ведут себя по‑разному в зависимости от того, где находятся doctest:

Python modules (docstrings): функции действуют только в этом конкретном docstring, позволяя другим docstring в том же модуле выполниться как обычно.

Text files: функции пропускают/xfail проверки до конца всего файла.

## Альтернативы

Хотя встроенная поддержка pytest даёт хороший набор функций для doctest, при активном использовании doctest могут быть интересны внешние пакеты, которые добавляют больше возможностей и интеграцию с pytest:

[pytest-doctestplus](https://github.com/scientific-python/pytest-doctestplus): расширенная поддержка doctest и тестирование файлов reStructuredText (“.rst”).

[Sybil](https://sybil.readthedocs.io/): способ тестировать примеры в документации, парся их из исходников документации и выполняя как часть обычного запуска тестов.