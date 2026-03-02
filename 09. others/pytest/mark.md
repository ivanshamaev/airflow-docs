# Как помечать тестовые функции атрибутами¶

С помощью хелпера `pytest.mark` можно легко задавать метаданные для тестовых функций. Полный список встроенных маркеров можно найти в [справочнике API](https://docs.pytest.org/en/stable/reference/reference.html#marks-ref). Также можно вывести все маркеры (включая встроенные и пользовательские) через CLI — `pytest --markers`.

Вот некоторые из встроенных маркеров:

[usefixtures](https://docs.pytest.org/en/stable/how-to/fixtures.html#usefixtures) — использовать фикстуры в тестовой функции или классе

[filterwarnings](https://docs.pytest.org/en/stable/how-to/capture-warnings.html#filterwarnings) — фильтровать определённые предупреждения (warnings) тестовой функции

[skip](https://docs.pytest.org/en/stable/how-to/skipping.html#skip) — всегда пропускать тестовую функцию

[skipif](https://docs.pytest.org/en/stable/how-to/skipping.html#skipif) — пропускать тестовую функцию, если выполняется условие

[xfail](https://docs.pytest.org/en/stable/how-to/skipping.html#xfail) — выдавать результат «ожидаемое падение» (“expected failure”), если выполняется условие

[parametrize](https://docs.pytest.org/en/stable/how-to/parametrize.html#parametrizemark) — выполнять несколько запусков одной и той же тестовой функции.

Легко создавать пользовательские маркеры или применять маркеры ко всему тестовому классу или модулю. Эти маркеры могут использоваться плагинами и также обычно используются для [выбора тестов](https://docs.pytest.org/en/stable/example/markers.html#mark-run) в командной строке опцией [-m](https://docs.pytest.org/en/stable/reference/reference.html#cmdoption-m).

Примеры, которые также служат документацией, см. в [Working with custom markers](https://docs.pytest.org/en/stable/example/markers.html#mark-examples).

Note

Метки (marks) можно применять только к тестам; на [фикстуры](https://docs.pytest.org/en/stable/reference/fixtures.html#fixtures) они не влияют.

## Регистрация маркеров¶

Можно зарегистрировать пользовательские маркеры в конфигурационном файле так:

toml

```
[pytest]
markers = [
    "slow: marks tests as slow (deselect with '-m \"not slow\"')",
    "serial",
]

```

ini

```
[pytest]
markers =
    slow: marks tests as slow (deselect with '-m "not slow"')
    serial

```

Обратите внимание, что всё после `:` в имени маркера — необязательное описание.

Альтернативно можно регистрировать новые маркеры программно в хуке [pytest_configure](https://docs.pytest.org/en/stable/reference/reference.html#initialization-hooks):

```
def pytest_configure(config):
    config.addinivalue_line(
        "markers", "env(name): mark test to run only on named environment"
    )

```

Зарегистрированные маркеры появляются в тексте справки pytest и не вызывают предупреждений (см. следующий раздел). Рекомендуется, чтобы сторонние плагины всегда [регистрировали свои маркеры](https://docs.pytest.org/en/stable/how-to/writing_plugins.html#registering-markers).

## Выдача ошибок при неизвестных маркерах¶

Незарегистрированные маркеры, применённые декоратором `@pytest.mark.name_of_the_mark`, всегда будут выдавать предупреждение, чтобы избежать «тихого» неожиданного поведения из‑за опечаток. Как описано в предыдущем разделе, предупреждение для пользовательских маркеров можно отключить, зарегистрировав их в конфигурационном файле или через пользовательский хук `pytest_configure`.

Когда включена опция конфигурации [strict_markers](https://docs.pytest.org/en/stable/reference/reference.html#confval-strict_markers), любые неизвестные маркеры, применённые декоратором `@pytest.mark.name_of_the_mark`, приведут к ошибке. Эту проверку можно включить в проекте, задав [strict_markers](https://docs.pytest.org/en/stable/reference/reference.html#confval-strict_markers) в конфигурации:

toml

```
[pytest]
addopts = ["--strict-markers"]
markers = [
    "slow: marks tests as slow (deselect with '-m \"not slow\"')",
    "serial",
]

```

ini

```
[pytest]
strict_markers = true
markers =
    slow: marks tests as slow (deselect with '-m "not slow"')
    serial

```