# Как запускать pytest

См. также

[Полный справочник флагов командной строки pytest](https://docs.pytest.org/en/stable/reference/reference.html#command-line-flags)

В общем случае pytest запускается командой `pytest` (см. ниже другие способы запуска pytest). Она выполнит все тесты во всех файлах, чьи имена соответствуют шаблону `test_*.py` или `*_test.py` в текущем каталоге и его подкаталогах. В более общем виде pytest следует [стандартным правилам обнаружения тестов](https://docs.pytest.org/en/stable/explanation/goodpractices.html#test-discovery).

## Указание, какие тесты запускать

Pytest поддерживает несколько способов запускать и выбирать тесты из командной строки или из файла (см. ниже чтение аргументов из файла).

Запуск тестов в модуле

```
pytest test_mod.py

```

Запуск тестов в каталоге

```
pytest testing/

```

Запуск тестов по ключевому выражению

```
pytest -k 'MyClass and not method'

```

Это запустит тесты, имена которых соответствуют заданному строковому выражению (без учёта регистра), в котором можно использовать операторы Python, рассматривая имена файлов, классов и функций как переменные. Пример выше запустит `TestMyClass.test_something`, но не `TestMyClass.test_method_simple`. В Windows используйте `""` вместо `''` в выражении.

Запуск тестов по аргументам коллекции

Передайте имя файла модуля относительно рабочей директории, затем уточнения вроде имени класса и имени функции, разделённые `::`, а параметры параметризации — в `[]`.

Чтобы запустить конкретный тест в модуле:

```
pytest tests/test_mod.py::test_func

```

Чтобы запустить все тесты в классе:

```
pytest tests/test_mod.py::TestClass

```

Указать конкретный метод теста:

```
pytest tests/test_mod.py::TestClass::test_method

```

Указать конкретную параметризацию теста:

```
pytest tests/test_mod.py::test_func[x1,y2]

```

Запуск тестов по выражению маркеров (marker expressions)

Чтобы запустить все тесты, декорированные `@pytest.mark.slow`:

```
pytest -m slow

```

Чтобы запустить все тесты, декорированные аннотированным `@pytest.mark.slow(phase=1)`, где keyword-аргумент `phase` равен `1`:

```
pytest -m "slow(phase=1)"

```

Подробнее см. [marks](https://docs.pytest.org/en/stable/how-to/mark.html#mark).

Запуск тестов из пакетов

```
pytest --pyargs pkg.testing

```

Это импортирует `pkg.testing` и использует его расположение в файловой системе, чтобы найти и запустить тесты.

Чтение аргументов из файла

Добавлено в версии 8.2.

Всё перечисленное выше можно прочитать из файла, используя префикс `@`:

```
pytest @tests_to_run.txt

```

где `tests_to_run.txt` содержит по одной записи на строку, например:

```
tests/test_file.py
tests/test_mod.py::test_func[x1,y2]
tests/test_mod.py::TestClass
-m slow

```

Этот файл можно также сгенерировать через `pytest --collect-only -q` и модифицировать при необходимости.

## Получение справки о версии, именах опций, переменных окружения

```
pytest --version   # shows where pytest was imported from
pytest --fixtures  # show available builtin function arguments
pytest -h | --help # show help on command line and config file options

```

## Профилирование длительности выполнения тестов

Изменено в версии 6.0.

Чтобы получить список 10 самых долгих тестов с длительностью более 1.0s:

```
pytest --durations=10 --durations-min=1.0

```

По умолчанию pytest не показывает длительности тестов, которые слишком малы (<0.005s), если не передать `-vv` в командной строке.

## Управление загрузкой плагинов

### Ранняя загрузка плагинов

Можно явно «раньше времени» загрузить плагины (встроенные и внешние) в командной строке опцией [-p](https://docs.pytest.org/en/stable/reference/reference.html#cmdoption-p):

```
pytest -p mypluginmodule

```

Опция принимает параметр `name`, который может быть:

Полное точечное имя модуля, например `myproject.plugins`. Это имя должно быть импортируемым.

Имя entry-point плагина. Это имя, которое передаётся в `importlib` при регистрации плагина. Например, чтобы рано загрузить плагин [pytest-cov](https://pypi.org/project/pytest-cov), можно использовать:

```
pytest -p pytest_cov

```

### Отключение плагинов

Чтобы отключить загрузку конкретных плагинов при запуске, используйте опцию [-p](https://docs.pytest.org/en/stable/reference/reference.html#cmdoption-p) вместе с префиксом `no:`.

Пример: чтобы отключить загрузку плагина `doctest`, который отвечает за выполнение doctest-тестов из текстовых файлов, запустите pytest так:

```
pytest -p no:doctest

```

## Другие способы запуска pytest

### Запуск pytest через python -m pytest

Можно запускать тестирование через интерпретатор Python из командной строки:

```
python -m pytest [...]

```

Это почти эквивалентно прямому запуску скрипта командной строки `pytest [...]`, за исключением того, что вызов через `python` также добавит текущий каталог в `sys.path`.

### Вызов pytest из Python-кода

Можно вызвать `pytest` напрямую из Python-кода:

```
retcode = pytest.main()

```

Это действует так, будто вы вызвали «pytest» из командной строки. Оно не выбрасывает [SystemExit](https://docs.python.org/3/library/exceptions.html#SystemExit), а вместо этого возвращает [код выхода](https://docs.pytest.org/en/stable/reference/exit-codes.html#exit-codes). Если аргументы не переданы, `main` прочитает их из аргументов командной строки процесса ([sys.argv](https://docs.python.org/3/library/sys.html#sys.argv)), что может быть нежелательно. Можно явно передать опции и аргументы:

```
retcode = pytest.main(["-x", "mytestdir"])

```

Можно указать дополнительные плагины для `pytest.main`:

```
# content of myinvoke.py
import sys

import pytest


class MyPlugin:
    def pytest_sessionfinish(self):
        print("*** test run reporting finishing")


if __name__ == "__main__":
    sys.exit(pytest.main(["-qq"], plugins=[MyPlugin()]))

```

Запуск покажет, что `MyPlugin` был добавлен и его хук был вызван:

```
$ python myinvoke.py
*** test run reporting finishing

```

Note

Вызов `pytest.main()` приведёт к импорту ваших тестов и любых модулей, которые они импортируют. Из‑за механизма кэширования системы импорта Python повторные вызовы `pytest.main()` в рамках одного процесса не отразят изменения в этих файлах между вызовами. Поэтому делать несколько вызовов `pytest.main()` из одного процесса (например, чтобы перезапускать тесты) не рекомендуется.