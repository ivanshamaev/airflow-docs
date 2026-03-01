# Добро пожаловать в Pydantic

> **Что нового** — мы запустили [Pydantic Logfire](https://pydantic.dev/logfire) 🔥, чтобы вы могли отслеживать и разбирать валидации Pydantic.

Документация по версии: [v2.12.5](https://github.com/pydantic/pydantic/releases/tag/v2.12.5).

**Pydantic** — самая распространённая библиотека валидации данных для Python.

Она быстрая и расширяемая, удобно работает с линтерами, IDE и вашим рабочим процессом. Описывайте, какими должны быть данные, в чистом каноническом Python 3.9+; проверяйте их с помощью Pydantic.

## Мониторинг Pydantic с Pydantic Logfire 🔥

[Pydantic Logfire](https://pydantic.dev/logfire) — инструмент мониторинга приложений, столь же простой и мощный, как и сам Pydantic.

Logfire интегрируется с многими популярными библиотеками Python, включая FastAPI, OpenAI и сам Pydantic, поэтому с его помощью можно отслеживать валидации Pydantic и понимать, почему часть входных данных не проходит проверку:

```python
from datetime import datetime

import logfire

from pydantic import BaseModel

logfire.configure()
logfire.instrument_pydantic()  # (1)!


class Delivery(BaseModel):
    timestamp: datetime
    dimensions: tuple[int, int]


# успешная валидация будет записана в logfire
m = Delivery(timestamp='2020-01-02T03:04:05Z', dimensions=['10', '20'])
print(repr(m.timestamp))
#> datetime.datetime(2020, 1, 2, 3, 4, 5, tzinfo=TzInfo(UTC))
print(m.dimensions)
#> (10, 20)

Delivery(timestamp='2020-01-02T03:04:05Z', dimensions=['10'])  # (2)!
```

1. По умолчанию Logfire записывает и успешные, и неуспешные валидации; чтобы только неуспешные — укажите `record='failure'`. [Подробнее](https://logfire.pydantic.dev/docs/integrations/pydantic/).
2. Здесь будет выброшено `ValidationError`: в `dimensions` слишком мало элементов; входные данные и ошибки валидации попадут в Logfire.

В платформе Logfire вы увидите примерно такую картину:

*Изображение: мониторинг Pydantic в Logfire (см. [документацию Logfire](https://logfire.pydantic.dev/docs/)).*

Это упрощённый пример, но он показывает ценность инструментирования более сложного приложения.

[Подробнее о Pydantic Logfire](https://logfire.pydantic.dev/docs/)

Подпишитесь на рассылку The Pydantic Stack с новостями и туториалами по Pydantic, Logfire и Pydantic AI: [Subscribe](https://pydantic.dev/newsletter).

## Зачем использовать Pydantic?

- **Типы в основе** — схема валидации и сериализация задаются аннотациями типов: меньше учить, меньше кода, удобная работа с IDE и статическим анализом. [Подробнее…](https://docs.pydantic.dev/latest/why/#type-hints)
- **Скорость** — ядро валидации написано на Rust, поэтому Pydantic входит в число самых быстрых библиотек валидации для Python. [Подробнее…](https://docs.pydantic.dev/latest/why/#performance)
- **JSON Schema** — модели Pydantic могут выдавать JSON Schema для интеграции с другими инструментами. [Подробнее…](https://docs.pydantic.dev/latest/why/#json-schema)
- **Строгий и мягкий режим** — в строгом режиме данные не преобразуются; в мягком Pydantic по возможности приводит данные к нужному типу. [Подробнее…](https://docs.pydantic.dev/latest/why/#strict-lax)
- **Dataclasses, TypedDict и не только** — Pydantic умеет валидировать многие типы из стандартной библиотеки, включая `dataclass` и `TypedDict`. [Подробнее…](https://docs.pydantic.dev/latest/why/#dataclasses-typeddict-more)
- **Гибкая настройка** — в Pydantic можно задавать кастомные валидаторы и сериализаторы и по-разному обрабатывать данные. [Подробнее…](https://docs.pydantic.dev/latest/why/#customisation)
- **Экосистема** — около 8000 пакетов на PyPI зависят от Pydantic, в том числе FastAPI, Hugging Face, Django Ninja, SQLModel и LangChain. [Подробнее…](https://docs.pydantic.dev/latest/why/#ecosystem)
- **Проверено на практике** — Pydantic скачивают более 360 млн раз в месяц; его используют все компании FAANG и 20 из 25 крупнейших компаний на NASDAQ. Если вы хотите что-то сделать с Pydantic, скорее всего, это уже делали. [Подробнее…](https://docs.pydantic.dev/latest/why/#using-pydantic)

Установка: [Installing Pydantic](https://docs.pydantic.dev/latest/install/) — `pip install pydantic`

## Примеры Pydantic

Простой пример: класс, наследующийся от `BaseModel`:

### Успешная валидация

```python
from datetime import datetime

from pydantic import BaseModel, PositiveInt


class User(BaseModel):
    id: int  # (1)!
    name: str = 'John Doe'  # (2)!
    signup_ts: datetime | None  # (3)!
    tastes: dict[str, PositiveInt]  # (4)!


external_data = {
    'id': 123,
    'signup_ts': '2019-06-01 12:22',  # (5)!
    'tastes': {
        'wine': 9,
        b'cheese': 7,  # (6)!
        'cabbage': '1',  # (7)!
    },
}

user = User(**external_data)  # (8)!

print(user.id)  # (9)!
#> 123
print(user.model_dump())  # (10)!
"""
{
    'id': 123,
    'name': 'John Doe',
    'signup_ts': datetime.datetime(2019, 6, 1, 12, 22),
    'tastes': {'wine': 9, 'cheese': 7, 'cabbage': 1},
}
"""
```

1. `id` имеет тип `int`; при такой аннотации поле обязательное. Строки, bytes или float по возможности приводятся к int; иначе выбрасывается исключение.
2. `name` — строка с значением по умолчанию, поле необязательное.
3. `signup_ts` — поле [datetime](https://docs.python.org/3/library/datetime.html#datetime.datetime), обязательное, но можно передать `None`. Pydantic принимает целое (Unix timestamp, например `1496498400`) или строку с датой и временем.
4. `tastes` — словарь со строковыми ключами и положительными целыми. Тип `PositiveInt` — краткая запись для `Annotated[int, annotated_types.Gt(0)]`.
5. Вход — datetime в формате [ISO 8601](https://en.wikipedia.org/wiki/ISO_8601), Pydantic преобразует его в [datetime](https://docs.python.org/3/library/datetime.html#datetime.datetime).
6. Ключ здесь в виде `bytes`, Pydantic приведёт его к строке.
7. Строку `'1'` Pydantic приведёт к целому `1`.
8. Экземпляр `User` создаётся передачей внешних данных как keyword-аргументов.
9. К полям модели обращаются как к атрибутам.
10. Преобразование модели в словарь: [model_dump()](https://docs.pydantic.dev/latest/api/base_model/#pydantic.BaseModel.model_dump).

При ошибке валидации Pydantic выбрасывает исключение с разбором проблем:

### Ошибка валидации

```python
# продолжение примера выше...

from datetime import datetime
from pydantic import BaseModel, PositiveInt, ValidationError


class User(BaseModel):
    id: int
    name: str = 'John Doe'
    signup_ts: datetime | None
    tastes: dict[str, PositiveInt]


external_data = {'id': 'not an int', 'tastes': {}}  # (1)!

try:
    User(**external_data)  # (2)!
except ValidationError as e:
    print(e.errors())
    """
    [
        {
            'type': 'int_parsing',
            'loc': ('id',),
            'msg': 'Input should be a valid integer, unable to parse string as an integer',
            'input': 'not an int',
            'url': 'https://errors.pydantic.dev/2/v/int_parsing',
        },
        {
            'type': 'missing',
            'loc': ('signup_ts',),
            'msg': 'Field required',
            'input': {'id': 'not an int', 'tastes': {}},
            'url': 'https://errors.pydantic.dev/2/v/missing',
        },
    ]
    """
```

1. Входные данные некорректны: `id` не целое число, `signup_ts` отсутствует.
2. При создании `User` выбрасывается [ValidationError](https://docs.pydantic.dev/latest/api/pydantic_core/#pydantic_core.ValidationError) со списком ошибок.

## Кто использует Pydantic?

Сотни организаций и пакетов используют Pydantic. Среди известных компаний и организаций — Adobe, Amazon и AWS, Anthropic, Apple, ASML, AstraZeneca, Cisco Systems, Comcast, Datadog, Facebook, GitHub, Google, HSBC, IBM, Intel, Intuit, Intergovernmental Panel on Climate Change, JPMorgan, Jupyter, Microsoft, Molecular Science Software Institute, NASA, Netflix, NSA, NVIDIA, OpenAI, Oracle, Palantir, Qualcomm, Red Hat, Revolut, Robusta, Salesforce, Starbucks, Texas Instruments, Twilio, Twitter, UK Home Office и другие.

*Изображение: кто использует Pydantic.*

Более полный список open-source проектов: [зависимости на GitHub](https://github.com/pydantic/pydantic/network/dependents) и подборка в [awesome-pydantic](https://github.com/Kludex/awesome-pydantic).
