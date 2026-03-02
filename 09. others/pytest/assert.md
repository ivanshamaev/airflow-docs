# Как писать и отображать утверждения (assertions) в тестах

## Утверждения с помощью инструкции assert

`pytest` позволяет использовать стандартную Python-инструкцию `assert` для проверки ожиданий и значений в тестах на Python. Например, можно написать так:

```
# content of test_assert1.py
def f():
    return 3


def test_function():
    assert f() == 4

```

чтобы проверить, что функция возвращает определённое значение. Если утверждение упадёт, вы увидите возвращаемое значение вызова функции:

```
$ pytest test_assert1.py
=========================== test session starts ============================
platform linux -- Python 3.x.y, pytest-9.x.y, pluggy-1.x.y
rootdir: /home/sweet/project
collected 1 item

test_assert1.py F                                                    [100%]

================================= FAILURES =================================
______________________________ test_function _______________________________

    def test_function():
>       assert f() == 4
E       assert 3 == 4
E        +  where 3 = f()

test_assert1.py:6: AssertionError
========================= short test summary info ==========================
FAILED test_assert1.py::test_function - assert 3 == 4
============================ 1 failed in 0.12s =============================

```

`pytest` умеет показывать значения наиболее распространённых подвыражений, включая вызовы, атрибуты, сравнения, бинарные и унарные операторы. (См. [Demo of Python failure reports with pytest](https://docs.pytest.org/en/stable/example/reportingdemo.html#tbreportdemo).) Это позволяет использовать идиоматические конструкции Python без «обвязки», не теряя информацию для интроспекции.

Если в утверждении задано сообщение, например так:

```
assert a % 2 == 0, "value was odd, should be even"

```

то оно печатается вместе с результатом интроспекции утверждения в traceback.

Подробнее об интроспекции утверждений см. в разделе Assertion introspection details.

## Утверждения о приблизительном равенстве

При сравнении чисел с плавающей точкой (или массивов float) небольшие ошибки округления — обычное дело. Вместо `assert abs(a - b) < tol` или `numpy.isclose` можно использовать [pytest.approx()](https://docs.pytest.org/en/stable/reference/reference.html#pytest.approx):

```
import pytest
import numpy as np


def test_floats():
    assert (0.1 + 0.2) == pytest.approx(0.3)


def test_arrays():
    a = np.array([1.0, 2.0, 3.0])
    b = np.array([0.9999, 2.0001, 3.0])
    assert a == pytest.approx(b)

```

`pytest.approx` работает со скалярами, списками, словарями и массивами NumPy. Также поддерживаются сравнения с NaN.

Подробности см. в [pytest.approx()](https://docs.pytest.org/en/stable/reference/reference.html#pytest.approx).

## Утверждения об ожидаемых исключениях

Чтобы писать утверждения о выброшенных исключениях, можно использовать [pytest.raises()](https://docs.pytest.org/en/stable/reference/reference.html#pytest.raises) как контекстный менеджер:

```
import pytest


def test_zero_division():
    with pytest.raises(ZeroDivisionError):
        1 / 0

```

а если нужен доступ к информации о фактическом исключении, можно сделать так:

```
def test_recursion_depth():
    with pytest.raises(RuntimeError) as excinfo:

        def f():
            f()

        f()
    assert "maximum recursion" in str(excinfo.value)

```

`excinfo` — это экземпляр [ExceptionInfo](https://docs.pytest.org/en/stable/reference/reference.html#pytest.ExceptionInfo), который является обёрткой над реально выброшенным исключением. Основные интересующие атрибуты: `.type`, `.value` и `.traceback`.

Обратите внимание: `pytest.raises` сопоставляет тип исключения или любой его подкласс (как и стандартный `except`). Если нужно проверить, что блок кода выбрасывает исключение строго определённого типа, это нужно проверить явно:

```
def test_foo_not_implemented():
    def foo():
        raise NotImplementedError

    with pytest.raises(RuntimeError) as excinfo:
        foo()
    assert excinfo.type is RuntimeError

```

Вызов [pytest.raises()](https://docs.pytest.org/en/stable/reference/reference.html#pytest.raises) пройдёт успешно, хотя функция выбрасывает [NotImplementedError](https://docs.python.org/3/library/exceptions.html#NotImplementedError), потому что [NotImplementedError](https://docs.python.org/3/library/exceptions.html#NotImplementedError) — подкласс [RuntimeError](https://docs.python.org/3/library/exceptions.html#RuntimeError); однако следующий `assert` поймает проблему.

### Сопоставление сообщений исключений

Можно передать keyword-параметр `match` в контекстный менеджер, чтобы проверить, что регулярное выражение совпадает со строковым представлением исключения (аналогично `TestCase.assertRaisesRegex` из `unittest`):

```
import pytest


def myfunc():
    raise ValueError("Exception 123 raised")


def test_match():
    with pytest.raises(ValueError, match=r".* 123 .*"):
        myfunc()

```

Notes:

Параметр `match` сопоставляется функцией [re.search()](https://docs.python.org/3/library/re.html#re.search), поэтому в примере выше сработало бы и `match='123'`.

Параметр `match` также сопоставляется с [PEP-678](https://peps.python.org/pep-0678/) `__notes__`.

### Утверждения об ожидаемых группах исключений

Если ожидается [BaseExceptionGroup](https://docs.python.org/3/library/exceptions.html#BaseExceptionGroup) или [ExceptionGroup](https://docs.python.org/3/library/exceptions.html#ExceptionGroup), можно использовать [pytest.RaisesGroup](https://docs.pytest.org/en/stable/reference/reference.html#pytest.RaisesGroup):

```
def test_exception_in_group():
    with pytest.RaisesGroup(ValueError):
        raise ExceptionGroup("group msg", [ValueError("value msg")])
    with pytest.RaisesGroup(ValueError, TypeError):
        raise ExceptionGroup("msg", [ValueError("foo"), TypeError("bar")])

```

Он принимает параметр `match`, который проверяет сообщение группы, и параметр `check`, который принимает произвольный callable: ему передаётся группа, и контекст проходит успешно только если callable возвращает `True`.

```
def test_raisesgroup_match_and_check():
    with pytest.RaisesGroup(BaseException, match="my group msg"):
        raise BaseExceptionGroup("my group msg", [KeyboardInterrupt()])
    with pytest.RaisesGroup(
        Exception, check=lambda eg: isinstance(eg.__cause__, ValueError)
    ):
        raise ExceptionGroup("", [TypeError()]) from ValueError()

```

Он строг к структуре и «неразвёрнутым» (unwrapped) исключениям, в отличие от [except*](https://docs.python.org/3/reference/compound_stmts.html#except-star), поэтому, возможно, вы захотите задать параметры `flatten_subgroups` и/или `allow_unwrapped`.

```
def test_structure():
    with pytest.RaisesGroup(pytest.RaisesGroup(ValueError)):
        raise ExceptionGroup("", (ExceptionGroup("", (ValueError(),)),))
    with pytest.RaisesGroup(ValueError, flatten_subgroups=True):
        raise ExceptionGroup("1st group", [ExceptionGroup("2nd group", [ValueError()])])
    with pytest.RaisesGroup(ValueError, allow_unwrapped=True):
        raise ValueError

```

Чтобы задать больше деталей о содержащемся исключении, можно использовать [pytest.RaisesExc](https://docs.pytest.org/en/stable/reference/reference.html#pytest.RaisesExc)

```
def test_raises_exc():
    with pytest.RaisesGroup(pytest.RaisesExc(ValueError, match="foo")):
        raise ExceptionGroup("", (ValueError("foo")))

```

Оба предоставляют методы [pytest.RaisesGroup.matches()](https://docs.pytest.org/en/stable/reference/reference.html#pytest.RaisesGroup.matches) и [pytest.RaisesExc.matches()](https://docs.pytest.org/en/stable/reference/reference.html#pytest.RaisesExc.matches), если вы хотите делать сопоставление вне контекстного менеджера. Это может быть полезно при проверке `.__context__` или `.__cause__`.

```
def test_matches():
    exc = ValueError()
    exc_group = ExceptionGroup("", [exc])
    if RaisesGroup(ValueError).matches(exc_group):
        ...
    # helpful error is available in `.fail_reason` if it fails to match
    r = RaisesExc(ValueError)
    assert r.matches(e), r.fail_reason

```

Подробности и примеры см. в документации по [pytest.RaisesGroup](https://docs.pytest.org/en/stable/reference/reference.html#pytest.RaisesGroup) и [pytest.RaisesExc](https://docs.pytest.org/en/stable/reference/reference.html#pytest.RaisesExc).

### ExceptionInfo.group_contains()

Warning

Этот хелпер облегчает проверку наличия конкретных исключений, но он очень плох для проверки того, что группа не содержит никаких других исключений. Поэтому это пройдёт:

```
class EXTREMELYBADERROR(BaseException):
    """This is a very bad error to miss"""


def test_for_value_error():
    with pytest.raises(ExceptionGroup) as excinfo:
        excs = [ValueError()]
        if very_unlucky():
            excs.append(EXTREMELYBADERROR())
        raise ExceptionGroup("", excs)
    # This passes regardless of if there's other exceptions.
    assert excinfo.group_contains(ValueError)
    # You can't simply list all exceptions you *don't* want to get here.

```

Нет хорошего способа использовать [excinfo.group_contains()](https://docs.pytest.org/en/stable/reference/reference.html#pytest.ExceptionInfo.group_contains), чтобы гарантировать, что вы не получаете никаких других исключений, кроме ожидаемого. Вместо этого используйте [pytest.RaisesGroup](https://docs.pytest.org/en/stable/reference/reference.html#pytest.RaisesGroup), см. Assertions about expected exception groups.

Также можно использовать метод [excinfo.group_contains()](https://docs.pytest.org/en/stable/reference/reference.html#pytest.ExceptionInfo.group_contains), чтобы проверять исключения, возвращаемые как часть [ExceptionGroup](https://docs.python.org/3/library/exceptions.html#ExceptionGroup):

```
def test_exception_in_group():
    with pytest.raises(ExceptionGroup) as excinfo:
        raise ExceptionGroup(
            "Group message",
            [
                RuntimeError("Exception 123 raised"),
            ],
        )
    assert excinfo.group_contains(RuntimeError, match=r".* 123 .*")
    assert not excinfo.group_contains(TypeError)

```

Необязательный keyword-параметр `match` работает так же, как и в [pytest.raises()](https://docs.pytest.org/en/stable/reference/reference.html#pytest.raises).

По умолчанию `group_contains()` рекурсивно ищет подходящее исключение на любом уровне вложенных `ExceptionGroup`. Можно задать keyword-параметр `depth`, если вы хотите совпадение только на определённом уровне; исключения, содержащиеся непосредственно в верхнем `ExceptionGroup`, соответствуют `depth=1`.

```
def test_exception_in_group_at_given_depth():
    with pytest.raises(ExceptionGroup) as excinfo:
        raise ExceptionGroup(
            "Group message",
            [
                RuntimeError(),
                ExceptionGroup(
                    "Nested group",
                    [
                        TypeError(),
                    ],
                ),
            ],
        )
    assert excinfo.group_contains(RuntimeError, depth=1)
    assert excinfo.group_contains(TypeError, depth=2)
    assert not excinfo.group_contains(RuntimeError, depth=2)
    assert not excinfo.group_contains(TypeError, depth=1)

```

### Альтернативная форма pytest.raises (legacy)

Есть альтернативная форма [pytest.raises()](https://docs.pytest.org/en/stable/reference/reference.html#pytest.raises), где вы передаёте функцию, которая будет выполнена, а также `*args` и `**kwargs`. Тогда [pytest.raises()](https://docs.pytest.org/en/stable/reference/reference.html#pytest.raises) выполнит функцию с этими аргументами и проверит, что было выброшено заданное исключение:

```
def func(x):
    if x <= 0:
        raise ValueError("x needs to be larger than zero")


pytest.raises(ValueError, func, x=-1)

```

Репортёр покажет полезный вывод в случае падений, например если исключение не было выброшено или выброшено не то исключение.

Эта форма была исходным API [pytest.raises()](https://docs.pytest.org/en/stable/reference/reference.html#pytest.raises), разработанным ещё до появления `with` в языке Python. Сейчас она используется редко: более читаемой считается форма контекстного менеджера (с `with`). Тем не менее эта форма полностью поддерживается и не является устаревшей (deprecated).

### Маркер xfail и pytest.raises

Также можно указать аргумент `raises` у [pytest.mark.xfail](https://docs.pytest.org/en/stable/reference/reference.html#pytest-mark-xfail-ref), который проверяет, что тест падает более специфичным образом, чем «просто выброшено любое исключение»:

```
def f():
    raise IndexError()


@pytest.mark.xfail(raises=IndexError)
def test_f():
    f()

```

Это приведёт к “xfail” только если тест падает, выбрасывая `IndexError` или подклассы.

Использование [pytest.mark.xfail](https://docs.pytest.org/en/stable/reference/reference.html#pytest-mark-xfail-ref) с параметром `raises` вероятно лучше подходит для документирования неисправленных багов (когда тест описывает, что «должно» происходить) или багов в зависимостях.

Использование [pytest.raises()](https://docs.pytest.org/en/stable/reference/reference.html#pytest.raises) вероятно лучше подходит для случаев, когда вы тестируете исключения, которые ваш собственный код выбрасывает намеренно — а это большинство случаев.

Можно также использовать [pytest.RaisesGroup](https://docs.pytest.org/en/stable/reference/reference.html#pytest.RaisesGroup):

```
def f():
    raise ExceptionGroup("", [IndexError()])


@pytest.mark.xfail(raises=RaisesGroup(IndexError))
def test_f():
    f()

```

## Утверждения об ожидаемых предупреждениях

Можно проверить, что код выбрасывает определённое предупреждение, используя [pytest.warns](https://docs.pytest.org/en/stable/how-to/capture-warnings.html#warns).

## Использование контекстно-зависимых сравнений

`pytest` богато поддерживает предоставление контекстно-зависимой информации, когда встречает сравнения. Например:

```
# content of test_assert2.py
def test_set_comparison():
    set1 = set("1308")
    set2 = set("8035")
    assert set1 == set2

```

если запустить этот модуль:

```
$ pytest test_assert2.py
=========================== test session starts ============================
platform linux -- Python 3.x.y, pytest-9.x.y, pluggy-1.x.y
rootdir: /home/sweet/project
collected 1 item

test_assert2.py F                                                    [100%]

================================= FAILURES =================================
___________________________ test_set_comparison ____________________________

    def test_set_comparison():
        set1 = set("1308")
        set2 = set("8035")
>       assert set1 == set2
E       AssertionError: assert {'0', '1', '3', '8'} == {'0', '3', '5', '8'}
E
E         Extra items in the left set:
E         '1'
E         Extra items in the right set:
E         '5'
E         Use -v to get more diff

test_assert2.py:4: AssertionError
========================= short test summary info ==========================
FAILED test_assert2.py::test_set_comparison - AssertionError: assert {'0'...
============================ 1 failed in 0.12s =============================

```

Специальные сравнения выполняются для ряда случаев:

сравнение длинных строк: показывается контекстный diff

сравнение длинных последовательностей: показываются первые индексы, на которых есть расхождения

сравнение словарей: показываются отличающиеся записи

См. [reporting demo](https://docs.pytest.org/en/stable/example/reportingdemo.html#tbreportdemo) для множества других примеров.

## Определение своего объяснения для упавших утверждений

Можно добавить собственные подробные объяснения, реализовав хук `pytest_assertrepr_compare`.

pytest_assertrepr_compare(config, op, left, right) [[source]](https://docs.pytest.org/en/stable/_modules/_pytest/hookspec.html#pytest_assertrepr_compare)

Вернуть объяснение для сравнений в упавших assert-выражениях.

Верните None, если пользовательского объяснения нет; иначе верните список строк. Строки будут соединены переводами строк, но любые переводы строк внутри отдельной строки будут экранированы. Обратите внимание: все строки, кроме первой, будут слегка с отступом — предполагается, что первая строка является кратким резюме.

Parameters:

config ([Config](https://docs.pytest.org/en/stable/reference/reference.html#pytest.Config)) – объект конфигурации pytest.

op ([str](https://docs.python.org/3/library/stdtypes.html#str)) – оператор, например `"=="`, `"!="`, `"not in"`.

left ([object](https://docs.python.org/3/library/functions.html#object)) – левый операнд.

right ([object](https://docs.python.org/3/library/functions.html#object)) – правый операнд.

### Использование в conftest-плагинах

Любой файл conftest может реализовать этот хук. Для конкретного item учитываются только файлы conftest в каталоге item и его родительских каталогах.

Например, рассмотрим добавление следующего хука в файл [conftest.py](https://docs.pytest.org/en/stable/reference/fixtures.html#conftest-py), который предоставляет альтернативное объяснение для объектов `Foo`:

```
# content of conftest.py
from test_foocompare import Foo


def pytest_assertrepr_compare(op, left, right):
    if isinstance(left, Foo) and isinstance(right, Foo) and op == "==":
        return [
            "Comparing Foo instances:",
            f"   vals: {left.val} != {right.val}",
        ]

```

теперь, имея такой тестовый модуль:

```
# content of test_foocompare.py
class Foo:
    def __init__(self, val):
        self.val = val

    def __eq__(self, other):
        return self.val == other.val


def test_compare():
    f1 = Foo(1)
    f2 = Foo(2)
    assert f1 == f2

```

можно запустить модуль и получить пользовательский вывод, определённый в conftest:

```
$ pytest -q test_foocompare.py
F                                                                    [100%]
================================= FAILURES =================================
_______________________________ test_compare _______________________________

    def test_compare():
        f1 = Foo(1)
        f2 = Foo(2)
>       assert f1 == f2
E       assert Comparing Foo instances:
E            vals: 1 != 2

test_foocompare.py:12: AssertionError
========================= short test summary info ==========================
FAILED test_foocompare.py::test_compare - assert Comparing Foo instances:
1 failed in 0.12s

```

## Возврат значения, отличного от None, из тестовых функций

Предупреждение [pytest.PytestReturnNotNoneWarning](https://docs.pytest.org/en/stable/reference/reference.html#pytest.PytestReturnNotNoneWarning) выдаётся, когда тестовая функция возвращает значение, отличное от `None`.

Это помогает предотвратить распространённую ошибку новичков: они предполагают, что возвращаемое `bool` (например `True` или `False`) определит, пройдёт тест или упадёт.

Пример:

```
@pytest.mark.parametrize(
    ["a", "b", "result"],
    [
        [1, 2, 5],
        [2, 3, 8],
        [5, 3, 18],
    ],
)
def test_foo(a, b, result):
    return foo(a, b) == result  # Incorrect usage, do not do this.

```

Поскольку pytest игнорирует возвращаемые значения, может быть удивительно, что тест никогда не упадёт из‑за возвращаемого значения.

Правильное исправление — заменить `return` на `assert`:

```
@pytest.mark.parametrize(
    ["a", "b", "result"],
    [
        [1, 2, 5],
        [2, 3, 8],
        [5, 3, 18],
    ],
)
def test_foo(a, b, result):
    assert foo(a, b) == result

```

## Подробности об интроспекции утверждений

Подробности об упавшем утверждении формируются за счёт переписывания (rewriting) assert-инструкций до их выполнения. Переписанные assert-инструкции помещают информацию интроспекции в сообщение об ошибке утверждения. `pytest` переписывает только те тестовые модули, которые он напрямую обнаруживает в процессе collection, поэтому `assert` в вспомогательных модулях, которые сами по себе не являются тестовыми модулями, переписаны не будут.

Можно вручную включить переписывание утверждений для импортируемого модуля, вызвав [register_assert_rewrite](https://docs.pytest.org/en/stable/how-to/writing_plugins.html#assertion-rewriting) до его импорта (хорошее место для этого — корневой `conftest.py`).

Дополнительную информацию см. в статье Benjamin Peterson: [Behind the scenes of pytest’s new assertion rewriting](http://pybites.blogspot.com/2011/07/behind-scenes-of-pytests-new-assertion.html).

### Переписывание утверждений кэширует файлы на диск

`pytest` записывает переписанные модули на диск для кэширования. Это поведение можно отключить (например, чтобы не оставлять устаревшие `.pyc` в проектах, где часто перемещают файлы), добавив в начало `conftest.py`:

```
import sys

sys.dont_write_bytecode = True

```

Обратите внимание: вы всё равно получаете преимущества интроспекции утверждений; единственное изменение в том, что `.pyc` файлы не будут кэшироваться на диск.

Кроме того, переписывание молча пропустит кэширование, если не может записать новые `.pyc` файлы, например в файловой системе только для чтения или в zipfile.

### Отключение переписывания assert

`pytest` переписывает тестовые модули при импорте, используя import hook, чтобы писать новые `pyc` файлы. В большинстве случаев это работает прозрачно. Однако если вы сами работаете с механизмом импорта, этот hook может мешать.

Если это ваш случай, есть два варианта:

Отключить переписывание для конкретного модуля, добавив строку `PYTEST_DONT_REWRITE` в его docstring.

Отключить переписывание для всех модулей, используя [--assert=plain](https://docs.pytest.org/en/stable/reference/reference.html#cmdoption-assert).