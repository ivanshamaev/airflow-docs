# Справочник по фикстурам

См. также

[About fixtures](https://docs.pytest.org/en/stable/explanation/fixtures.html#about-fixtures)

См. также

[How to use fixtures](https://docs.pytest.org/en/stable/how-to/fixtures.html#how-to-fixtures)

## Встроенные фикстуры

[Фикстуры](https://docs.pytest.org/en/stable/reference/reference.html#fixtures-api) определяются с помощью декоратора [@pytest.fixture](https://docs.pytest.org/en/stable/reference/reference.html#pytest-fixture-api). В pytest есть несколько полезных встроенных фикстур:

[capfd](https://docs.pytest.org/en/stable/reference/reference.html#std-fixture-capfd)

Перехватывать вывод в файловые дескрипторы `1` и `2` в виде текста.

[capfdbinary](https://docs.pytest.org/en/stable/reference/reference.html#std-fixture-capfdbinary)

Перехватывать вывод в файловые дескрипторы `1` и `2` в виде байтов.

[caplog](https://docs.pytest.org/en/stable/reference/reference.html#std-fixture-caplog)

Управлять логированием и получать доступ к записям логов.

[capsys](https://docs.pytest.org/en/stable/reference/reference.html#std-fixture-capsys)

Перехватывать вывод в `sys.stdout` и `sys.stderr` в виде текста.

[capteesys](https://docs.pytest.org/en/stable/reference/reference.html#std-fixture-capteesys)

Перехватывать вывод так же, как [capsys](https://docs.pytest.org/en/stable/reference/reference.html#std-fixture-capsys), но также пропускать текст дальше в соответствии с [--capture](https://docs.pytest.org/en/stable/reference/reference.html#cmdoption-capture).

[capsysbinary](https://docs.pytest.org/en/stable/reference/reference.html#std-fixture-capsysbinary)

Перехватывать вывод в `sys.stdout` и `sys.stderr` в виде байтов.

[cache](https://docs.pytest.org/en/stable/reference/reference.html#std-fixture-cache)

Хранить и получать значения между запусками pytest.

[doctest_namespace](https://docs.pytest.org/en/stable/reference/reference.html#std-fixture-doctest_namespace)

Предоставить словарь, внедряемый в пространство имён doctest.

[monkeypatch](https://docs.pytest.org/en/stable/reference/reference.html#std-fixture-monkeypatch)

Временно изменять классы, функции, словари, `os.environ` и другие объекты.

[pytestconfig](https://docs.pytest.org/en/stable/reference/reference.html#std-fixture-pytestconfig)

Доступ к значениям конфигурации, pluginmanager и хукам плагинов.

[subtests](https://docs.pytest.org/en/stable/reference/reference.html#std-fixture-subtests)

Включать объявление subtests внутри тестовых функций.

[record_property](https://docs.pytest.org/en/stable/reference/reference.html#std-fixture-record_property)

Добавлять дополнительные свойства к тесту.

[record_testsuite_property](https://docs.pytest.org/en/stable/reference/reference.html#std-fixture-record_testsuite_property)

Добавлять дополнительные свойства к набору тестов.

[recwarn](https://docs.pytest.org/en/stable/reference/reference.html#std-fixture-recwarn)

Записывать предупреждения, создаваемые тестовыми функциями.

[request](https://docs.pytest.org/en/stable/reference/reference.html#std-fixture-request)

Предоставлять информацию о выполняющейся тестовой функции.

[testdir](https://docs.pytest.org/en/stable/reference/reference.html#std-fixture-testdir)

Предоставлять временный тестовый каталог, помогающий запускать и тестировать плагины pytest.

[tmp_path](https://docs.pytest.org/en/stable/reference/reference.html#std-fixture-tmp_path)

Предоставлять объект [pathlib.Path](https://docs.python.org/3/library/pathlib.html#pathlib.Path), указывающий на временный каталог, уникальный для каждой тестовой функции.

[tmp_path_factory](https://docs.pytest.org/en/stable/reference/reference.html#std-fixture-tmp_path_factory)

Создавать временные каталоги с областью `session` и возвращать объекты [pathlib.Path](https://docs.python.org/3/library/pathlib.html#pathlib.Path).

[tmpdir](https://docs.pytest.org/en/stable/reference/reference.html#std-fixture-tmpdir)

Предоставлять объект [py.path.local](https://py.readthedocs.io/en/latest/path.html) к временному каталогу, уникальному для каждой тестовой функции; заменена на [tmp_path](https://docs.pytest.org/en/stable/reference/reference.html#std-fixture-tmp_path).

[tmpdir_factory](https://docs.pytest.org/en/stable/reference/reference.html#std-fixture-tmpdir_factory)

Создавать временные каталоги с областью `session` и возвращать объекты `py.path.local`; заменена на [tmp_path_factory](https://docs.pytest.org/en/stable/reference/reference.html#std-fixture-tmp_path_factory).

## Доступность фикстур

Доступность фикстур определяется с точки зрения теста. Фикстура доступна для запроса тестами только в том случае, если они находятся в области видимости, в которой определена эта фикстура. Если фикстура определена внутри класса, её могут запрашивать только тесты внутри этого класса. Но если фикстура определена в глобальной области видимости модуля, то любой тест в этом модуле, даже если он определён внутри класса, может запрашивать её.

Аналогично, тест может быть затронут autouse-фикстурой только в том случае, если тест находится в той же области видимости, в которой определена эта autouse-фикстура (см. Autouse fixtures are executed first within their scope).

Фикстура также может запрашивать любую другую фикстуру, независимо от того, где она определена, пока тест, который их запрашивает, «видит» все вовлечённые фикстуры.

Например, вот тестовый файл с фикстурой (`outer`), которая запрашивает фикстуру (`inner`) из области видимости, где она не была определена:

```
from __future__ import annotations

import pytest


@pytest.fixture
def order():
    return []


@pytest.fixture
def outer(order, inner):
    order.append("outer")


class TestOne:
    @pytest.fixture
    def inner(self, order):
        order.append("one")

    def test_order(self, order, outer):
        assert order == ["one", "outer"]


class TestTwo:
    @pytest.fixture
    def inner(self, order):
        order.append("two")

    def test_order(self, order, outer):
        assert order == ["two", "outer"]

```

С точки зрения тестов у них нет проблем с тем, чтобы «увидеть» каждую из фикстур, от которых они зависят:

Поэтому при выполнении `outer` без проблем найдёт `inner`, потому что pytest ищет фикстуры, начиная с точки зрения тестов.

Note

Область, в которой определена фикстура, никак не влияет на порядок её создания: порядок определяется логикой, описанной здесь.

### conftest.py: совместное использование фикстур в нескольких файлах

Файл `conftest.py` служит средством предоставления фикстур для всего каталога. Фикстуры, определённые в `conftest.py`, могут использоваться любым тестом в этом пакете без необходимости их импортировать (pytest обнаружит их автоматически).

Можно иметь несколько вложенных каталогов/пакетов с тестами, и в каждом каталоге может быть свой `conftest.py` со своими фикстурами, которые дополняют фикстуры, предоставленные `conftest.py` в родительских каталогах.

Например, при следующей структуре файлов с тестами:

```
tests/
    __init__.py

    conftest.py
        # content of tests/conftest.py
        import pytest

        @pytest.fixture
        def order():
            return []

        @pytest.fixture
        def top(order, innermost):
            order.append("top")

    test_top.py
        # content of tests/test_top.py
        import pytest

        @pytest.fixture
        def innermost(order):
            order.append("innermost top")

        def test_order(order, top):
            assert order == ["innermost top", "top"]

    subpackage/
        __init__.py

        conftest.py
            # content of tests/subpackage/conftest.py
            import pytest

            @pytest.fixture
            def mid(order):
                order.append("mid subpackage")

        test_subpackage.py
            # content of tests/subpackage/test_subpackage.py
            import pytest

            @pytest.fixture
            def innermost(order, mid):
                order.append("innermost subpackage")

            def test_order(order, top):
                assert order == ["mid subpackage", "innermost subpackage", "top"]

```

Границы областей видимости можно визуализировать так:

Каталоги становятся своего рода собственной областью, в которой фикстуры, определённые в `conftest.py` этого каталога, становятся доступными для всей этой области.

Тестам разрешено искать фикстуры «вверх» (выходя за пределы круга), но никогда нельзя идти «вниз» (заходить внутрь круга), чтобы продолжить поиск. Поэтому `tests/subpackage/test_subpackage.py::test_order` сможет найти фикстуру `innermost`, определённую в `tests/subpackage/test_subpackage.py`, но фикстура, определённая в `tests/test_top.py`, будет для него недоступна, потому что для её поиска пришлось бы спуститься на уровень ниже (зайти внутрь круга).

Первая найденная тестом фикстура и будет использована, поэтому при необходимости [фикстуры можно переопределять](https://docs.pytest.org/en/stable/how-to/fixtures.html#override-fixtures), если вы хотите изменить или расширить поведение фикстуры для конкретной области.

Также файл `conftest.py` можно использовать для реализации [локальных плагинов на уровне каталога](https://docs.pytest.org/en/stable/how-to/writing_plugins.html#conftest-py-plugins).

### Фикстуры из сторонних плагинов

Фикстуры не обязательно должны быть определены в этой структуре, чтобы быть доступными для тестов. Они также могут предоставляться сторонними плагинами, которые установлены в окружении — так работают многие плагины pytest. Пока такие плагины установлены, предоставляемые ими фикстуры могут запрашиваться из любого места в наборе тестов.

Поскольку они приходят извне структуры вашего набора тестов, сторонние плагины не дают собственной области видимости так же, как это делают файлы `conftest.py` и каталоги вашего набора тестов. В результате pytest будет искать фикстуры, «выходя» наружу через области так, как описано выше, и только в последнюю очередь будет достигать фикстур, определённых в плагинах.

Например, при следующей структуре файлов:

```
tests/
    __init__.py

    conftest.py
        # content of tests/conftest.py
        import pytest

        @pytest.fixture
        def order():
            return []

    subpackage/
        __init__.py

        conftest.py
            # content of tests/subpackage/conftest.py
            import pytest

            @pytest.fixture(autouse=True)
            def mid(order, b_fix):
                order.append("mid subpackage")

        test_subpackage.py
            # content of tests/subpackage/test_subpackage.py
            import pytest

            @pytest.fixture
            def inner(order, mid, a_fix):
                order.append("inner subpackage")

            def test_order(order, inner):
                assert order == ["b_fix", "mid subpackage", "a_fix", "inner subpackage"]

```

Если установлен `plugin_a`, предоставляющий фикстуру `a_fix`, и установлен `plugin_b`, предоставляющий фикстуру `b_fix`, то поиск фикстур для теста будет выглядеть так:

pytest будет искать `a_fix` и `b_fix` в плагинах только после того, как сначала поищет их во всех областях внутри `tests/`.

## Порядок создания (instantiation) фикстур

Когда pytest собирается выполнить тест и уже знает, какие фикстуры будут выполнены, ему нужно определить порядок их выполнения. Для этого он учитывает три фактора:

scope

dependencies

autouse

Имена фикстур или тестов, место их определения, порядок, в котором они определены, и порядок, в котором запрашиваются фикстуры, не влияют на порядок выполнения, кроме как по совпадению. Хотя pytest старается сделать так, чтобы такие совпадения оставались консистентными от запуска к запуску, полагаться на это нельзя. Если вы хотите контролировать порядок, безопаснее всего полагаться на эти три вещи и чётко задавать зависимости.

### Фикстуры с более широкой областью видимости выполняются первыми

В рамках одного запроса фикстур для теста фикстуры с более широкой областью (`session` и т. п.) выполняются раньше фикстур с более узкой областью (`function` или `class` и т. п.).

Пример:

```
from __future__ import annotations

import pytest


@pytest.fixture(scope="session")
def order():
    return []


@pytest.fixture
def func(order):
    order.append("function")


@pytest.fixture(scope="class")
def cls(order):
    order.append("class")


@pytest.fixture(scope="module")
def mod(order):
    order.append("module")


@pytest.fixture(scope="package")
def pack(order):
    order.append("package")


@pytest.fixture(scope="session")
def sess(order):
    order.append("session")


class TestClass:
    def test_order(self, func, cls, mod, pack, sess, order):
        assert order == ["session", "package", "module", "class", "function"]

```

Тест пройдёт, потому что фикстуры с более широкой областью выполняются первыми.

Разбор порядка:

### Фикстуры с одинаковой областью выполняются на основе зависимостей

Когда фикстура запрашивает другую фикстуру, другая выполняется первой. То есть, если фикстура `a` запрашивает фикстуру `b`, то `b` выполнится первой, потому что `a` зависит от `b` и не может работать без неё. Даже если `a` не использует результат `b`, она всё равно может запросить `b`, если нужно гарантировать, что она выполнится после `b`.

Например:

```
from __future__ import annotations

import pytest


@pytest.fixture
def order():
    return []


@pytest.fixture
def a(order):
    order.append("a")


@pytest.fixture
def b(a, order):
    order.append("b")


@pytest.fixture
def c(b, order):
    order.append("c")


@pytest.fixture
def d(c, b, order):
    order.append("d")


@pytest.fixture
def e(d, b, order):
    order.append("e")


@pytest.fixture
def f(e, order):
    order.append("f")


@pytest.fixture
def g(f, c, order):
    order.append("g")


def test_order(g, order):
    assert order == ["a", "b", "c", "d", "e", "f", "g"]

```

Если изобразить зависимости в виде графа, получится примерно так:

Правила, задаваемые каждой фикстурой (какие фикстуры должны выполняться до неё), достаточно полны, чтобы можно было «сплющить» граф до следующего вида:

Должно быть предоставлено достаточно информации через эти запросы, чтобы pytest мог получить чёткую линейную цепочку зависимостей и, как следствие, порядок операций для данного теста. Если есть неоднозначность и порядок можно интерпретировать более чем одним способом, нужно считать, что pytest может выбрать любую из этих интерпретаций в любой момент.

Например, если бы `d` не запрашивала `c` (т. е. граф выглядел бы так):

Поскольку `c` запрашивается только `g`, а `g` также запрашивает `f`, теперь неясно, должна ли `c` идти до/после `f`, `e` или `d`. Единственные «правила», заданные для `c`, — что она должна выполниться после `b` и до `g`.

pytest не знает, где именно должна быть `c` в этом случае, поэтому нужно считать, что она может находиться где угодно между `g` и `b`.

Это не обязательно плохо, но об этом стоит помнить. Если порядок выполнения может повлиять на поведение, которое тест проверяет, или как‑то иначе влияет на результат теста, порядок должен быть задан явно так, чтобы pytest мог линейно упорядочить («сплющить») этот порядок.

### Autouse-фикстуры выполняются первыми в своей области

Autouse-фикстуры считаются применимыми ко всем тестам, которые могут на них сослаться, поэтому они выполняются раньше других фикстур в своей области. Фикстуры, которые запрашиваются autouse-фикстурами, по сути становятся autouse-фикстурами для тех тестов, к которым применяется настоящая autouse-фикстура.

То есть если фикстура `a` — autouse, а фикстура `b` — нет, но `a` запрашивает `b`, то `b` фактически тоже становится autouse-фикстурой, но только для тех тестов, к которым применяется `a`.

В предыдущем примере граф становился неоднозначным, если `d` не запрашивала `c`. Но если бы `c` была autouse, то `b` и `a` по сути тоже стали бы autouse-фикстурами, поскольку `c` от них зависит. В результате они все были бы подняты выше невиртуальных (не-autouse) фикстур в этой области.

Так что если файл тестов выглядел бы так:

```
from __future__ import annotations

import pytest


@pytest.fixture
def order():
    return []


@pytest.fixture
def a(order):
    order.append("a")


@pytest.fixture
def b(a, order):
    order.append("b")


@pytest.fixture(autouse=True)
def c(b, order):
    order.append("c")


@pytest.fixture
def d(b, order):
    order.append("d")


@pytest.fixture
def e(d, order):
    order.append("e")


@pytest.fixture
def f(e, order):
    order.append("f")


@pytest.fixture
def g(f, c, order):
    order.append("g")


def test_order_and_g(g, order):
    assert order == ["a", "b", "c", "d", "e", "f", "g"]

```

граф выглядел бы так:

Поскольку `c` теперь можно поднять выше `d` в графе, pytest снова может «сплющить» граф до следующего вида:

В этом примере `c` делает `b` и `a` фактически autouse-фикстурами.

С autouse нужно быть осторожным: autouse-фикстура автоматически выполняется для каждого теста, который может до неё «дотянуться», даже если тест её не запрашивает. Например:

```
from __future__ import annotations

import pytest


@pytest.fixture(scope="class")
def order():
    return []


@pytest.fixture(scope="class", autouse=True)
def c1(order):
    order.append("c1")


@pytest.fixture(scope="class")
def c2(order):
    order.append("c2")


@pytest.fixture(scope="class")
def c3(order, c1):
    order.append("c3")


class TestClassWithC1Request:
    def test_order(self, order, c1, c3):
        assert order == ["c1", "c3"]


class TestClassWithoutC1Request:
    def test_order(self, order, c2):
        assert order == ["c1", "c2"]

```

Несмотря на то, что внутри `TestClassWithoutC1Request` ничего не запрашивает `c1`, она всё равно выполняется для тестов внутри этого класса:

Но только потому, что одна autouse-фикстура запросила обычную (не-autouse) фикстуру, это не превращает последнюю в настоящую autouse-фикстуру для всех контекстов, в которых она могла бы применяться. Она фактически становится autouse только для тех контекстов, для которых применяется реальная autouse-фикстура (та, которая её запрашивает).

Например, рассмотрим такой файл:

```
from __future__ import annotations

import pytest


@pytest.fixture
def order():
    return []


@pytest.fixture
def c1(order):
    order.append("c1")


@pytest.fixture
def c2(order):
    order.append("c2")


class TestClassWithAutouse:
    @pytest.fixture(autouse=True)
    def c3(self, order, c2):
        order.append("c3")

    def test_req(self, order, c1):
        assert order == ["c2", "c3", "c1"]

    def test_no_req(self, order):
        assert order == ["c2", "c3"]


class TestClassWithoutAutouse:
    def test_req(self, order, c1):
        assert order == ["c1"]

    def test_no_req(self, order):
        assert order == []

```

Это можно проиллюстрировать так:

Для `test_req` и `test_no_req` внутри `TestClassWithAutouse` `c3` фактически делает `c2` autouse-фикстурой, поэтому `c2` и `c3` выполняются для обоих тестов, несмотря на то что они явно не запрашиваются, и `c2` и `c3` выполняются до `c1` для `test_req`.

Если бы это сделало `c2` настоящей autouse-фикстурой, то `c2` выполнялась бы и для тестов внутри `TestClassWithoutAutouse`, так как эти тесты могли бы сослаться на `c2`, если бы захотели. Но этого не происходит, потому что с точки зрения тестов в `TestClassWithoutAutouse` `c2` не является autouse-фикстурой — они не «видят» `c3`.

