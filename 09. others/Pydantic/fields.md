# Поля (Fields)

Документация API: [pydantic.fields.Field](https://docs.pydantic.dev/latest/api/fields/#pydantic.fields.Field)

В этом разделе описаны способы настройки полей моделей Pydantic: значения по умолчанию, метаданные JSON Schema, ограничения и т.д.

Для этого используется функция [Field()](https://docs.pydantic.dev/latest/api/fields/#pydantic.fields.Field). Она ведёт себя так же, как [field()](https://docs.python.org/3/library/dataclasses.html#dataclasses.field) из стандартной библиотеки для dataclasses — присваивается аннотированному атрибуту:

```python
from pydantic import BaseModel, Field


class Model(BaseModel):
    name: str = Field(frozen=True)
```

> Несмотря на то что `name` присвоено значение, поле остаётся обязательным и не имеет значения по умолчанию. Чтобы явно подчеркнуть, что значение должно быть передано, можно использовать [многоточие](https://docs.python.org/3/library/constants.html#Ellipsis):
>
> Примечание

```python
class Model(BaseModel):
    name: str = Field(..., frozen=True)
```

Однако такой приём не рекомендуется: он плохо сочетается со статическими анализаторами типов.

## Аннотированный паттерн (The annotated pattern)

Чтобы задать ограничения или прикрепить [Field()](https://docs.pydantic.dev/latest/api/fields/#pydantic.fields.Field) к полю модели, Pydantic также поддерживает конструкцию [Annotated](https://docs.python.org/3/library/typing.html#typing.Annotated) для добавления метаданных к аннотации:

```python
from typing import Annotated

from pydantic import BaseModel, Field, WithJsonSchema


class Model(BaseModel):
    name: Annotated[str, Field(strict=True), WithJsonSchema({'extra': 'data'})]
```

Для статических анализаторов типов `name` по-прежнему имеет тип `str`, а Pydantic использует метаданные для валидации, ограничений типов и т.д.

У этого подхода есть плюсы:

- Типы можно выносить и переиспользовать (см. [кастомные типы](https://docs.pydantic.dev/latest/concepts/types/#using-the-annotated-pattern)).
- К полю можно привязать произвольное количество элементов метаданных. У [Field()](https://docs.pydantic.dev/latest/api/fields/#pydantic.fields.Field) ограниченный набор ограничений/метаданных; в ряде случаев нужны другие утилиты, например [WithJsonSchema](https://docs.pydantic.dev/latest/api/json_schema/#pydantic.json_schema.WithJsonSchema).
- Запись вида `f = Field(...)` может создавать впечатление, что у `f` есть значение по умолчанию, хотя поле остаётся обязательным.
- Декоратор computed_field

Однако аргументы [Field()](https://docs.pydantic.dev/latest/api/fields/#pydantic.fields.Field) `default`, `default_factory` и `alias` учитываются статическими анализаторами при формировании `__init__()`. Аннотированный паттерн ими не понимается, поэтому для них нужно использовать обычную форму с присваиванием.

> Аннотированный паттерн можно применять и к отдельным частям типа. Например, так задают ограничения валидации:
>
> Совет

```python
from typing import Annotated

from pydantic import BaseModel, Field


class Model(BaseModel):
    int_list: list[Annotated[int, Field(gt=0)]]
    # Валидно: [1, 3]
    # Невалидно: [-1, 2]
```

Не смешивайте метаданные поля и метаданные типа:

```python
class Model(BaseModel):
    field_bad: Annotated[int, Field(deprecated=True)] | None = None  # (1)!
    field_ok: Annotated[int | None, Field(deprecated=True)] = None  # (2)!
```

1. [Field()](https://docs.pydantic.dev/latest/api/fields/#pydantic.fields.Field) применяется к типу `int`, поэтому флаг `deprecated` не действует. Название функции подразумевает применение к полю, но API проектировался, когда это был единственный способ задать метаданные. Альтернатива — библиотека [annotated_types](https://github.com/annotated-types/annotated-types), которую Pydantic поддерживает.
2. [Field()](https://docs.pydantic.dev/latest/api/fields/#pydantic.fields.Field) применяется к «верхнему» union-типу, поэтому флаг `deprecated` относится к полю.

## Инспекция полей модели (Inspecting model fields)

Поля модели доступны через атрибут класса [model_fields](https://docs.pydantic.dev/latest/api/base_model/#pydantic.BaseModel.model_fields) (или `__pydantic_fields__` для [Pydantic dataclasses](https://docs.pydantic.dev/latest/concepts/dataclasses/)). Это отображение имён полей на их описание (экземпляры [FieldInfo](https://docs.pydantic.dev/latest/api/fields/#pydantic.fields.FieldInfo)).

```python
from typing import Annotated

from pydantic import BaseModel, Field, WithJsonSchema


class Model(BaseModel):
    a: Annotated[
        int, Field(gt=1), WithJsonSchema({'extra': 'data'}), Field(alias='b')
    ] = 1


field_info = Model.model_fields['a']
print(field_info.annotation)
#> <class 'int'>
print(field_info.alias)
#> b
print(field_info.metadata)
#> [Gt(gt=1), WithJsonSchema(json_schema={'extra': 'data'}, mode=None)]
```

## Значения по умолчанию (Default values)

Значения по умолчанию задают обычным присваиванием или аргументом `default`:

```python
from pydantic import BaseModel, Field


class User(BaseModel):
    # Оба поля необязательны:
    name: str = 'John Doe'
    age: int = Field(default=20)
```

> В [Pydantic V1](https://docs.pydantic.dev/latest/migration/#required-optional-and-nullable-fields) тип [Any](https://docs.python.org/3/library/typing.html#typing.Any) или обёрнутый в [Optional](https://docs.python.org/3/library/typing.html#typing.Optional) получал неявное значение по умолчанию `None`, даже если оно не задавалось. В Pydantic V2 так больше не делается.
>
> Предупреждение

Фабрику значения по умолчанию задают через аргумент `default_factory` (вызываемый без аргументов):

```python
from uuid import uuid4

from pydantic import BaseModel, Field


class User(BaseModel):
    id: str = Field(default_factory=lambda: uuid4().hex)
```

`default_factory` может принимать один обязательный аргумент — тогда в него передаётся уже провалидированная часть данных в виде словаря:

```python
from pydantic import BaseModel, EmailStr, Field


class User(BaseModel):
    email: EmailStr
    username: str = Field(default_factory=lambda data: data['email'])


user = User(email='[email protected]')
print(user.username)
#> [email protected]
```

Аргумент `data` содержит только уже провалидированные данные с учётом [порядка полей модели](https://docs.pydantic.dev/latest/concepts/models/#field-ordering) (пример выше не сработает, если `username` объявлен перед `email`).

## Валидация значений по умолчанию (Validate default values)

По умолчанию Pydantic не валидирует значения по умолчанию. Параметр поля `validate_default` (или конфигурация [validate_default](https://docs.pydantic.dev/latest/api/config/#pydantic.config.ConfigDict.validate_default)) включает такую проверку:

```python
from pydantic import BaseModel, Field, ValidationError


class User(BaseModel):
    age: int = Field(default='twelve', validate_default=True)


try:
    user = User()
except ValidationError as e:
    print(e)
    """
    1 validation error for User
    age
      Input should be a valid integer, unable to parse string as an integer [type=int_parsing, input_value='twelve', input_type=str]
    """
```

### Изменяемые значения по умолчанию (Mutable default values)

Частая причина ошибок в Python — использование изменяемого объекта как значения по умолчанию для аргумента функции: один и тот же экземпляр используется при каждом вызове.

Модуль [dataclasses](https://docs.python.org/3/library/dataclasses.html#module-dataclasses) в таком случае выдаёт ошибку и рекомендует [default factory](https://docs.python.org/3/library/dataclasses.html#default-factory-functions).

В Pydantic то же можно сделать, но не обязательно. Если значение по умолчанию не хешируемо, Pydantic при создании каждого экземпляра модели делает его глубокую копию:

```python
from pydantic import BaseModel


class Model(BaseModel):
    item_counts: list[dict[str, int]] = [{}]


m1 = Model()
m1.item_counts[0]['a'] = 1
print(m1.item_counts)
#> [{'a': 1}]

m2 = Model()
print(m2.item_counts)
#> [{}]
```

## Алиасы полей (Field aliases)

> Подробнее об алиасах см. в [отдельном разделе](https://docs.pydantic.dev/latest/concepts/alias/).
>
> Совет

Для валидации и сериализации можно задать алиас поля. Способы:

- `Field(serialization_alias='foo')`
- `Field(validation_alias='foo')`
- `Field(alias='foo')`

Параметр `alias` используется и при валидации, и при сериализации. Для разных алиасов задайте `validation_alias` и `serialization_alias`.

Пример с `alias`:

```python
from pydantic import BaseModel, Field


class User(BaseModel):
    name: str = Field(alias='username')


user = User(username='johndoe')  # (1)!
print(user)
#> name='johndoe'
print(user.model_dump(by_alias=True))  # (2)!
#> {'username': 'johndoe'}
```

Для перевода модели в сериализуемый формат используется [model_dump()](https://docs.pydantic.dev/latest/api/base_model/#pydantic.BaseModel.model_dump). Аргумент `by_alias` по умолчанию `False`; чтобы выводить поля по алиасам сериализации, его нужно явно указать. Поведение можно задать на уровне модели через [ConfigDict.serialize_by_alias](https://docs.pydantic.dev/latest/api/config/#pydantic.config.ConfigDict.serialize_by_alias).

При `by_alias=True` при сериализации используется алиас `'username'`.

1. Алиас `'username'` используется при создании экземпляра и при валидации.

Только для валидации — параметр `validation_alias`:

```python
from pydantic import BaseModel, Field


class User(BaseModel):
    name: str = Field(validation_alias='username')


user = User(username='johndoe')  # (1)!
print(user)
#> name='johndoe'
print(user.model_dump(by_alias=True))  # (2)!
#> {'name': 'johndoe'}
```

1. При сериализации используется имя поля `'name'`.
2. При валидации используется алиас `'username'`.

Только для сериализации — параметр `serialization_alias`:

```python
from pydantic import BaseModel, Field


class User(BaseModel):
    name: str = Field(serialization_alias='username')


user = User(name='johndoe')  # (1)!
print(user)
#> name='johndoe'
print(user.model_dump(by_alias=True))  # (2)!
#> {'username': 'johndoe'}
```

1. Алиас сериализации `'username'` используется при сериализации.
2. При валидации используется имя поля `'name'`.

**Приоритет алиасов:** при одновременном использовании `alias` с `validation_alias` или `serialization_alias` для валидации приоритет у `validation_alias`, для сериализации — у `serialization_alias`. При заданной настройке модели [alias_generator](https://docs.pydantic.dev/latest/api/config/#pydantic.config.ConfigDict.alias_generator) порядок приоритета алиаса поля и сгенерированных алиасов задаётся параметром поля `alias_priority`. Подробнее: [приоритет алиасов](https://docs.pydantic.dev/latest/concepts/alias/#alias-precedence).

**Статическая проверка типов / поддержка IDE:** при задании параметра `alias` анализаторы типов подставляют этот алиас вместо имени поля в `__init__`:

```python
from pydantic import BaseModel, Field


class User(BaseModel):
    name: str = Field(alias='username')


user = User(username='johndoe')  # (1)!
```

1. Принимается анализаторами типов.

При настройке модели [validate_by_name](https://docs.pydantic.dev/latest/api/config/#pydantic.config.ConfigDict.validate_by_name) (допускающей при валидации и имя поля, и алиас) анализаторы будут ругаться на использование имени поля:

```python
from pydantic import BaseModel, ConfigDict, Field


class User(BaseModel):
    model_config = ConfigDict(validate_by_name=True)

    name: str = Field(alias='username')


user = User(name='johndoe')  # (1)!
```

1. Не принимается анализаторами типов.

Чтобы анализаторы использовали имя поля, а не алиас, можно применить аннотированный паттерн (его понимает только Pydantic):

```python
from typing import Annotated

from pydantic import BaseModel, ConfigDict, Field


class User(BaseModel):
    model_config = ConfigDict(validate_by_name=True, validate_by_alias=True)

    name: Annotated[str, Field(alias='username')]


user = User(name='johndoe')  # (1)!
user = User(username='johndoe')  # (2)!
```

1. Не принимается анализаторами типов.
2. Принимается анализаторами типов.

### Validation Alias

Pydantic при создании экземпляров обрабатывает `alias` и `validation_alias` одинаково, но анализаторы типов понимают только параметр поля `alias`. Обходной вариант — задать и `alias`, и `serialization_alias` (равный имени поля); при сериализации `serialization_alias` переопределит `alias`:

```python
from pydantic import BaseModel, Field


class MyModel(BaseModel):
    my_field: int = Field(validation_alias='myValidationAlias')
```

заменить на:

```python
from pydantic import BaseModel, Field


class MyModel(BaseModel):
    my_field: int = Field(
        alias='myValidationAlias',
        serialization_alias='my_field',
    )


m = MyModel(myValidationAlias=1)
print(m.model_dump(by_alias=True))
#> {'my_field': 1}
```

## Ограничения полей (Field constraints)

Функция [Field()](https://docs.pydantic.dev/latest/api/fields/#pydantic.fields.Field) может задавать ограничения для конкретных типов:

```python
from decimal import Decimal

from pydantic import BaseModel, Field


class Model(BaseModel):
    positive: int = Field(gt=0)
    short_str: str = Field(max_length=3)
    precise_decimal: Decimal = Field(max_digits=5, decimal_places=2)
```

Доступные ограничения по типам и их влияние на JSON Schema описаны в [документации по стандартным типам](https://docs.pydantic.dev/latest/api/standard_library_types/).

## Строгие поля (Strict fields)

Параметр `strict` функции [Field()](https://docs.pydantic.dev/latest/api/fields/#pydantic.fields.Field) задаёт валидацию поля в [строгом режиме](https://docs.pydantic.dev/latest/concepts/strict_mode/).

```python
from pydantic import BaseModel, Field


class User(BaseModel):
    name: str = Field(strict=True)
    age: int = Field(strict=False)  # (1)!


user = User(name='John', age='42')  # (2)!
print(user)
#> name='John' age=42
```

1. Поле `age` валидируется в «свободном» режиме, поэтому допускается строка.
2. Это значение по умолчанию.

Поведение в строгом режиме по типам описано в [документации по стандартным типам](https://docs.pydantic.dev/latest/api/standard_library_types/).

## Поля dataclass (Dataclass fields)

Некоторые параметры [Field()](https://docs.pydantic.dev/latest/api/fields/#pydantic.fields.Field) применимы к [dataclasses](https://docs.pydantic.dev/latest/concepts/dataclasses/):

- `kw_only`: поле становится только keyword-аргументом в конструкторе dataclass.
- `init_var`: поле считается [только для инициализации](https://docs.python.org/3/library/dataclasses.html#dataclasses-init-only-variables).
- `init`: включать ли поле в сгенерированный `__init__()` dataclass.

Пример:

```python
from pydantic import BaseModel, Field
from pydantic.dataclasses import dataclass


@dataclass
class Foo:
    bar: str
    baz: str = Field(init_var=True)
    qux: str = Field(kw_only=True)


class Model(BaseModel):
    foo: Foo


model = Model(foo=Foo('bar', baz='baz', qux='qux'))
print(model.model_dump())  # (1)!
#> {'foo': {'bar': 'bar', 'qux': 'qux'}}
```

1. Поле `baz` не входит в результат сериализации, так как оно только для инициализации.

## Представление поля (Field Representation)

Параметр `repr` задаёт, включать ли поле в строковое представление модели.

```python
from pydantic import BaseModel, Field


class User(BaseModel):
    name: str = Field(repr=True)  # (1)!
    age: int = Field(repr=False)


user = User(name='John', age=42)
print(user)
#> name='John'
```

1. Это значение по умолчанию.

## Дискриминатор (Discriminator)

Параметр `discriminator` задаёт поле, по которому различают варианты union моделей. Принимает имя поля или экземпляр `Discriminator`. Вариант с `Discriminator` удобен, когда поле-дискриминатор у моделей в union разное.

Пример с именем поля:

**Python 3.9+** (Union):

```python
from typing import Literal, Union

from pydantic import BaseModel, Field


class Cat(BaseModel):
    pet_type: Literal['cat']
    age: int


class Dog(BaseModel):
    pet_type: Literal['dog']
    age: int


class Model(BaseModel):
    pet: Union[Cat, Dog] = Field(discriminator='pet_type')


print(Model.model_validate({'pet': {'pet_type': 'cat', 'age': 12}}))  # (1)!
#> pet=Cat(pet_type='cat', age=12)
```

1. Подробнее см. [Validating data](https://docs.pydantic.dev/latest/concepts/models/#validating-data) на странице [Models](https://docs.pydantic.dev/latest/concepts/models/).

**Python 3.10+** (синтаксис `|`):

```python
from typing import Literal

from pydantic import BaseModel, Field


class Cat(BaseModel):
    pet_type: Literal['cat']
    age: int


class Dog(BaseModel):
    pet_type: Literal['dog']
    age: int


class Model(BaseModel):
    pet: Cat | Dog = Field(discriminator='pet_type')


print(Model.model_validate({'pet': {'pet_type': 'cat', 'age': 12}}))  # (1)!
#> pet=Cat(pet_type='cat', age=12)
```

1. Подробнее см. [Validating data](https://docs.pydantic.dev/latest/concepts/models/#validating-data) на странице [Models](https://docs.pydantic.dev/latest/concepts/models/).

Пример с аргументом `discriminator` и экземпляром `Discriminator`:

**Python 3.9+**:

```python
from typing import Annotated, Literal, Union

from pydantic import BaseModel, Discriminator, Field, Tag


class Cat(BaseModel):
    pet_type: Literal['cat']
    age: int


class Dog(BaseModel):
    pet_kind: Literal['dog']
    age: int


def pet_discriminator(v):
    if isinstance(v, dict):
        return v.get('pet_type', v.get('pet_kind'))
    return getattr(v, 'pet_type', getattr(v, 'pet_kind', None))


class Model(BaseModel):
    pet: Union[Annotated[Cat, Tag('cat')], Annotated[Dog, Tag('dog')]] = Field(
        discriminator=Discriminator(pet_discriminator)
    )


print(repr(Model.model_validate({'pet': {'pet_type': 'cat', 'age': 12}})))
#> Model(pet=Cat(pet_type='cat', age=12))

print(repr(Model.model_validate({'pet': {'pet_kind': 'dog', 'age': 12}})))
#> Model(pet=Dog(pet_kind='dog', age=12))
```

**Python 3.10+**:

```python
from typing import Annotated, Literal

from pydantic import BaseModel, Discriminator, Field, Tag


class Cat(BaseModel):
    pet_type: Literal['cat']
    age: int


class Dog(BaseModel):
    pet_kind: Literal['dog']
    age: int


def pet_discriminator(v):
    if isinstance(v, dict):
        return v.get('pet_type', v.get('pet_kind'))
    return getattr(v, 'pet_type', getattr(v, 'pet_kind', None))


class Model(BaseModel):
    pet: Annotated[Cat, Tag('cat')] | Annotated[Dog, Tag('dog')] = Field(
        discriminator=Discriminator(pet_discriminator)
    )


print(repr(Model.model_validate({'pet': {'pet_type': 'cat', 'age': 12}})))
#> Model(pet=Cat(pet_type='cat', age=12))

print(repr(Model.model_validate({'pet': {'pet_kind': 'dog', 'age': 12}})))
#> Model(pet=Dog(pet_kind='dog', age=12))
```

Дискриминированные union можно задавать и через `Annotated`. Подробнее: [Discriminated Unions](https://docs.pydantic.dev/latest/concepts/unions/#discriminated-unions).

## Неизменяемость (Immutability)

Параметр `frozen` имитирует поведение замороженного dataclass: запрещает присваивание полю после создания модели. Подробнее: [документация по frozen dataclass](https://docs.python.org/3/library/dataclasses.html#frozen-instances).

```python
from pydantic import BaseModel, Field, ValidationError


class User(BaseModel):
    name: str = Field(frozen=True)
    age: int


user = User(name='John', age=42)

try:
    user.name = 'Jane'  # (1)!
except ValidationError as e:
    print(e)
    """
    1 validation error for User
    name
      Field is frozen [type=frozen_field, input_value='Jane', input_type=str]
    """
```

1. Так как поле `name` заморожено, присваивание запрещено.

## Исключение полей (Excluding fields)

Параметры `exclude` и `exclude_if` задают, какие поля не попадают в вывод при экспорте модели. Пример:

```python
from pydantic import BaseModel, Field


class User(BaseModel):
    name: str
    age: int = Field(exclude=True)


user = User(name='John', age=42)
print(user.model_dump())  # (1)!
#> {'name': 'John'}
```

1. Поле `age` не входит в результат [model_dump()](https://docs.pydantic.dev/latest/api/base_model/#pydantic.BaseModel.model_dump), так как исключено.

Подробнее: [раздел о сериализации](https://docs.pydantic.dev/latest/concepts/serialization/#field-inclusion-and-exclusion).

## Устаревшие поля (Deprecated fields)

Параметр `deprecated` помечает поле как устаревшее. В результате:

- В сгенерированной JSON Schema выставляется ключевое слово [deprecated](https://json-schema.org/draft/2020-12/json-schema-validation#section-9.3).
- При обращении к полю выдаётся предупреждение об устаревании (runtime).

Параметр может быть разных типов (см. ниже).

### deprecated как строка

Строка используется как сообщение об устаревании.

```python
from typing import Annotated

from pydantic import BaseModel, Field


class Model(BaseModel):
    deprecated_field: Annotated[int, Field(deprecated='This is deprecated')]


print(Model.model_json_schema()['properties']['deprecated_field'])
#> {'deprecated': True, 'title': 'Deprecated Field', 'type': 'integer'}
```

### deprecated через декоратор @warnings.deprecated

В качестве значения можно использовать декоратор [@warnings.deprecated](https://docs.python.org/3/library/warnings.html#warnings.deprecated) (или [бэкпорт из typing_extensions](https://typing-extensions.readthedocs.io/en/latest/index.html#typing_extensions.deprecated) для Python 3.12 и ниже).

**Python 3.9+** (typing_extensions):

```python
from typing import Annotated

from typing_extensions import deprecated

from pydantic import BaseModel, Field


class Model(BaseModel):
    deprecated_field: Annotated[int, deprecated('This is deprecated')]

    # Или явно через Field:
    alt_form: Annotated[int, Field(deprecated=deprecated('This is deprecated'))]
```

**Python 3.13+** (warnings):

```python
from typing import Annotated
from warnings import deprecated

from pydantic import BaseModel, Field


class Model(BaseModel):
    deprecated_field: Annotated[int, deprecated('This is deprecated')]

    # Или явно через Field:
    alt_form: Annotated[int, Field(deprecated=deprecated('This is deprecated'))]
```

**Поддержка `category` и `stacklevel`:** текущая реализация не учитывает аргументы `category` и `stacklevel` декоратора `deprecated`. Это может появиться в будущих версиях Pydantic.

### deprecated как булево значение

```python
from typing import Annotated

from pydantic import BaseModel, Field


class Model(BaseModel):
    deprecated_field: Annotated[int, Field(deprecated=True)]


print(Model.model_json_schema()['properties']['deprecated_field'])
#> {'deprecated': True, 'title': 'Deprecated Field', 'type': 'integer'}
```

**Обращение к устаревшему полю в валидаторах:** при доступе к устаревшему полю внутри валидатора предупреждение всё равно выдаётся. Его можно подавить через [catch_warnings](https://docs.python.org/3/library/warnings.html#warnings.catch_warnings):

```python
import warnings

from typing_extensions import Self

from pydantic import BaseModel, Field, model_validator


class Model(BaseModel):
    deprecated_field: int = Field(deprecated='This is deprecated')

    @model_validator(mode='after')
    def validate_model(self) -> Self:
        with warnings.catch_warnings():
            warnings.simplefilter('ignore', DeprecationWarning)
            self.deprecated_field = self.deprecated_field * 2
```

## Настройка JSON Schema (Customizing JSON Schema)

Некоторые параметры полей влияют только на генерируемую JSON Schema:

- `json_schema_extra`
- `examples`
- `description`
- `title`

Подробнее о настройке и изменении JSON Schema на уровне полей: [Customizing JSON Schema](https://docs.pydantic.dev/latest/concepts/json_schema/#field-level-customization).

## Декоратор computed_field (The computed_field decorator)

Документация API: [computed_field](https://docs.pydantic.dev/latest/api/fields/#pydantic.fields.computed_field)

Декоратор [computed_field](https://docs.pydantic.dev/latest/api/fields/#pydantic.fields.computed_field) подключает атрибуты-[property](https://docs.python.org/3/library/functions.html#property) или [cached_property](https://docs.python.org/3/library/functools.html#functools.cached_property) к сериализации модели или dataclass. Свойство также учитывается в JSON Schema (в режиме сериализации).

> Свойства удобны для полей, вычисляемых из других полей, или для дорогих вычислений (с кешированием через [cached_property](https://docs.python.org/3/library/functools.html#functools.cached_property)). Pydantic не выполняет дополнительной логики для обёрнутого свойства (валидации, инвалидации кеша и т.д.).
>
> Примечание

Пример JSON Schema (режим сериализации) для модели с вычисляемым полем:

```python
from pydantic import BaseModel, computed_field


class Box(BaseModel):
    width: float
    height: float
    depth: float

    @computed_field
    @property  # (1)!
    def volume(self) -> float:
        return self.width * self.height * self.depth


print(Box.model_json_schema(mode='serialization'))
"""
{
    'properties': {
        'width': {'title': 'Width', 'type': 'number'},
        'height': {'title': 'Height', 'type': 'number'},
        'depth': {'title': 'Depth', 'type': 'number'},
        'volume': {'readOnly': True, 'title': 'Volume', 'type': 'number'},
    },
    'required': ['width', 'height', 'depth', 'volume'],
    'title': 'Box',
    'type': 'object',
}
"""
```

1. Если не указать, [computed_field](https://docs.pydantic.dev/latest/api/fields/#pydantic.fields.computed_field) неявно превратит метод в [property](https://docs.python.org/3/library/functions.html#property). Явное использование [@property](https://docs.python.org/3/library/functions.html#property) предпочтительнее для проверки типов.

Пример с методом `model_dump` и вычисляемым полем:

```python
from pydantic import BaseModel, computed_field


class Box(BaseModel):
    width: float
    height: float
    depth: float

    @computed_field
    @property
    def volume(self) -> float:
        return self.width * self.height * self.depth


b = Box(width=1, height=2, depth=3)
print(b.model_dump())
#> {'width': 1.0, 'height': 2.0, 'depth': 3.0, 'volume': 6.0}
```

Вычисляемые поля, как и обычные, можно помечать как устаревшие:

```python
from typing_extensions import deprecated

from pydantic import BaseModel, computed_field


class Box(BaseModel):
    width: float
    height: float
    depth: float

    @computed_field
    @property
    @deprecated("'volume' is deprecated")
    def volume(self) -> float:
        return self.width * self.height * self.depth
```
