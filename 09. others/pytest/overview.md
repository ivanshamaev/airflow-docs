# pytest: помогает писать лучшие программы¶

Фреймворк `pytest` упрощает написание небольших, читаемых тестов и может масштабироваться для поддержки сложного функционального тестирования приложений и библиотек.

Имя пакета на PyPI: [pytest](https://pypi.org/project/pytest)

## Быстрый пример¶

```
# content of test_sample.py
def inc(x):
    return x + 1


def test_answer():
    assert inc(3) == 5

```

Чтобы выполнить:

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
>       assert inc(3) == 5
E       assert 4 == 5
E        +  where 4 = inc(3)

test_sample.py:6: AssertionError
========================= short test summary info ==========================
FAILED test_sample.py::test_answer - assert 4 == 5
============================ 1 failed in 0.12s =============================

```

Благодаря подробной интроспекции утверждений (assertion introspection) в `pytest` используются только обычные инструкции `assert`. Базовое введение в использование pytest см. в разделе [Get started](https://docs.pytest.org/en/stable/getting-started.html#getstarted).

## Возможности¶

Подробная информация о падении [assert-выражений](https://docs.pytest.org/en/stable/how-to/assert.html#assert) (не нужно помнить имена `self.assert*`)

[Автообнаружение](https://docs.pytest.org/en/stable/explanation/goodpractices.html#test-discovery) тестовых модулей и функций

[Модульные фикстуры](https://docs.pytest.org/en/stable/reference/fixtures.html#fixture) для управления небольшими или параметризованными, «долго живущими» ресурсами для тестов

Из коробки может запускать наборы тестов [unittest](https://docs.pytest.org/en/stable/how-to/unittest.html#unittest) (включая trial)

Python 3.10+ или PyPy 3

Богатая архитектура плагинов: более 1300+ [внешних плагинов](https://docs.pytest.org/en/stable/reference/plugin_list.html#plugin-list) и активное сообщество

## Документация¶

[Get started](https://docs.pytest.org/en/stable/getting-started.html#get-started) — установить pytest и освоить основы всего за двадцать минут

[How-to guides](https://docs.pytest.org/en/stable/how-to/index.html#how-to) — пошаговые руководства, охватывающие широкий спектр сценариев и потребностей

[Reference guides](https://docs.pytest.org/en/stable/reference/index.html#reference) — включает полную справку по API pytest, списки плагинов и другое

[Explanation](https://docs.pytest.org/en/stable/explanation/index.html#explanation) — фон, обсуждение ключевых тем, ответы на более «высокоуровневые» вопросы

## Ошибки/запросы¶

Пожалуйста, используйте [трекер задач GitHub](https://github.com/pytest-dev/pytest/issues), чтобы сообщать об ошибках или запрашивать возможности.

## Поддержать pytest¶

[Open Collective](https://opencollective.com/) — онлайн-платформа финансирования для открытых и прозрачных сообществ. Она предоставляет инструменты для сбора средств и публикации финансов в формате полной прозрачности.

Это платформа, которую выбирают частные лица и компании, желающие делать разовые или ежемесячные пожертвования напрямую проекту.

Подробнее см. в [pytest collective](https://opencollective.com/pytest).

## pytest для enterprise¶

Доступно в составе подписки Tidelift.

Мейнтейнеры pytest и тысяч других пакетов работают с Tidelift, чтобы предоставлять коммерческую поддержку и сопровождение зависимостей с открытым исходным кодом, которые вы используете для разработки приложений. Экономьте время, снижайте риски и улучшайте качество кода, при этом оплачивая поддержку именно тех зависимостей, которые вы используете.

[Узнать больше.](https://tidelift.com/subscription/pkg/pypi-pytest?utm_source=pypi-pytest&utm_medium=referral&utm_campaign=enterprise&utm_term=repo)

### Безопасность¶

pytest никогда не был связан с уязвимостями безопасности, но в любом случае, чтобы сообщить об уязвимости, используйте [контакт Tidelift по безопасности](https://tidelift.com/security). Tidelift скоординирует исправление и раскрытие информации.