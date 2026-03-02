# Как использовать subtests

Добавлено в версии 9.0.

Note

Эта возможность экспериментальная. Её поведение, особенно то, как сообщаются ошибки, может меняться в будущих релизах. Однако основная функциональность и использование считаются стабильными.

pytest позволяет группировать утверждения (assertions) внутри обычного теста; такие группы называются subtests.

Subtests — альтернатива параметризации, особенно полезная, когда точные значения параметризации неизвестны на этапе сбора (collection).

```
# content of test_subtest.py


def test(subtests):
    for i in range(5):
        with subtests.test(msg="custom message", i=i):
            assert i % 2 == 0

```

Каждое падение assert или ошибка перехватываются контекстным менеджером и репортятся по отдельности:

```
$ pytest -q test_subtest.py
uuuuuF                                                               [100%]
================================= FAILURES =================================
_______________________ test [custom message] (i=1) ________________________

subtests = <_pytest.subtests.Subtests object at 0xdeadbeef0001>

    def test(subtests):
        for i in range(5):
            with subtests.test(msg="custom message", i=i):
>               assert i % 2 == 0
E               assert (1 % 2) == 0

test_subtest.py:6: AssertionError
_______________________ test [custom message] (i=3) ________________________

subtests = <_pytest.subtests.Subtests object at 0xdeadbeef0001>

    def test(subtests):
        for i in range(5):
            with subtests.test(msg="custom message", i=i):
>               assert i % 2 == 0
E               assert (3 % 2) == 0

test_subtest.py:6: AssertionError
___________________________________ test ___________________________________
contains 2 failed subtests
========================= short test summary info ==========================
SUBFAILED[custom message] (i=1) test_subtest.py::test - assert (1 % 2) == 0
SUBFAILED[custom message] (i=3) test_subtest.py::test - assert (3 % 2) == 0
FAILED test_subtest.py::test - contains 2 failed subtests
3 failed, 3 subtests passed in 0.12s

```

В выводе выше:

Падения subtests отображаются как `SUBFAILED`.

Сначала показываются subtests, а «верхнеуровневый» тест показывается в конце отдельно.

Обратите внимание: можно использовать `subtests` несколько раз в одном тесте или даже сочетать с обычными утверждениями вне блока `subtests.test`:

```
def test(subtests):
    for i in range(5):
        with subtests.test("stage 1", i=i):
            assert i % 2 == 0

    assert func() == 10

    for i in range(10, 20):
        with subtests.test("stage 2", i=i):
            assert i % 2 == 0

```

Note

В качестве альтернативы subtests см. [How to parametrize fixtures and test functions](https://docs.pytest.org/en/stable/how-to/parametrize.html#parametrize).

## Уровень подробности (verbosity)

По умолчанию показываются только падения subtests. При более высоких уровнях подробности ([-v](https://docs.pytest.org/en/stable/reference/reference.html#cmdoption-v)) также будет показан прогресс выполнения успешно прошедших subtests.

Можно управлять подробностью вывода subtests, задав [verbosity_subtests](https://docs.pytest.org/en/stable/reference/reference.html#confval-verbosity_subtests).

## Типизация

[pytest.Subtests](https://docs.pytest.org/en/stable/reference/reference.html#pytest.Subtests) экспортируется, поэтому его можно использовать в аннотациях типов:

```
def test(subtests: pytest.Subtests) -> None: ...

```

## Параметризация vs Subtests

Хотя [традиционная параметризация pytest](https://docs.pytest.org/en/stable/how-to/parametrize.html#parametrize) и `subtests` похожи, у них есть важные различия и сценарии применения.

### Параметризация

Происходит во время collection.

Генерирует отдельные тесты.

На параметризованные тесты можно ссылаться из командной строки.

Хорошо работает с плагинами, которые управляют выполнением тестов, например [--last-failed](https://docs.pytest.org/en/stable/reference/reference.html#cmdoption-last-failed).

Идеально для тестирования по таблицам решений (decision table testing).

### Subtests

Происходят во время выполнения теста.

Не известны во время collection.

Могут генерироваться динамически.

Нельзя ссылаться на них по отдельности из командной строки.

Плагины, которые управляют выполнением тестов, не могут нацеливаться на отдельные subtests.

Падение утверждения внутри subtest не прерывает тест, позволяя увидеть все ошибки в одном отчёте.

Note

Эта возможность изначально была реализована отдельным плагином [pytest-subtests](https://github.com/pytest-dev/pytest-subtests), но начиная с `9.0` была объединена с ядром.

Реализация в core должна быть совместима с реализацией плагина, за исключением отсутствия пользовательских опций командной строки для управления выводом subtests.