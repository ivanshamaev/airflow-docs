# Как параметризовать фикстуры и тестовые функции

pytest позволяет параметризовать тесты на нескольких уровнях:

[pytest.fixture()](https://docs.pytest.org/en/stable/reference/reference.html#pytest.fixture) позволяет [параметризовать функции фикстур](https://docs.pytest.org/en/stable/how-to/fixtures.html#fixture-parametrize).

@pytest.mark.parametrize позволяет задавать несколько наборов аргументов и фикстур на уровне тестовой функции или класса.

pytest_generate_tests позволяет определять собственные схемы параметризации или расширения.

Note

В качестве альтернативы параметризации см. [How to use subtests](https://docs.pytest.org/en/stable/how-to/subtests.html#subtests).

## @pytest.mark.parametrize: параметризация тестовых функций

Встроенный декоратор [pytest.mark.parametrize](https://docs.pytest.org/en/stable/reference/reference.html#pytest-mark-parametrize-ref) включает параметризацию аргументов для тестовой функции. Типичный пример теста, который проверяет, что определённый вход даёт ожидаемый выход:

```
# content of test_expectation.py
import pytest


@pytest.mark.parametrize("test_input,expected", [("3+5", 8), ("2+4", 6), ("6*9", 42)])
def test_eval(test_input, expected):
    assert eval(test_input) == expected

```

Здесь декоратор `@parametrize` задаёт три разных кортежа `(test_input,expected)`, так что функция `test_eval` будет выполнена три раза, последовательно используя их:

```
$ pytest
=========================== test session starts ============================
platform linux -- Python 3.x.y, pytest-9.x.y, pluggy-1.x.y
rootdir: /home/sweet/project
collected 3 items

test_expectation.py ..F                                              [100%]

================================= FAILURES =================================
____________________________ test_eval[6*9-42] _____________________________

test_input = '6*9', expected = 42

    @pytest.mark.parametrize("test_input,expected", [("3+5", 8), ("2+4", 6), ("6*9", 42)])
    def test_eval(test_input, expected):
>       assert eval(test_input) == expected
E       AssertionError: assert 54 == 42
E        +  where 54 = eval('6*9')

test_expectation.py:6: AssertionError
========================= short test summary info ==========================
FAILED test_expectation.py::test_eval[6*9-42] - AssertionError: assert 54...
======================= 1 failed, 2 passed in 0.12s ========================

```

Note

Параметры передаются в тесты «как есть» (никакого копирования не происходит).

Например, если передать список или словарь в качестве значения параметра, а тестовый код изменит его, изменения отразятся на последующих вызовах теста.

Note

По умолчанию pytest экранирует любые не-ASCII символы, используемые в unicode-строках для параметризации, потому что это имеет несколько недостатков. Если же вы хотите использовать unicode-строки в параметризации и видеть их в терминале как есть (без экранирования), используйте эту опцию в конфигурационном файле:

toml

```
[pytest]
disable_test_id_escaping_and_forfeit_all_rights_to_community_support = true

```

ini

```
[pytest]
disable_test_id_escaping_and_forfeit_all_rights_to_community_support = true

```

Однако помните, что это может привести к нежелательным побочным эффектам и даже ошибкам в зависимости от ОС и установленных плагинов, поэтому используйте на свой риск.

Как задумано в этом примере, только одна пара вход/выход падает. И, как обычно с аргументами тестовой функции, значения `input` и `output` видны в traceback.

Также можно использовать маркер parametrize на уровне класса или модуля (см. [How to mark test functions with attributes](https://docs.pytest.org/en/stable/how-to/mark.html#mark)), что приведёт к вызову нескольких функций с наборами аргументов, например:

```
import pytest


@pytest.mark.parametrize("n,expected", [(1, 2), (3, 4)])
class TestClass:
    def test_simple_case(self, n, expected):
        assert n + 1 == expected

    def test_weird_simple_case(self, n, expected):
        assert (n * 1) + 1 == expected

```

Чтобы параметризовать все тесты в модуле, можно присвоить значение глобальной переменной [pytestmark](https://docs.pytest.org/en/stable/reference/reference.html#globalvar-pytestmark):

```
import pytest

pytestmark = pytest.mark.parametrize("n,expected", [(1, 2), (3, 4)])


class TestClass:
    def test_simple_case(self, n, expected):
        assert n + 1 == expected

    def test_weird_simple_case(self, n, expected):
        assert (n * 1) + 1 == expected

```

Также можно помечать отдельные экземпляры тестов внутри параметризации, например встроенным `mark.xfail`:

```
# content of test_expectation.py
import pytest


@pytest.mark.parametrize(
    "test_input,expected",
    [("3+5", 8), ("2+4", 6), pytest.param("6*9", 42, marks=pytest.mark.xfail)],
)
def test_eval(test_input, expected):
    assert eval(test_input) == expected

```

Запустим:

```
$ pytest
=========================== test session starts ============================
platform linux -- Python 3.x.y, pytest-9.x.y, pluggy-1.x.y
rootdir: /home/sweet/project
collected 3 items

test_expectation.py ..x                                              [100%]

======================= 2 passed, 1 xfailed in 0.12s =======================

```

Набор параметров, который раньше вызывал падение, теперь отображается как “xfailed” (ожидаемо падающий) тест.

Если значения, переданные в `parametrize`, дают пустой список — например, если они генерируются динамически функцией — поведение pytest определяется опцией [empty_parameter_set_mark](https://docs.pytest.org/en/stable/reference/reference.html#confval-empty_parameter_set_mark).

Чтобы получить все комбинации нескольких параметризованных аргументов, можно «стекать» декораторы `parametrize`:

```
import pytest


@pytest.mark.parametrize("x", [0, 1])
@pytest.mark.parametrize("y", [2, 3])
def test_foo(x, y):
    pass

```

Это запустит тест с аргументами `x=0/y=2`, `x=1/y=2`, `x=0/y=3` и `x=1/y=3`, исчерпывая параметры в порядке декораторов.

## Базовый пример pytest_generate_tests

Иногда нужно реализовать свою схему параметризации или добавить динамику в определение параметров или области видимости фикстуры. Для этого можно использовать хук `pytest_generate_tests`, который вызывается при сборе (collection) тестовой функции. Через переданный объект `metafunc` можно исследовать контекст теста и, главное, вызвать `metafunc.parametrize()`, чтобы включить параметризацию.

Например, допустим, мы хотим запускать тест, который принимает строковые входы, задаваемые через новую опцию командной строки `pytest`. Сначала напишем простой тест, принимающий аргумент фикстуры `stringinput`:

```
# content of test_strings.py


def test_valid_string(stringinput):
    assert stringinput.isalpha()

```

Теперь добавим файл `conftest.py`, содержащий добавление опции командной строки и параметризацию нашей тестовой функции:

```
# content of conftest.py


def pytest_addoption(parser):
    parser.addoption(
        "--stringinput",
        action="append",
        default=[],
        help="list of stringinputs to pass to test functions",
    )


def pytest_generate_tests(metafunc):
    if "stringinput" in metafunc.fixturenames:
        metafunc.parametrize("stringinput", metafunc.config.getoption("stringinput"))

```

Note

Хук [pytest_generate_tests](https://docs.pytest.org/en/stable/reference/reference.html#std-hook-pytest_generate_tests) можно реализовать непосредственно в тестовом модуле или внутри тестового класса — в отличие от других хуков, pytest обнаружит его и там. Другие хуки должны находиться в [conftest.py](https://docs.pytest.org/en/stable/how-to/writing_plugins.html#localplugin) или плагине. См. [Writing hook functions](https://docs.pytest.org/en/stable/how-to/writing_hook_functions.html#writinghooks).

Если теперь передать два значения stringinput, тест выполнится дважды:

```
$ pytest -q --stringinput="hello" --stringinput="world" test_strings.py
..                                                                   [100%]
2 passed in 0.12s

```

Запустим также со строкой, которая приведёт к падению:

```
$ pytest -q --stringinput="!" test_strings.py
F                                                                    [100%]
================================= FAILURES =================================
___________________________ test_valid_string[!] ___________________________

stringinput = '!'

    def test_valid_string(stringinput):
>       assert stringinput.isalpha()
E       AssertionError: assert False
E        +  where False = <built-in method isalpha of str object at 0xdeadbeef0001>()
E        +    where <built-in method isalpha of str object at 0xdeadbeef0001> = '!'.isalpha

test_strings.py:4: AssertionError
========================= short test summary info ==========================
FAILED test_strings.py::test_valid_string[!] - AssertionError: assert False
1 failed in 0.12s

```

Как и ожидалось, тест падает.

Если не указать stringinput, тест будет пропущен, потому что `metafunc.parametrize()` будет вызван с пустым списком параметров:

```
$ pytest -q -rs test_strings.py
s                                                                    [100%]
========================= short test summary info ==========================
SKIPPED [1] test_strings.py: got empty parameter set for (stringinput)
1 skipped in 0.12s

```

Обратите внимание: при многократном вызове `metafunc.parametrize` с разными наборами параметров имена параметров не могут дублироваться между наборами, иначе будет выброшена ошибка.

## Больше примеров

Дополнительные примеры см. в [more parametrization examples](https://docs.pytest.org/en/stable/example/parametrize.html#paramexamples).