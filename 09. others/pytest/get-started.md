# Начало работы¶

## Установка pytest¶

Выполните следующую команду в командной строке:

```
pip install -U pytest

```

Проверьте, что установили правильную версию:

```
$ pytest --version
pytest 9.0.2

```

## Создайте свой первый тест¶

Создайте новый файл с именем `test_sample.py`, содержащий функцию и тест:

```
# content of test_sample.py
def func(x):
    return x + 1


def test_answer():
    assert func(3) == 5

```

Тест:

```
$ pytest
=========================== test session starts ============================
platform linux -- Python 3.x.y, pytest-9.x.y, pluggy-1.x.y
rootdir: /home/sweet/project
collected 1 item

test_sample.py F                                                     [100%]

================================= FAILURES =================================
_______________________________ test_answer ________________________________

    def test_answer():
>       assert func(3) == 5
E       assert 4 == 5
E        +  where 4 = func(3)

test_sample.py:6: AssertionError
========================= short test summary info ==========================
FAILED test_sample.py::test_answer - assert 4 == 5
============================ 1 failed in 0.12s =============================

```

`[100%]` относится к общему прогрессу выполнения всех тестов. После завершения pytest показывает отчёт об ошибке, потому что `func(3)` не возвращает `5`.

Note

Можно использовать инструкцию `assert`, чтобы проверять ожидания в тестах. [Продвинутая интроспекция assert-выражений](https://docs.python.org/3/reference/simple_stmts.html#assert) в pytest умно выводит промежуточные значения выражения assert, так что можно не запоминать множество названий [унаследованных от JUnit методов](https://docs.python.org/3/library/unittest.html#testcase-objects).

## Запуск нескольких тестов¶

`pytest` запускает все файлы вида `test_*.py` или `*_test.py` в текущем каталоге и его подкаталогах. В более общем виде он следует [стандартным правилам обнаружения тестов](https://docs.pytest.org/en/stable/explanation/goodpractices.html#test-discovery).

## Проверка, что выбрасывается определённое исключение¶

Используйте хелпер [raises](https://docs.pytest.org/en/stable/how-to/assert.html#assertraises), чтобы проверить, что некоторый код выбрасывает исключение:

```
# content of test_sysexit.py
import pytest


def f():
    raise SystemExit(1)


def test_mytest():
    with pytest.raises(SystemExit):
        f()

```

Выполните тестовую функцию в «тихом» режиме отчётов:

```
$ pytest -q test_sysexit.py
.                                                                    [100%]
1 passed in 0.12s

```

Note

Флаг `-q/--quiet` делает вывод кратким в этом и следующих примерах.

См. [Assertions about approximate equality](https://docs.pytest.org/en/stable/how-to/assert.html#assertraises), чтобы задать больше деталей об ожидаемом исключении.

## Группировка нескольких тестов в классе¶

Когда тестов становится больше, может захотеться сгруппировать их в классе. pytest позволяет легко создать класс, содержащий более одного теста:

```
# content of test_class.py
class TestClass:
    def test_one(self):
        x = "this"
        assert "h" in x

    def test_two(self):
        x = "hello"
        assert hasattr(x, "check")

```

`pytest` обнаруживает все тесты согласно своим [соглашениям обнаружения тестов Python](https://docs.pytest.org/en/stable/explanation/goodpractices.html#test-discovery), поэтому он найдёт обе функции, начинающиеся с `test_`. Ничего наследовать не нужно, но убедитесь, что имя класса начинается с `Test`, иначе класс будет пропущен. Можно просто запустить модуль, передав его имя файла:

```
$ pytest -q test_class.py
.F                                                                   [100%]
================================= FAILURES =================================
____________________________ TestClass.test_two ____________________________

self = <test_class.TestClass object at 0xdeadbeef0001>

    def test_two(self):
        x = "hello"
>       assert hasattr(x, "check")
E       AssertionError: assert False
E        +  where False = hasattr('hello', 'check')

test_class.py:8: AssertionError
========================= short test summary info ==========================
FAILED test_class.py::TestClass::test_two - AssertionError: assert False
1 failed, 1 passed in 0.12s

```

Первый тест прошёл, второй упал. Промежуточные значения в assert помогают понять причину падения.

Группировка тестов в классах может быть полезна по следующим причинам:

Организация тестов

Общие фикстуры для тестов только в этом классе

Применение меток на уровне класса с неявным применением ко всем тестам

При группировке тестов в классе стоит помнить, что каждый тест получает уникальный экземпляр класса. Если бы каждый тест делил один и тот же экземпляр класса, это было бы вредно для изоляции тестов и поощряло бы плохие практики. Это показано ниже:

```
# content of test_class_demo.py
class TestClassDemoInstance:
    value = 0

    def test_one(self):
        self.value = 1
        assert self.value == 1

    def test_two(self):
        assert self.value == 1

```

```
$ pytest -k TestClassDemoInstance -q
.F                                                                   [100%]
================================= FAILURES =================================
______________________ TestClassDemoInstance.test_two ______________________

self = <test_class_demo.TestClassDemoInstance object at 0xdeadbeef0002>

    def test_two(self):
>       assert self.value == 1
E       assert 0 == 1
E        +  where 0 = <test_class_demo.TestClassDemoInstance object at 0xdeadbeef0002>.value

test_class_demo.py:9: AssertionError
========================= short test summary info ==========================
FAILED test_class_demo.py::TestClassDemoInstance::test_two - assert 0 == 1
1 failed, 1 passed in 0.12s

```

Обратите внимание: атрибуты, добавленные на уровне класса, являются атрибутами класса, поэтому они будут разделяться между тестами.

## Сравнение чисел с плавающей точкой с pytest.approx¶

`pytest` также предоставляет набор утилит, упрощающих написание тестов. Например, можно использовать [pytest.approx()](https://docs.pytest.org/en/stable/reference/reference.html#pytest.approx) для сравнения чисел с плавающей точкой, где возможны небольшие ошибки округления:

```
# content of test_approx.py
import pytest


def test_sum():
    assert (0.1 + 0.2) == pytest.approx(0.3)

```

Это избавляет от ручных проверок допусков или использования `math.isclose` и работает со скалярами, списками и массивами NumPy.

## Запрос уникального временного каталога для функциональных тестов¶

`pytest` предоставляет [встроенные фикстуры/аргументы функций](https://docs.pytest.org/en/stable/builtin.html), чтобы запрашивать ресурсы, например уникальный временный каталог:

```
# content of test_tmp_path.py
def test_needsfiles(tmp_path):
    print(tmp_path)
    assert 0

```

Укажите имя `tmp_path` в сигнатуре тестовой функции — и `pytest` найдёт и вызовет фабрику фикстуры, чтобы создать ресурс перед вызовом теста. Перед запуском теста `pytest` создаёт уникальный (для данного вызова теста) временный каталог:

```
$ pytest -q test_tmp_path.py
F                                                                    [100%]
================================= FAILURES =================================
_____________________________ test_needsfiles ______________________________

tmp_path = PosixPath('PYTEST_TMPDIR/test_needsfiles0')

    def test_needsfiles(tmp_path):
        print(tmp_path)
>       assert 0
E       assert 0

test_tmp_path.py:3: AssertionError
--------------------------- Captured stdout call ---------------------------
PYTEST_TMPDIR/test_needsfiles0
========================= short test summary info ==========================
FAILED test_tmp_path.py::test_needsfiles - assert 0
1 failed in 0.12s

```

Подробнее о временных каталогах см. в [Temporary directories and files](https://docs.pytest.org/en/stable/how-to/tmp_path.html#tmp-path-handling).

Посмотреть, какие встроенные [pytest fixtures](https://docs.pytest.org/en/stable/reference/fixtures.html#fixtures) существуют, можно командой:

```
pytest --fixtures   # shows builtin and custom fixtures

```

Обратите внимание: команда не покажет фикстуры, начинающиеся с `_`, если не добавить опцию [-v](https://docs.pytest.org/en/stable/reference/reference.html#cmdoption-v).

## Продолжение¶

Дополнительные материалы pytest, которые помогут вам настроить тесты под ваш workflow:

“ [How to invoke pytest](https://docs.pytest.org/en/stable/how-to/usage.html#usage)” — примеры запуска из командной строки

“ [How to use pytest with an existing test suite](https://docs.pytest.org/en/stable/how-to/existingtestsuite.html#existingtestsuite)” — работа с уже существующим набором тестов

“ [How to mark test functions with attributes](https://docs.pytest.org/en/stable/how-to/mark.html#mark)” — механизм `pytest.mark`

“ [Fixtures reference](https://docs.pytest.org/en/stable/reference/fixtures.html#fixtures)” — справочник по фикстурам

“ [Writing plugins](https://docs.pytest.org/en/stable/how-to/writing_plugins.html#plugins)” — управление и написание плагинов

“ [Good Integration Practices](https://docs.pytest.org/en/stable/explanation/goodpractices.html#goodpractices)” — практики интеграции для virtualenv и структуры тестов