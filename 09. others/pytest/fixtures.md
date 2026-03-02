# Как использовать фикстуры¶

См. также

[About fixtures](https://docs.pytest.org/en/stable/explanation/fixtures.html#about-fixtures)

См. также

[Fixtures reference](https://docs.pytest.org/en/stable/reference/fixtures.html#reference-fixtures)

## «Запрос» фикстур¶

На базовом уровне тестовые функции «запрашивают» нужные им фикстуры, объявляя их как аргументы.

Когда pytest собирается запустить тест, он смотрит на параметры в сигнатуре тестовой функции и затем ищет фикстуры с теми же именами, что и эти параметры. Когда pytest находит их, он выполняет эти фикстуры, сохраняет то, что они вернули (если вообще что‑то вернули), и передаёт эти объекты в тестовую функцию как аргументы.

### Короткий пример¶

```
import pytest


class Fruit:
    def __init__(self, name):
        self.name = name
        self.cubed = False

    def cube(self):
        self.cubed = True


class FruitSalad:
    def __init__(self, *fruit_bowl):
        self.fruit = fruit_bowl
        self._cube_fruit()

    def _cube_fruit(self):
        for fruit in self.fruit:
            fruit.cube()


# Arrange
@pytest.fixture
def fruit_bowl():
    return [Fruit("apple"), Fruit("banana")]


def test_fruit_salad(fruit_bowl):
    # Act
    fruit_salad = FruitSalad(*fruit_bowl)

    # Assert
    assert all(fruit.cubed for fruit in fruit_salad.fruit)

```

В этом примере `test_fruit_salad` «запрашивает» `fruit_bowl` (то есть `def test_fruit_salad(fruit_bowl):`), и когда pytest видит это, он выполнит функцию фикстуры `fruit_bowl` и передаст возвращаемый ею объект в `test_fruit_salad` как аргумент `fruit_bowl`.

Вот примерно что происходит, если сделать это вручную:

```
def fruit_bowl():
    return [Fruit("apple"), Fruit("banana")]


def test_fruit_salad(fruit_bowl):
    # Act
    fruit_salad = FruitSalad(*fruit_bowl)

    # Assert
    assert all(fruit.cubed for fruit in fruit_salad.fruit)


# Arrange
bowl = fruit_bowl()
test_fruit_salad(fruit_bowl=bowl)

```

### Фикстуры могут запрашивать другие фикстуры¶

Одна из сильнейших сторон pytest — чрезвычайно гибкая система фикстур. Она позволяет сводить сложные требования тестов к более простым и организованным функциям, где каждой достаточно описать, от чего она зависит. Ниже мы рассмотрим это подробнее, а пока — быстрый пример того, как фикстуры могут использовать другие фикстуры:

```
# contents of test_append.py
import pytest


# Arrange
@pytest.fixture
def first_entry():
    return "a"


# Arrange
@pytest.fixture
def order(first_entry):
    return [first_entry]


def test_string(order):
    # Act
    order.append("b")

    # Assert
    assert order == ["a", "b"]

```

Обратите внимание: это тот же пример, что и выше, но изменилось очень мало. Фикстуры в pytest «запрашивают» фикстуры так же, как и тесты. Все правила запроса, применимые к тестам, применимы и к фикстурам. Вот как бы работал этот пример, если бы мы делали его вручную:

```
def first_entry():
    return "a"


def order(first_entry):
    return [first_entry]


def test_string(order):
    # Act
    order.append("b")

    # Assert
    assert order == ["a", "b"]


entry = first_entry()
the_list = order(first_entry=entry)
test_string(order=the_list)

```

### Фикстуры переиспользуемы¶

Одна из причин, по которой система фикстур pytest настолько мощная, — она даёт возможность определить обобщённый шаг подготовки (setup), который можно переиспользовать снова и снова, как обычную функцию. Два разных теста могут запросить одну и ту же фикстуру, и pytest выдаст каждому тесту его собственный результат этой фикстуры.

Это чрезвычайно полезно, чтобы убедиться, что тесты не влияют друг на друга. С помощью этой системы можно гарантировать, что каждый тест получает свою «свежую» порцию данных и стартует из чистого состояния, обеспечивая согласованные, воспроизводимые результаты.

Вот пример, где это может пригодиться:

```
# contents of test_append.py
import pytest


# Arrange
@pytest.fixture
def first_entry():
    return "a"


# Arrange
@pytest.fixture
def order(first_entry):
    return [first_entry]


def test_string(order):
    # Act
    order.append("b")

    # Assert
    assert order == ["a", "b"]


def test_int(order):
    # Act
    order.append(2)

    # Assert
    assert order == ["a", 2]

```

Каждый тест получает собственную копию этого объекта `list`, что означает: фикстура `order` выполняется дважды (то же верно и для фикстуры `first_entry`). Если бы мы делали это вручную, это выглядело бы примерно так:

```
def first_entry():
    return "a"


def order(first_entry):
    return [first_entry]


def test_string(order):
    # Act
    order.append("b")

    # Assert
    assert order == ["a", "b"]


def test_int(order):
    # Act
    order.append(2)

    # Assert
    assert order == ["a", 2]


entry = first_entry()
the_list = order(first_entry=entry)
test_string(order=the_list)

entry = first_entry()
the_list = order(first_entry=entry)
test_int(order=the_list)

```

### Тест/фикстура может запрашивать больше одной фикстуры одновременно¶

Тесты и фикстуры не ограничены запросом одной фикстуры за раз. Они могут запросить столько, сколько им нужно. Ещё один быстрый пример:

```
# contents of test_append.py
import pytest


# Arrange
@pytest.fixture
def first_entry():
    return "a"


# Arrange
@pytest.fixture
def second_entry():
    return 2


# Arrange
@pytest.fixture
def order(first_entry, second_entry):
    return [first_entry, second_entry]


# Arrange
@pytest.fixture
def expected_list():
    return ["a", 2, 3.0]


def test_string(order, expected_list):
    # Act
    order.append(3.0)

    # Assert
    assert order == expected_list

```

### Фикстуры можно запрашивать больше одного раза в одном тесте (возвращаемые значения кэшируются)¶

Фикстуры также можно запрашивать больше одного раза в рамках одного и того же теста, и pytest не будет выполнять их повторно для этого теста. Это значит, что мы можем запрашивать фикстуры в нескольких фикстурах, которые от них зависят (и даже снова в самом тесте), не выполняя эти фикстуры более одного раза.

```
# contents of test_append.py
import pytest


# Arrange
@pytest.fixture
def first_entry():
    return "a"


# Arrange
@pytest.fixture
def order():
    return []


# Act
@pytest.fixture
def append_first(order, first_entry):
    return order.append(first_entry)


def test_string_only(append_first, order, first_entry):
    # Assert
    assert order == [first_entry]

```

Если бы запрошенная фикстура выполнялась каждый раз, когда её запрашивают в рамках теста, то этот тест упал бы, потому что и `append_first`, и `test_string_only` видели бы `order` как пустой список (то есть `[]`). Но поскольку возвращаемое значение `order` было закэшировано (вместе с любыми побочными эффектами её выполнения) после первого вызова, и тест, и `append_first` ссылались на один и тот же объект, и тест увидел эффект, который `append_first` оказал на этот объект.

## Autouse фикстуры (фикстуры, которые не нужно запрашивать)¶

Иногда вы хотите иметь фикстуру (или даже несколько), от которых, как вы знаете, зависят все ваши тесты. «Autouse» фикстуры — удобный способ заставить все тесты автоматически «запрашивать» их. Это позволяет убрать много дублирующих запросов и даже даёт более продвинутые варианты использования фикстур (об этом ниже).

Сделать фикстуру autouse можно, передав `autouse=True` в декоратор фикстуры. Простой пример:

```
# contents of test_append.py
import pytest


@pytest.fixture
def first_entry():
    return "a"


@pytest.fixture
def order(first_entry):
    return []


@pytest.fixture(autouse=True)
def append_first(order, first_entry):
    return order.append(first_entry)


def test_string_only(order, first_entry):
    assert order == [first_entry]


def test_string_and_int(order, first_entry):
    order.append(2)
    assert order == [first_entry, 2]

```

В этом примере фикстура `append_first` — autouse. Поскольку она применяется автоматически, оба теста зависят от неё, даже хотя ни один из тестов её явно не запрашивает. Однако это не означает, что её нельзя запросить — просто это не обязательно.

## Scope: совместное использование фикстур между классами, модулями, пакетами или сессией¶

Фикстуры, требующие сетевого доступа, зависят от соединения и обычно дороги по времени создания. Расширяя предыдущий пример, можно добавить параметр `scope="module"` в вызов [@pytest.fixture](https://docs.pytest.org/en/stable/reference/reference.html#pytest.fixture), чтобы функция фикстуры `smtp_connection`, отвечающая за создание подключения к уже существующему SMTP-серверу, вызывалась только один раз на тестовый модуль (по умолчанию — один раз на тестовую функцию). Тогда несколько тестовых функций в модуле будут получать один и тот же экземпляр фикстуры `smtp_connection`, экономя время. Возможные значения `scope`: `function`, `class`, `module`, `package` или `session`.

Следующий пример помещает функцию фикстуры в отдельный файл `conftest.py`, чтобы тесты из нескольких тестовых модулей в каталоге могли обращаться к фикстуре:

```
# content of conftest.py
import smtplib

import pytest


@pytest.fixture(scope="module")
def smtp_connection():
    return smtplib.SMTP("smtp.gmail.com", 587, timeout=5)

```

```
# content of test_module.py


def test_ehlo(smtp_connection):
    response, msg = smtp_connection.ehlo()
    assert response == 250
    assert b"smtp.gmail.com" in msg
    assert 0  # for demo purposes


def test_noop(smtp_connection):
    response, msg = smtp_connection.noop()
    assert response == 250
    assert 0  # for demo purposes

```

Здесь `test_ehlo` нужен результат фикстуры `smtp_connection`. pytest обнаружит и вызовет функцию фикстуры `smtp_connection`, помеченную [@pytest.fixture](https://docs.pytest.org/en/stable/reference/reference.html#pytest.fixture). Запуск теста выглядит так:

```
$ pytest test_module.py
=========================== test session starts ============================
platform linux -- Python 3.x.y, pytest-9.x.y, pluggy-1.x.y
rootdir: /home/sweet/project
collected 2 items

test_module.py FF                                                    [100%]

================================= FAILURES =================================
________________________________ test_ehlo _________________________________

smtp_connection = <smtplib.SMTP object at 0xdeadbeef0001>

    def test_ehlo(smtp_connection):
        response, msg = smtp_connection.ehlo()
        assert response == 250
        assert b"smtp.gmail.com" in msg
>       assert 0  # for demo purposes
        ^^^^^^^^
E       assert 0

test_module.py:7: AssertionError
________________________________ test_noop _________________________________

smtp_connection = <smtplib.SMTP object at 0xdeadbeef0001>

    def test_noop(smtp_connection):
        response, msg = smtp_connection.noop()
        assert response == 250
>       assert 0  # for demo purposes
        ^^^^^^^^
E       assert 0

test_module.py:13: AssertionError
========================= short test summary info ==========================
FAILED test_module.py::test_ehlo - assert 0
FAILED test_module.py::test_noop - assert 0
============================ 2 failed in 0.12s =============================

```

Видно, что падают два `assert 0`, и, что более важно, видно, что в обе тестовые функции был передан один и тот же объект `smtp_connection`, потому что pytest показывает значения входящих аргументов в traceback. В результате две тестовые функции, использующие `smtp_connection`, выполняются почти так же быстро, как одна, поскольку переиспользуют один и тот же экземпляр.

Если вы решите, что предпочитаете иметь `smtp_connection` со scope на сессию, можно просто объявить так:

```
@pytest.fixture(scope="session")
def smtp_connection():
    # the returned fixture value will be shared for
    # all tests requesting it
    ...

```

### Области видимости фикстур (fixture scopes)¶

Фикстуры создаются при первом запросе тестом и уничтожаются в зависимости от `scope`:

`function`: область видимости по умолчанию; фикстура уничтожается в конце теста.

`class`: фикстура уничтожается во время teardown последнего теста в классе.

`module`: фикстура уничтожается во время teardown последнего теста в модуле.

`package`: фикстура уничтожается во время teardown последнего теста в пакете, где определена фикстура, включая подпакеты и подкаталоги внутри него.

`session`: фикстура уничтожается в конце тестовой сессии.

Note

Pytest кэширует только один экземпляр фикстуры за раз, что означает: при использовании параметризованной фикстуры pytest может вызывать фикстуру более одного раза в пределах заданного scope.

### Динамический scope¶

Добавлено в версии 5.2.

В некоторых случаях вы можете захотеть изменить scope фикстуры, не меняя код. Для этого передайте callable в `scope`. Callable должен вернуть строку с корректным scope и будет выполнен только один раз — при определении фикстуры. Он будет вызван с двумя keyword-аргументами: `fixture_name` (строка) и `config` (объект конфигурации).

Это может быть особенно полезно при работе с фикстурами, которым требуется время на setup, например запуск docker-контейнера. Вы можете использовать аргумент командной строки, чтобы управлять scope запущенных контейнеров для разных окружений. См. пример ниже.

```
def determine_scope(fixture_name, config):
    if config.getoption("--keep-containers", None):
        return "session"
    return "function"


@pytest.fixture(scope=determine_scope)
def docker_container():
    yield spawn_container()

```

## Teardown/Cleanup (AKA финализация фикстур)¶

Когда мы запускаем тесты, нам важно убедиться, что они «убирают за собой», чтобы не мешать другим тестам (и чтобы не оставлять горы тестовых данных, раздувающих систему). Фикстуры в pytest предлагают очень полезную систему teardown, которая позволяет определить конкретные шаги, необходимые каждой фикстуре для очистки за собой.

Эту систему можно использовать двумя способами.

### 1. yield-фикстуры (рекомендуется)¶

«Yield»-фикстуры используют `yield` вместо `return`. С такими фикстурами мы можем выполнить некоторый код и вернуть объект в запрашивающую фикстуру/тест, как и с другими фикстурами. Отличия только в том, что:

`return` заменяется на `yield`.

Любой teardown-код для этой фикстуры размещается после `yield`.

Когда pytest определит линейный порядок фикстур, он выполнит каждую из них до места, где она возвращает значение или делает yield, затем перейдёт к следующей фикстуре в списке и сделает то же самое.

После завершения теста pytest пройдёт по списку фикстур в обратном порядке, возьмёт каждую, которая делала yield, и выполнит код внутри неё, который находится после `yield`.

В качестве простого примера рассмотрим базовый email-модуль:

```
# content of emaillib.py
class MailAdminClient:
    def create_user(self):
        return MailUser()

    def delete_user(self, user):
        # do some cleanup
        pass


class MailUser:
    def __init__(self):
        self.inbox = []

    def send_email(self, email, other):
        other.inbox.append(email)

    def clear_mailbox(self):
        self.inbox.clear()


class Email:
    def __init__(self, subject, body):
        self.subject = subject
        self.body = body

```

Допустим, мы хотим протестировать отправку письма от одного пользователя другому. Нам нужно сначала создать каждого пользователя, затем отправить письмо от одного пользователю к другому и, наконец, проверить, что второй пользователь получил сообщение во входящих. Если мы хотим «убраться» после теста, нам, вероятно, нужно убедиться, что почтовый ящик второго пользователя очищен до удаления пользователя, иначе система может пожаловаться.

Вот как это может выглядеть:

```
# content of test_emaillib.py
from emaillib import Email, MailAdminClient

import pytest


@pytest.fixture
def mail_admin():
    return MailAdminClient()


@pytest.fixture
def sending_user(mail_admin):
    user = mail_admin.create_user()
    yield user
    mail_admin.delete_user(user)


@pytest.fixture
def receiving_user(mail_admin):
    user = mail_admin.create_user()
    yield user
    user.clear_mailbox()
    mail_admin.delete_user(user)


def test_email_received(sending_user, receiving_user):
    email = Email(subject="Hey!", body="How's it going?")
    sending_user.send_email(email, receiving_user)
    assert email in receiving_user.inbox

```

Поскольку `receiving_user` — последняя фикстура, запускаемая при setup, она же запускается первой при teardown.

Есть риск, что даже правильный порядок действий при teardown не гарантирует безопасную очистку. Это чуть подробнее рассматривается в Safe teardowns.

```
$ pytest -q test_emaillib.py
.                                                                    [100%]
1 passed in 0.12s

```

#### Обработка ошибок для yield-фикстуры¶

Если yield-фикстура выбрасывает исключение до yield, pytest не будет пытаться выполнить teardown-код после `yield` этой фикстуры. Однако для каждой фикстуры, которая уже успела успешно выполниться для данного теста, pytest всё равно попытается выполнить teardown как обычно.

### 2. Добавление финализаторов напрямую¶

Хотя yield-фикстуры считаются более «чистым» и прямолинейным вариантом, есть и другой подход — добавлять функции‑«финализаторы» (finalizer) напрямую в объект контекста запроса (request-context) теста. Результат похож на yield-фикстуры, но требует немного больше многословности.

Чтобы использовать этот подход, нужно запросить объект контекста `request` (так же, как мы запрашиваем любую другую фикстуру) в фикстуре, для которой требуется teardown-код, а затем передать callable, содержащий teardown-код, в метод `addfinalizer`.

Однако нужно быть осторожным, потому что pytest выполнит финализатор, как только он добавлен, даже если фикстура выбросит исключение после добавления финализатора. Поэтому, чтобы не выполнять финализатор тогда, когда в нём нет нужды, мы добавляем финализатор только после того, как фикстура сделала что‑то, что нужно будет «откатывать» (teardown).

Вот как выглядел бы предыдущий пример с использованием `addfinalizer`:

```
# content of test_emaillib.py
from emaillib import Email, MailAdminClient

import pytest


@pytest.fixture
def mail_admin():
    return MailAdminClient()


@pytest.fixture
def sending_user(mail_admin):
    user = mail_admin.create_user()
    yield user
    mail_admin.delete_user(user)


@pytest.fixture
def receiving_user(mail_admin, request):
    user = mail_admin.create_user()

    def delete_user():
        mail_admin.delete_user(user)

    request.addfinalizer(delete_user)
    return user


@pytest.fixture
def email(sending_user, receiving_user, request):
    _email = Email(subject="Hey!", body="How's it going?")
    sending_user.send_email(_email, receiving_user)

    def empty_mailbox():
        receiving_user.clear_mailbox()

    request.addfinalizer(empty_mailbox)
    return _email


def test_email_received(receiving_user, email):
    assert email in receiving_user.inbox

```

Это немного длиннее, чем yield-фикстуры, и немного сложнее, но иногда даёт нюансы, которые могут помочь, когда вы «в безвыходной ситуации».

```
$ pytest -q test_emaillib.py
.                                                                    [100%]
1 passed in 0.12s

```

#### Примечание о порядке финализаторов¶

Финализаторы выполняются в порядке first-in-last-out. Для yield-фикстур первый teardown-код, который выполняется, относится к самой правой фикстуре, то есть к последнему параметру теста.

```
# content of test_finalizers.py
import pytest


def test_bar(fix_w_yield1, fix_w_yield2):
    print("test_bar")


@pytest.fixture
def fix_w_yield1():
    yield
    print("after_yield_1")


@pytest.fixture
def fix_w_yield2():
    yield
    print("after_yield_2")

```

```
$ pytest -s test_finalizers.py
=========================== test session starts ============================
platform linux -- Python 3.x.y, pytest-9.x.y, pluggy-1.x.y
rootdir: /home/sweet/project
collected 1 item

test_finalizers.py test_bar
.after_yield_2
after_yield_1


============================ 1 passed in 0.12s =============================

```

Для финализаторов первая фикстура, которая выполняется, — это последний вызов `request.addfinalizer`.

```
# content of test_finalizers.py
from functools import partial
import pytest


@pytest.fixture
def fix_w_finalizers(request):
    request.addfinalizer(partial(print, "finalizer_2"))
    request.addfinalizer(partial(print, "finalizer_1"))


def test_bar(fix_w_finalizers):
    print("test_bar")

```

```
$ pytest -s test_finalizers.py
=========================== test session starts ============================
platform linux -- Python 3.x.y, pytest-9.x.y, pluggy-1.x.y
rootdir: /home/sweet/project
collected 1 item

test_finalizers.py test_bar
.finalizer_1
finalizer_2


============================ 1 passed in 0.12s =============================

```

Это происходит потому, что yield-фикстуры используют `addfinalizer` «под капотом»: когда фикстура выполняется, `addfinalizer` регистрирует функцию, которая возобновляет генератор, который, в свою очередь, вызывает teardown-код.

## Безопасные teardown (Safe teardowns)¶

Система фикстур pytest очень мощная, но она всё равно выполняется компьютером, а значит не способна сама понять, как безопасно «разбирать» (teardown) всё, что мы ей дадим. Если не быть осторожным, ошибка в неподходящем месте может оставить после тестов «хвосты», и это довольно быстро приведёт к дальнейшим проблемам.

Например, рассмотрим следующие тесты (на основе примера с почтой выше):

```
# content of test_emaillib.py
from emaillib import Email, MailAdminClient

import pytest


@pytest.fixture
def setup():
    mail_admin = MailAdminClient()
    sending_user = mail_admin.create_user()
    receiving_user = mail_admin.create_user()
    email = Email(subject="Hey!", body="How's it going?")
    sending_user.send_email(email, receiving_user)
    yield receiving_user, email
    receiving_user.clear_mailbox()
    mail_admin.delete_user(sending_user)
    mail_admin.delete_user(receiving_user)


def test_email_received(setup):
    receiving_user, email = setup
    assert email in receiving_user.inbox

```

Эта версия гораздо компактнее, но её сложнее читать, у фикстуры не очень описательное имя, и ни одна из фикстур не может быть легко переиспользована.

Есть и более серьёзная проблема: если любой из шагов setup выбросит исключение, никакой teardown-код не выполнится.

Один из вариантов — использовать `addfinalizer` вместо yield-фикстур, но это может стать довольно сложным и трудным в поддержке (и уже не будет компактным).

```
$ pytest -q test_emaillib.py
.                                                                    [100%]
1 passed in 0.12s

```

### Безопасная структура фикстур¶

Самая безопасная и простая структура фикстур требует ограничить каждую фикстуру одним действием, изменяющим состояние, и затем «оборачивать» это действие вместе с teardown-кодом, как показывали email-примеры выше.

Вероятность того, что операция изменения состояния может упасть, но всё же изменить состояние, пренебрежимо мала, так как большинство таких операций основаны на [транзакциях](https://en.wikipedia.org/wiki/Transaction_processing) (по крайней мере на уровне тестирования, где может остаться «состояние»). Поэтому если мы гарантируем, что любое успешное изменение состояния будет откатано, вынеся его в отдельную фикстуру и отделив от других потенциально падающих операций изменения состояния, то наши тесты с наибольшей вероятностью оставят окружение тестирования в таком же виде, в каком нашли.

Для примера, допустим, у нас есть сайт со страницей логина, и у нас есть admin API, через которое можно создавать пользователей. В тесте мы хотим:

Создать пользователя через admin API

Запустить браузер с помощью Selenium

Перейти на страницу логина нашего сайта

Войти как созданный пользователь

Проверить, что имя пользователя есть в заголовке landing page

Мы не хотим оставлять пользователя в системе и не хотим оставлять запущенную сессию браузера, поэтому нам нужно убедиться, что фикстуры, создающие эти вещи, убирают за собой.

Вот как это может выглядеть:

Note

В этом примере некоторые фикстуры (то есть `base_url` и `admin_credentials`) подразумеваются существующими где‑то ещё. Пока будем считать, что они существуют, просто мы на них не смотрим.
