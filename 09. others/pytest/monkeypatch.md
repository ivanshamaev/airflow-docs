# Как делать monkeypatch/mock модулей и окружений

Иногда тестам нужно вызывать функциональность, которая зависит от глобальных настроек или вызывает код, который трудно тестировать, например сетевой доступ. Фикстура `monkeypatch` помогает безопасно установить/удалить атрибут, элемент словаря или переменную окружения, либо модифицировать `sys.path` для импорта.

Фикстура `monkeypatch` предоставляет следующие вспомогательные методы для безопасного патчинга и мокинга в тестах:

[monkeypatch.setattr(obj, name, value, raising=True)](https://docs.pytest.org/en/stable/reference/reference.html#pytest.MonkeyPatch.setattr)

[monkeypatch.delattr(obj, name, raising=True)](https://docs.pytest.org/en/stable/reference/reference.html#pytest.MonkeyPatch.delattr)

[monkeypatch.setitem(mapping, name, value)](https://docs.pytest.org/en/stable/reference/reference.html#pytest.MonkeyPatch.setitem)

[monkeypatch.delitem(obj, name, raising=True)](https://docs.pytest.org/en/stable/reference/reference.html#pytest.MonkeyPatch.delitem)

[monkeypatch.setenv(name, value, prepend=None)](https://docs.pytest.org/en/stable/reference/reference.html#pytest.MonkeyPatch.setenv)

[monkeypatch.delenv(name, raising=True)](https://docs.pytest.org/en/stable/reference/reference.html#pytest.MonkeyPatch.delenv)

[monkeypatch.syspath_prepend(path)](https://docs.pytest.org/en/stable/reference/reference.html#pytest.MonkeyPatch.syspath_prepend)

[monkeypatch.chdir(path)](https://docs.pytest.org/en/stable/reference/reference.html#pytest.MonkeyPatch.chdir)

[monkeypatch.context()](https://docs.pytest.org/en/stable/reference/reference.html#pytest.MonkeyPatch.context)

Все изменения будут отменены после завершения тестовой функции или фикстуры, которые их запросили. Параметр `raising` определяет, будет ли выброшен `KeyError` или `AttributeError`, если цель операции установки/удаления не существует.

Рассмотрим следующие сценарии:

1. Изменение поведения функции или свойства класса для теста. Например, есть API-вызов или подключение к БД, которое вы не хотите выполнять в тесте, но вы знаете ожидаемый результат. Используйте [monkeypatch.setattr](https://docs.pytest.org/en/stable/reference/reference.html#pytest.MonkeyPatch.setattr), чтобы заменить функцию или свойство на желаемое поведение. Это может включать ваши собственные функции. Используйте [monkeypatch.delattr](https://docs.pytest.org/en/stable/reference/reference.html#pytest.MonkeyPatch.delattr), чтобы удалить функцию или свойство на время теста.

2. Изменение значений словарей. Например, у вас есть глобальная конфигурация, которую нужно изменить для отдельных тестов. Используйте [monkeypatch.setitem](https://docs.pytest.org/en/stable/reference/reference.html#pytest.MonkeyPatch.setitem), чтобы изменить словарь для теста. [monkeypatch.delitem](https://docs.pytest.org/en/stable/reference/reference.html#pytest.MonkeyPatch.delitem) можно использовать для удаления элементов.

3. Изменение переменных окружения для теста. Например, чтобы протестировать поведение программы, если переменной окружения нет, или чтобы установить несколько значений в известную переменную. Для этого используйте [monkeypatch.setenv](https://docs.pytest.org/en/stable/reference/reference.html#pytest.MonkeyPatch.setenv) и [monkeypatch.delenv](https://docs.pytest.org/en/stable/reference/reference.html#pytest.MonkeyPatch.delenv).

4. Используйте `monkeypatch.setenv("PATH", value, prepend=os.pathsep)`, чтобы изменить `$PATH`, и [monkeypatch.chdir](https://docs.pytest.org/en/stable/reference/reference.html#pytest.MonkeyPatch.chdir), чтобы менять текущую рабочую директорию в рамках теста.

5. Используйте [monkeypatch.syspath_prepend](https://docs.pytest.org/en/stable/reference/reference.html#pytest.MonkeyPatch.syspath_prepend), чтобы модифицировать `sys.path`; при этом также будет вызван `pkg_resources.fixup_namespace_packages` и [importlib.invalidate_caches()](https://docs.python.org/3/library/importlib.html#importlib.invalidate_caches).

6. Используйте [monkeypatch.context](https://docs.pytest.org/en/stable/reference/reference.html#pytest.MonkeyPatch.context), чтобы применять патчи только в определённой области видимости — это может помочь контролировать teardown сложных фикстур или патчи стандартной библиотеки.

В качестве вводного материала и обсуждения мотивации см. [пост о monkeypatch](https://tetamap.wordpress.com//2009/03/03/monkeypatching-in-unit-tests-done-right/).

## Monkeypatch функций

Рассмотрим сценарий, где вы работаете с пользовательскими директориями. В контексте тестирования вы не хотите, чтобы тест зависел от пользователя, под которым он запущен. `monkeypatch` можно использовать, чтобы «замокать» функции, зависящие от пользователя, так, чтобы они всегда возвращали конкретное значение.

В этом примере [monkeypatch.setattr](https://docs.pytest.org/en/stable/reference/reference.html#pytest.MonkeyPatch.setattr) используется для патчинга `Path.home`, чтобы во время теста всегда использовался известный путь `Path("/abc")`. Это снимает зависимость от пользователя. [monkeypatch.setattr](https://docs.pytest.org/en/stable/reference/reference.html#pytest.MonkeyPatch.setattr) должен быть вызван до функции, которая будет использовать пропатченную функцию. После завершения теста изменение `Path.home` будет отменено.

```
# contents of test_module.py with source code and the test
from pathlib import Path


def getssh():
    """Simple function to return expanded homedir ssh path."""
    return Path.home() / ".ssh"


def test_getssh(monkeypatch):
    # mocked return function to replace Path.home
    # always return '/abc'
    def mockreturn():
        return Path("/abc")

    # Application of the monkeypatch to replace Path.home
    # with the behavior of mockreturn defined above.
    monkeypatch.setattr(Path, "home", mockreturn)

    # Calling getssh() will use mockreturn in place of Path.home
    # for this test with the monkeypatch.
    x = getssh()
    assert x == Path("/abc/.ssh")

```

## Monkeypatch возвращаемых объектов: создание мок‑классов

[monkeypatch.setattr](https://docs.pytest.org/en/stable/reference/reference.html#pytest.MonkeyPatch.setattr) можно использовать вместе с классами, чтобы мокать возвращаемые объекты функций, а не значения. Представьте простую функцию, которая принимает URL API и возвращает JSON-ответ.

```
# contents of app.py, a simple API retrieval example
import requests


def get_json(url):
    """Takes a URL, and returns the JSON."""
    r = requests.get(url)
    return r.json()

```

Нужно замокать `r` — объект ответа, возвращаемый для целей тестирования. У мока `r` должен быть метод `.json()`, возвращающий словарь. Это можно сделать в тестовом файле, определив класс, представляющий `r`.

```
# contents of test_app.py, a simple test for our API retrieval
# import requests for the purposes of monkeypatching
import requests

# our app.py that includes the get_json() function
# this is the previous code block example
import app


# custom class to be the mock return value
# will override the requests.Response returned from requests.get
class MockResponse:
    # mock json() method always returns a specific testing dictionary
    @staticmethod
    def json():
        return {"mock_key": "mock_response"}


def test_get_json(monkeypatch):
    # Any arguments may be passed and mock_get() will always return our
    # mocked object, which only has the .json() method.
    def mock_get(*args, **kwargs):
        return MockResponse()

    # apply the monkeypatch for requests.get to mock_get
    monkeypatch.setattr(requests, "get", mock_get)

    # app.get_json, which contains requests.get, uses the monkeypatch
    result = app.get_json("https://fakeurl")
    assert result["mock_key"] == "mock_response"

```

`monkeypatch` применяет мок для `requests.get` через нашу функцию `mock_get`. `mock_get` возвращает экземпляр класса `MockResponse`, у которого определён метод `json()`, возвращающий известный словарь для тестирования и не требующий реального API-подключения.

Класс `MockResponse` можно сделать настолько сложным, насколько нужно в тестируемом сценарии. Например, можно добавить свойство `ok`, которое всегда возвращает `True`, или возвращать разные значения из замоканного `json()` в зависимости от входных строк.

Этот мок можно переиспользовать между тестами с помощью `fixture`:

```
# contents of test_app.py, a simple test for our API retrieval
import pytest
import requests

# app.py that includes the get_json() function
import app


# custom class to be the mock return value of requests.get()
class MockResponse:
    @staticmethod
    def json():
        return {"mock_key": "mock_response"}


# monkeypatched requests.get moved to a fixture
@pytest.fixture
def mock_response(monkeypatch):
    """Requests.get() mocked to return {'mock_key':'mock_response'}."""

    def mock_get(*args, **kwargs):
        return MockResponse()

    monkeypatch.setattr(requests, "get", mock_get)


# notice our test uses the custom fixture instead of monkeypatch directly
def test_get_json(mock_response):
    result = app.get_json("https://fakeurl")
    assert result["mock_key"] == "mock_response"

```

Более того, если мок должен применяться ко всем тестам, `fixture` можно перенести в `conftest.py` и использовать опцию `autouse=True`.

## Глобальный пример патча: запрет «requests» выполнять удалённые операции

Если вы хотите запретить библиотеке “requests” выполнять HTTP-запросы во всех тестах, можно сделать так:

```
# contents of conftest.py
import pytest


@pytest.fixture(autouse=True)
def no_requests(monkeypatch):
    """Remove requests.sessions.Session.request for all tests."""
    monkeypatch.delattr("requests.sessions.Session.request")

```

Эта autouse-фикстура будет выполняться для каждой тестовой функции и удалит метод `request.session.Session.request`, так что любые попытки выполнить HTTP-запросы в тестах будут приводить к ошибке.

Note

Учтите, что не рекомендуется патчить встроенные функции, такие как `open`, `compile` и т. п., потому что это может сломать внутреннюю работу pytest. Если избежать этого нельзя, могут помочь параметры [--tb=native](https://docs.pytest.org/en/stable/reference/reference.html#cmdoption-tb), [--assert=plain](https://docs.pytest.org/en/stable/reference/reference.html#cmdoption-assert) и [--capture=no](https://docs.pytest.org/en/stable/reference/reference.html#cmdoption-capture), но гарантий нет.

Note

Помните, что патчинг функций `stdlib` и некоторых сторонних библиотек, используемых pytest, может сломать сам pytest; поэтому в таких случаях рекомендуется использовать [MonkeyPatch.context()](https://docs.pytest.org/en/stable/reference/reference.html#pytest.MonkeyPatch.context), чтобы ограничить патчинг только блоком, который вы тестируете:

```
import functools


def test_partial(monkeypatch):
    with monkeypatch.context() as m:
        m.setattr(functools, "partial", 3)
        assert functools.partial == 3

```

Подробнее см. [#3290](https://github.com/pytest-dev/pytest/issues/3290).

## Monkeypatch переменных окружения

Если вы работаете с переменными окружения, часто нужно безопасно менять их значения или удалять их из системы для тестирования. `monkeypatch` предоставляет механизм для этого через методы `setenv` и `delenv`. Исходный код для тестирования:

```
# contents of our original code file e.g. code.py
import os


def get_os_user_lower():
    """Simple retrieval function.
    Returns lowercase USER or raises OSError."""
    username = os.getenv("USER")

    if username is None:
        raise OSError("USER environment is not set.")

    return username.lower()

```

Здесь есть два пути. Первый — переменная окружения `USER` установлена. Второй — переменной `USER` нет. С `monkeypatch` оба пути можно безопасно протестировать без влияния на текущее окружение:

```
# contents of our test file e.g. test_code.py
import pytest


def test_upper_to_lower(monkeypatch):
    """Set the USER env var to assert the behavior."""
    monkeypatch.setenv("USER", "TestingUser")
    assert get_os_user_lower() == "testinguser"


def test_raise_exception(monkeypatch):
    """Remove the USER env var and assert OSError is raised."""
    monkeypatch.delenv("USER", raising=False)

    with pytest.raises(OSError):
        _ = get_os_user_lower()

```

Это поведение можно вынести в структуры `fixture` и переиспользовать в тестах:

```
# contents of our test file e.g. test_code.py
import pytest


@pytest.fixture
def mock_env_user(monkeypatch):
    monkeypatch.setenv("USER", "TestingUser")


@pytest.fixture
def mock_env_missing(monkeypatch):
    monkeypatch.delenv("USER", raising=False)


# notice the tests reference the fixtures for mocks
def test_upper_to_lower(mock_env_user):
    assert get_os_user_lower() == "testinguser"


def test_raise_exception(mock_env_missing):
    with pytest.raises(OSError):
        _ = get_os_user_lower()

```

## Monkeypatch словарей

[monkeypatch.setitem](https://docs.pytest.org/en/stable/reference/reference.html#pytest.MonkeyPatch.setitem) можно использовать, чтобы безопасно задавать значения в словарях на время тестов. Возьмём упрощённый пример строки подключения:

```
# contents of app.py to generate a simple connection string
DEFAULT_CONFIG = {"user": "user1", "database": "db1"}


def create_connection_string(config=None):
    """Creates a connection string from input or defaults."""
    config = config or DEFAULT_CONFIG
    return f"User Id={config['user']}; Location={config['database']};"

```

Для тестирования можно «пропатчить» словарь `DEFAULT_CONFIG` конкретными значениями.

```
# contents of test_app.py
# app.py with the connection string function (prior code block)
import app


def test_connection(monkeypatch):
    # Patch the values of DEFAULT_CONFIG to specific
    # testing values only for this test.
    monkeypatch.setitem(app.DEFAULT_CONFIG, "user", "test_user")
    monkeypatch.setitem(app.DEFAULT_CONFIG, "database", "test_db")

    # expected result based on the mocks
    expected = "User Id=test_user; Location=test_db;"

    # the test uses the monkeypatched dictionary settings
    result = app.create_connection_string()
    assert result == expected

```

Можно использовать [monkeypatch.delitem](https://docs.pytest.org/en/stable/reference/reference.html#pytest.MonkeyPatch.delitem), чтобы удалять значения.

```
# contents of test_app.py
import pytest

# app.py with the connection string function
import app


def test_missing_user(monkeypatch):
    # patch the DEFAULT_CONFIG t be missing the 'user' key
    monkeypatch.delitem(app.DEFAULT_CONFIG, "user", raising=False)

    # Key error expected because a config is not passed, and the
    # default is now missing the 'user' entry.
    with pytest.raises(KeyError):
        _ = app.create_connection_string()

```

Модульность фикстур даёт гибкость: можно определить отдельные фикстуры для каждого мока и использовать их в нужных тестах.

```
# contents of test_app.py
import pytest

# app.py with the connection string function
import app


# all of the mocks are moved into separated fixtures
@pytest.fixture
def mock_test_user(monkeypatch):
    """Set the DEFAULT_CONFIG user to test_user."""
    monkeypatch.setitem(app.DEFAULT_CONFIG, "user", "test_user")


@pytest.fixture
def mock_test_database(monkeypatch):
    """Set the DEFAULT_CONFIG database to test_db."""
    monkeypatch.setitem(app.DEFAULT_CONFIG, "database", "test_db")


@pytest.fixture
def mock_missing_default_user(monkeypatch):
    """Remove the user key from DEFAULT_CONFIG"""
    monkeypatch.delitem(app.DEFAULT_CONFIG, "user", raising=False)


# tests reference only the fixture mocks that are needed
def test_connection(mock_test_user, mock_test_database):
    expected = "User Id=test_user; Location=test_db;"

    result = app.create_connection_string()
    assert result == expected


def test_missing_user(mock_missing_default_user):
    with pytest.raises(KeyError):
        _ = app.create_connection_string()

```

## Справочник API

См. документацию по классу [MonkeyPatch](https://docs.pytest.org/en/stable/reference/reference.html#pytest.MonkeyPatch).