# Как использовать временные каталоги и файлы в тестах¶

## Фикстура tmp_path¶

Можно использовать фикстуру `tmp_path`, которая предоставляет временный каталог, уникальный для каждой тестовой функции.

`tmp_path` — это объект [pathlib.Path](https://docs.python.org/3/library/pathlib.html#pathlib.Path). Пример использования в тесте:

```
# content of test_tmp_path.py
CONTENT = "content"


def test_create_file(tmp_path):
    d = tmp_path / "sub"
    d.mkdir()
    p = d / "hello.txt"
    p.write_text(CONTENT, encoding="utf-8")
    assert p.read_text(encoding="utf-8") == CONTENT
    assert len(list(tmp_path.iterdir())) == 1
    assert 0

```

При запуске тест бы прошёл, если бы не последняя строка `assert 0`, которую мы используем, чтобы посмотреть значения:

```
$ pytest test_tmp_path.py
=========================== test session starts ============================
platform linux -- Python 3.x.y, pytest-9.x.y, pluggy-1.x.y
rootdir: /home/sweet/project
collected 1 item

test_tmp_path.py F                                                   [100%]

================================= FAILURES =================================
_____________________________ test_create_file _____________________________

tmp_path = PosixPath('PYTEST_TMPDIR/test_create_file0')

    def test_create_file(tmp_path):
        d = tmp_path / "sub"
        d.mkdir()
        p = d / "hello.txt"
        p.write_text(CONTENT, encoding="utf-8")
        assert p.read_text(encoding="utf-8") == CONTENT
        assert len(list(tmp_path.iterdir())) == 1
>       assert 0
E       assert 0

test_tmp_path.py:11: AssertionError
========================= short test summary info ==========================
FAILED test_tmp_path.py::test_create_file - assert 0
============================ 1 failed in 0.12s =============================

```

По умолчанию `pytest` сохраняет временный каталог для последних 3 запусков `pytest`. Параллельные запуски одной и той же тестовой функции поддерживаются при настройке базового временного каталога так, чтобы он был уникален для каждого параллельного запуска. Подробнее см. temporary directory location and retention.

## Фикстура tmp_path_factory¶

`tmp_path_factory` — фикстура с областью `session`, которую можно использовать, чтобы создавать произвольные временные каталоги из других фикстур или тестов.

Например, допустим, вашему набору тестов требуется большой файл изображения на диске, который генерируется процедурно. Вместо того чтобы вычислять одно и то же изображение для каждого теста, который использует своё `tmp_path`, можно сгенерировать его один раз на сессию, чтобы сэкономить время:

```
# contents of conftest.py
import pytest


@pytest.fixture(scope="session")
def image_file(tmp_path_factory):
    img = compute_expensive_image()
    fn = tmp_path_factory.mktemp("data") / "img.png"
    img.save(fn)
    return fn


# contents of test_image.py
def test_histogram(image_file):
    img = load_image(image_file)
    # compute and test histogram

```

Подробнее см. [tmp_path_factory API](https://docs.pytest.org/en/stable/reference/reference.html#tmp-path-factory-factory-api).

## Фикстуры tmpdir и tmpdir_factory¶

Фикстуры `tmpdir` и `tmpdir_factory` похожи на `tmp_path` и `tmp_path_factory`, но используют/возвращают устаревшие объекты [py.path.local](https://py.readthedocs.io/en/latest/path.html), а не стандартные [pathlib.Path](https://docs.python.org/3/library/pathlib.html#pathlib.Path).

Note

В настоящее время предпочтительно использовать `tmp_path` и `tmp_path_factory`.

Чтобы помочь модернизировать старые кодовые базы, pytest можно запускать с отключённым плагином legacypath:

```
pytest -p no:legacypath

```

Это приведёт к ошибкам в тестах, использующих устаревшие пути. Это также можно задать постоянно через параметр [addopts](https://docs.pytest.org/en/stable/reference/reference.html#confval-addopts) в конфигурационном файле.

Подробнее см. API [tmpdir](https://docs.pytest.org/en/stable/reference/reference.html#std-fixture-tmpdir) и [tmpdir_factory](https://docs.pytest.org/en/stable/reference/reference.html#std-fixture-tmpdir_factory).

## Расположение и хранение временных каталогов¶

Временные каталоги, возвращаемые фикстурами [tmp_path](https://docs.pytest.org/en/stable/reference/reference.html#std-fixture-tmp_path) и (теперь устаревшей) [tmpdir](https://docs.pytest.org/en/stable/reference/reference.html#std-fixture-tmpdir), автоматически создаются внутри базового временного каталога. Структура зависит от опции [--basetemp](https://docs.pytest.org/en/stable/reference/reference.html#cmdoption-basetemp):

По умолчанию (когда опция [--basetemp](https://docs.pytest.org/en/stable/reference/reference.html#cmdoption-basetemp) не задана) временные каталоги следуют шаблону:

```
{temproot}/pytest-of-{user}/pytest-{num}/{testname}/

```

где:

`{temproot}` — системный временный каталог, определяемый [tempfile.gettempdir()](https://docs.python.org/3/library/tempfile.html#tempfile.gettempdir). Его можно переопределить переменной окружения [PYTEST_DEBUG_TEMPROOT](https://docs.pytest.org/en/stable/reference/reference.html#envvar-PYTEST_DEBUG_TEMPROOT).

`{user}` — имя пользователя, запускающего тесты,

`{num}` — число, которое увеличивается при каждом запуске набора тестов

`{testname}` — «очищенная» версия [имени текущего теста](https://docs.pytest.org/en/stable/reference/reference.html#pytest.nodes.Node.name).

Автоинкрементируемый плейсхолдер `{num}` обеспечивает базовую «ретеншн» функциональность и предотвращает слепое удаление результатов предыдущих запусков. По умолчанию сохраняются последние 3 временных каталога, но это можно настроить опциями [tmp_path_retention_count](https://docs.pytest.org/en/stable/reference/reference.html#confval-tmp_path_retention_count) и [tmp_path_retention_policy](https://docs.pytest.org/en/stable/reference/reference.html#confval-tmp_path_retention_policy).

Если используется опция [--basetemp](https://docs.pytest.org/en/stable/reference/reference.html#cmdoption-basetemp) (например `pytest --basetemp=mydir`), указанный путь используется напрямую как базовый временный каталог:

```
{basetemp}/{testname}/

```

Обратите внимание: в этом случае нет механизма хранения нескольких запусков — сохраняются только результаты последнего запуска.

Warning

Каталог, указанный в [--basetemp](https://docs.pytest.org/en/stable/reference/reference.html#cmdoption-basetemp), будет безусловно очищаться перед каждым запуском тестов, поэтому убедитесь, что используете каталог только для этой цели.

При распределённом запуске тестов на локальной машине с использованием `pytest-xdist` предпринимаются меры, чтобы автоматически настроить каталог `basetemp` для подпроцессов так, чтобы все временные данные попадали внутрь одного каталога, уникального для конкретного запуска тестов.