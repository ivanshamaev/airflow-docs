# Модели (Models)

Документация API: [pydantic.main.BaseModel](https://docs.pydantic.dev/latest/api/base_model/#pydantic.BaseModel)

Один из основных способов задать схему в Pydantic — **модели**. Модель — это класс, наследующий [BaseModel](https://docs.pydantic.dev/latest/api/base_model/#pydantic.BaseModel), с полями в виде аннотированных атрибутов.

Модели можно воспринимать как аналог структур в языках вроде C или как описание контракта одного эндпоинта API.

Модели во многом похожи на [dataclasses](https://docs.python.org/3/library/dataclasses.html#module-dataclasses) в Python, но спроектированы с отличиями, которые упрощают валидацию, сериализацию и генерацию JSON Schema. Подробнее в разделе [Dataclasses](https://docs.pydantic.dev/latest/concepts/dataclasses/).

Ненадёжные данные можно передать в модель; после разбора и валидации Pydantic гарантирует, что поля экземпляра модели будут соответствовать объявленным типам.

## Валидация — намерленная неточность термина

### Кратко

Под «валидацией» мы понимаем процесс создания экземпляра модели (или другого типа), соответствующего заданным типам и ограничениям. В быту это обычно и называют валидацией, хотя в других контекстах «валидация» может означать более узкую проверку.

---

### Подробнее

Путаница возникает из‑за того, что по смыслу Pydantic занимается не совсем тем, что в словаре называется «валидацией»:

**validation** (сущ.) — проверка или доказательство правильности/точности чего‑либо.

В Pydantic «валидация» — это создание экземпляра модели (или типа), соответствующего заданным типам и ограничениям. Pydantic гарантирует типы и ограничения **результата**, а не входных данных. Это видно по тому, что `ValidationError` выбрасывается, когда данные не удаётся успешно разобрать в экземпляр модели.

Различие может казаться мелким, но оно важно на практике. «Валидация» может включать копирование и приведение данных (в т.ч. копирование аргументов конструктора для приведения типов без изменения исходных данных). Подробнее в разделах Data Conversion и Attribute Copies ниже.

Итог: цель Pydantic — чтобы структура **после** обработки («валидации») точно соответствовала аннотациям типов. Поскольку в обиходе этот процесс называют валидацией, в документации мы используем тот же термин.

Раньше «parse» и «validation» использовались как синонимы; далее мы будем использовать только «validate», а «parse» — только в контексте [разбора JSON](https://docs.pydantic.dev/latest/concepts/json/).

## Базовое использование моделей

> Pydantic сильно опирается на типизацию Python. Полезные ссылки: [mypy](https://mypy.readthedocs.io/en/latest/), [Type System Guides](https://typing.readthedocs.io/en/latest/guides/index.html), раздел [Attribute copies](https://docs.pydantic.dev/latest/concepts/models/#attribute-copies) ниже.
>
> Примечание

```python
from pydantic import BaseModel, ConfigDict


class User(BaseModel):
    id: int
    name: str = 'Jane Doe'

    model_config = ConfigDict(str_max_length=10)  # (1)!
```

1. Модели Pydantic поддерживают разные [параметры конфигурации](https://docs.pydantic.dev/latest/concepts/config/) (см. [здесь](https://docs.pydantic.dev/latest/api/config/#pydantic.config.ConfigDict)).

В примере у `User` два поля:

- `id` — целое ([int](https://docs.python.org/3/library/functions.html#int)), обязательное;
- `name` — строка ([str](https://docs.python.org/3/library/stdtypes.html#str)), необязательная (есть значение по умолчанию).

Поддерживаемые типы описаны в разделе [types](https://docs.pydantic.dev/latest/concepts/types/). Поля можно настраивать через [Field()](https://docs.pydantic.dev/latest/api/fields/#pydantic.fields.Field), см. [документацию по полям](https://docs.pydantic.dev/latest/concepts/fields/).

Создание экземпляра:

```python
user = User(id='123')
```

`user` — экземпляр `User`. При инициализации выполняются разбор и валидация. Если не выброшено [ValidationError](https://docs.pydantic.dev/latest/api/pydantic_core/#pydantic_core.ValidationError), экземпляр валиден.

К полям обращаются как к атрибутам:

```python
assert user.name == 'Jane Doe'  # (1)!
assert user.id == 123  # (2)!
assert isinstance(user.id, int)
```

1. Строка `'123'` была приведена к целому `123`. Подробнее о приведении в разделе Data conversion.
2. `name` не задавался при создании `user`, использовано значение по умолчанию. Набор явно заданных при создании полей доступен в [model_fields_set](https://docs.pydantic.dev/latest/api/base_model/#pydantic.BaseModel.model_fields_set).

Сериализация — метод [model_dump()](https://docs.pydantic.dev/latest/api/base_model/#pydantic.BaseModel.model_dump):

```python
assert user.model_dump() == {'id': 123, 'name': 'Jane Doe'}
```

Вызов [dict](https://docs.python.org/3/reference/expressions.html#dict) от экземпляра тоже даёт словарь, но вложенные поля не рекурсивно превращаются в словари. У `model_dump()` есть аргументы для настройки сериализации.

По умолчанию модели изменяемы, поля можно менять через присваивание:

```python
user.id = 321
assert user.id == 321
```

> Избегайте совпадения имён полей с аннотацией типа. Например, следующий код ведёт себя неверно и даёт ошибку валидации:
>
> Предупреждение

```python
from typing import Optional

from pydantic import BaseModel


class Boo(BaseModel):
    int: Optional[int] = None


m = Boo(int=123)  # Валидация не пройдёт.
```

Из‑за того, как Python обрабатывает [аннотированные присваивания](https://docs.python.org/3/reference/simple_stmts.html#annassign), это эквивалентно `int: None = None`, что и вызывает ошибку.

### Методы и свойства моделей

У классов моделей есть методы и атрибуты:

- [model_rebuild()](https://docs.pydantic.dev/latest/api/base_model/#pydantic.BaseModel.model_rebuild): пересборка схемы модели (в т.ч. для рекурсивных дженериков). См. Rebuilding model schema.
- [model_post_init()](https://docs.pydantic.dev/latest/api/base_model/#pydantic.BaseModel.model_post_init): действия после создания модели и применения валидаторов полей.
- [model_parametrized_name()](https://docs.pydantic.dev/latest/api/base_model/#pydantic.BaseModel.model_parametrized_name): имя класса для параметризованных дженериков.
- [model_computed_fields](https://docs.pydantic.dev/latest/api/base_model/#pydantic.BaseModel.model_computed_fields): соответствие вычисляемых полей и их описаний (экземпляры [ComputedFieldInfo](https://docs.pydantic.dev/latest/api/fields/#pydantic.fields.ComputedFieldInfo)).
- [model_fields](https://docs.pydantic.dev/latest/api/base_model/#pydantic.BaseModel.model_fields): соответствие имён полей и их описаний ([FieldInfo](https://docs.pydantic.dev/latest/api/fields/#pydantic.fields.FieldInfo)).
- [model_json_schema()](https://docs.pydantic.dev/latest/api/base_model/#pydantic.BaseModel.model_json_schema): JSON Schema модели. См. [JSON Schema](https://docs.pydantic.dev/latest/concepts/json_schema/).
- [model_copy()](https://docs.pydantic.dev/latest/api/base_model/#pydantic.BaseModel.model_copy): копия модели (по умолчанию поверхностная). См. Model copy.
- [model_dump_json()](https://docs.pydantic.dev/latest/api/base_model/#pydantic.BaseModel.model_dump_json): JSON-строка из [model_dump()](https://docs.pydantic.dev/latest/api/base_model/#pydantic.BaseModel.model_dump). См. [Serialization](https://docs.pydantic.dev/latest/concepts/serialization/#json-mode).
- [model_dump()](https://docs.pydantic.dev/latest/api/base_model/#pydantic.BaseModel.model_dump): словарь полей и значений. См. [Serialization](https://docs.pydantic.dev/latest/concepts/serialization/#python-mode).
- [model_construct()](https://docs.pydantic.dev/latest/api/base_model/#pydantic.BaseModel.model_construct): создание модели без валидации. См. Creating models without validation.
- [model_validate_json()](https://docs.pydantic.dev/latest/api/base_model/#pydantic.BaseModel.model_validate_json): валидация переданных JSON-данных. См. Validating data.
- [model_validate()](https://docs.pydantic.dev/latest/api/base_model/#pydantic.BaseModel.model_validate): валидация переданного объекта. См. Validating data.

У экземпляров моделей есть атрибуты:

- [model_fields_set](https://docs.pydantic.dev/latest/api/base_model/#pydantic.BaseModel.model_fields_set): множество полей, явно переданных при инициализации.
- [model_extra](https://docs.pydantic.dev/latest/api/base_model/#pydantic.BaseModel.model_extra): дополнительные поля, установленные при валидации.

> Полный список методов и атрибутов см. в [API BaseModel](https://docs.pydantic.dev/latest/api/base_model/#pydantic.BaseModel).
>
> Примечание

> Изменения по сравнению с Pydantic V1: [Migration Guide](https://docs.pydantic.dev/latest/migration/) → [Changes to pydantic.BaseModel](https://docs.pydantic.dev/latest/migration/#changes-to-pydanticbasemodel).
>
> Совет

## Преобразование данных (Data conversion)

Pydantic может приводить входные данные к типам полей модели; иногда это ведёт к потере информации. Пример:

```python
from pydantic import BaseModel


class Model(BaseModel):
    a: int
    b: float
    c: str


print(Model(a=3.000, b='2.72', c=b'binary data').model_dump())
#> {'a': 3, 'b': 2.72, 'c': 'binary data'}
```

Это осознанное решение Pydantic, часто наиболее удобное. Подробнее: [обсуждение](https://github.com/pydantic/pydantic/issues/578).

Есть [строгий режим](https://docs.pydantic.dev/latest/concepts/strict_mode/), в котором преобразование не выполняется: значения должны уже иметь объявленный тип.

То же касается коллекций. Обычно лучше использовать конкретный тип (например, [list](https://docs.python.org/3/glossary.html#term-list)), а не абстрактный:

```python
from pydantic import BaseModel


class Model(BaseModel):
    items: list[int]  # (1)!


print(Model(items=(1, 2, 3)))
#> items=[1, 2, 3]
```

1. Можно было бы взять абстрактный [Sequence](https://docs.python.org/3/library/collections.abc.html#collections.abc.Sequence), чтобы допускать и списки, и кортежи. Pydantic сам приведёт кортеж к списку, так что в большинстве случаев в этом нет нужды. Абстрактные типы также могут [ухудшать производительность валидации](https://docs.pydantic.dev/latest/concepts/performance/#sequence-vs-list-or-tuple-with-mapping-vs-dict); конкретные типы коллекций позволяют избежать лишних проверок.

## Дополнительные данные (Extra data)

По умолчанию лишние данные не вызывают ошибки и просто игнорируются:

```python
from pydantic import BaseModel


class Model(BaseModel):
    x: int


m = Model(x=1, y='a')
assert m.model_dump() == {'x': 1}
```

Поведение задаётся параметром конфигурации [extra](https://docs.pydantic.dev/latest/api/config/#pydantic.config.ConfigDict.extra):

```python
from pydantic import BaseModel, ConfigDict


class Model(BaseModel):
    x: int

    model_config = ConfigDict(extra='allow')


m = Model(x=1, y='a')  # (1)!
assert m.model_dump() == {'x': 1, 'y': 'a'}
assert m.__pydantic_extra__ == {'y': 'a'}
```

1. При `extra='forbid'` это бы завершилось ошибкой.

Возможные значения:

- **`'allow'`** — лишние данные разрешены и сохраняются в `__pydantic_extra__`. Для них можно задать аннотацию и валидацию.
- **`'forbid'`** — лишние данные запрещены.
- **`'ignore'`** — лишние данные игнорируются (по умолчанию).

У методов валидации (например, [model_validate()](https://docs.pydantic.dev/latest/api/base_model/#pydantic.BaseModel.model_validate)) есть опциональный аргумент `extra`, переопределяющий настройку модели для этого вызова. Подробнее: [документация extra](https://docs.pydantic.dev/latest/api/config/#pydantic.config.ConfigDict.extra). Pydantic dataclasses тоже поддерживают extra (см. [dataclass configuration](https://docs.pydantic.dev/latest/concepts/dataclasses/#dataclass-configuration)).

## Вложенные модели (Nested models)

Иерархические структуры задают, используя модели в аннотациях типов.

**Python 3.9+** (с `Optional`):

```python
from typing import Optional

from pydantic import BaseModel

class Foo(BaseModel):
    count: int
    size: Optional[float] = None

class Bar(BaseModel):
    apple: str = 'x'
    banana: str = 'y'

class Spam(BaseModel):
    foo: Foo
    bars: list[Bar]

m = Spam(foo={'count': 4}, bars=[{'apple': 'x1'}, {'apple': 'x2'}])
print(m)
print(m.model_dump())
```

**Python 3.10+** (синтаксис `| None`):

```python
from pydantic import BaseModel


class Foo(BaseModel):
    count: int
    size: float | None = None


class Bar(BaseModel):
    apple: str = 'x'
    banana: str = 'y'


class Spam(BaseModel):
    foo: Foo
    bars: list[Bar]


m = Spam(foo={'count': 4}, bars=[{'apple': 'x1'}, {'apple': 'x2'}])
print(m)
# foo=Foo(count=4, size=None) bars=[Bar(apple='x1', banana='y'), Bar(apple='x2', banana='y')]
print(m.model_dump())
# {'foo': {'count': 4, 'size': None}, 'bars': [{'apple': 'x1', 'banana': 'y'}, {'apple': 'x2', 'banana': 'y'}]}
```

Поддерживаются самоссылающиеся модели. См. [forward annotations](https://docs.pydantic.dev/latest/concepts/forward_annotations/#self-referencing-or-recursive-models).

## Пересборка схемы модели (Rebuilding model schema)

При определении класса модели Pydantic анализирует его тело и строит core schema. Аннотации типов вычисляются при создании класса. Если в аннотациях используются символы, ещё не определённые в момент создания класса, используйте [model_rebuild()](https://docs.pydantic.dev/latest/api/base_model/#pydantic.BaseModel.model_rebuild):

```python
from pydantic import BaseModel, PydanticUserError


class Foo(BaseModel):
    x: 'Bar'  # (1)!


try:
    Foo.model_json_schema()
except PydanticUserError as e:
    print(e)
    # `Foo` is not fully defined; you should define `Bar`, then call `Foo.model_rebuild()`.


class Bar(BaseModel):
    pass


Foo.model_rebuild()
print(Foo.model_json_schema())
# {'$defs': {'Bar': {...}}, 'properties': {'x': {'$ref': '#/$defs/Bar'}}, 'required': ['x'], ...}
```

1. `Bar` ещё не определён при создании `Foo`, поэтому используется [forward annotation](https://docs.pydantic.dev/latest/concepts/forward_annotations/).

Pydantic пытается определять необходимость пересборки автоматически, но при рекурсивных или дженерик-моделях лучше вызывать [model_rebuild()](https://docs.pydantic.dev/latest/api/base_model/#pydantic.BaseModel.model_rebuild) явно.

В V2 `model_rebuild()` заменил `update_forward_refs()` из V1. При вызове на внешней модели пересобирается схема всей иерархии, поэтому все типы на всех уровнях должны быть определены до вызова.

## Валидация данных (Validating data)

Pydantic может валидировать данные в трёх режимах: Python, JSON и strings.

**Режим Python** используется при:

- [model_validate()](https://docs.pydantic.dev/latest/api/base_model/#pydantic.BaseModel.model_validate): данные — словарь или экземпляр модели (по умолчанию экземпляры считаются валидными; см. [revalidate_instances](https://docs.pydantic.dev/latest/api/config/#pydantic.config.ConfigDict.revalidate_instances)). При явном включении можно передавать произвольные объекты.
- Конструкторе `__init__()` модели. Значения полей передаются только именованными аргументами.

**Режимы JSON и strings** — отдельные методы:

- [model_validate_strings()](https://docs.pydantic.dev/latest/api/base_model/#pydantic.BaseModel.model_validate_strings): данные — словарь (возможно вложенный) со строковыми ключами и значениями; строки приводятся к нужным типам как в JSON-режиме.
- [model_validate_json()](https://docs.pydantic.dev/latest/api/base_model/#pydantic.BaseModel.model_validate_json): данные — JSON-строка или `bytes`. Для JSON-нагрузки обычно быстрее, чем ручной разбор в словарь. См. [JSON](https://docs.pydantic.dev/latest/concepts/json/).

У методов `model_validate_*()` можно задавать параметры валидации (строгость, лишние данные, [контекст валидации](https://docs.pydantic.dev/latest/concepts/validators/#validation-context) и т.д.).

> Поведение режимов Python и JSON может различаться (например, по [строгости](https://docs.pydantic.dev/latest/concepts/strict_mode/)). Если данные не из JSON, но нужна та же логика, что в JSON-режиме: сериализуйте данные в JSON ([json.dumps()](https://docs.python.org/3/library/json.html#json.dumps)) или используйте [model_validate_strings()](https://docs.pydantic.dev/latest/api/base_model/#pydantic.BaseModel.model_validate_strings) для словарей со строковыми ключами и значениями. Прогресс: [issue](https://github.com/pydantic/pydantic/issues/11154).
>
> Примечание

**Python 3.10+:**

```python
from datetime import datetime

from pydantic import BaseModel, ValidationError


class User(BaseModel):
    id: int
    name: str = 'John Doe'
    signup_ts: datetime | None = None


m = User.model_validate({'id': 123, 'name': 'James'})
print(m)
#> id=123 name='James' signup_ts=None

try:
    m = User.model_validate_json('{"id": 123, "name": 123}')
except ValidationError as e:
    print(e)
    """
    1 validation error for User
    name
      Input should be a valid string [type=string_type, input_value=123, input_type=int]
    """

m = User.model_validate_strings({'id': '123', 'name': 'James'})
print(m)
#> id=123 name='James' signup_ts=None

m = User.model_validate_strings(
    {'id': '123', 'name': 'James', 'signup_ts': '2024-04-01T12:00:00'}
)
print(m)
#> id=123 name='James' signup_ts=datetime.datetime(2024, 4, 1, 12, 0)

try:
    m = User.model_validate_strings(
        {'id': '123', 'name': 'James', 'signup_ts': '2024-04-01'}, strict=True
    )
except ValidationError as e:
    print(e)
    """
    1 validation error for User
    signup_ts
      Input should be a valid datetime, invalid datetime separator, expected `T`, `t`, `_` or space [type=datetime_parsing, input_value='2024-04-01', input_type=str]
    """
```

### Создание моделей без валидации

[model_construct()](https://docs.pydantic.dev/latest/api/base_model/#pydantic.BaseModel.model_construct) создаёт модель без валидации. Полезно, когда:

- валидаторы имеют побочные эффекты, которые не нужны;
- валидаторы неидемпотентны;
- данные уже заведомо валидны (ради производительности).

> [model_construct()](https://docs.pydantic.dev/latest/api/base_model/#pydantic.BaseModel.model_construct) не выполняет валидации и может создать невалидную модель. Используйте его только с уже провалидированными или доверенными данными.
>
> Предупреждение

> В Pydantic V2 разница в скорости между обычной валидацией и [model_construct()](https://docs.pydantic.dev/latest/api/base_model/#pydantic.BaseModel.model_construct) уменьшилась. Для простых моделей валидация может быть быстрее. При использовании `model_construct()` ради скорости стоит замерить свой сценарий.
>
> Примечание

Для root-моделей корневое значение в [model_construct()](https://docs.pydantic.dev/latest/api/base_model/#pydantic.BaseModel.model_construct) можно передать позиционно.

Поведение `model_construct()`:

- `__init__` модели и родительских классов не вызывается.
- Для моделей с приватными атрибутами `__pydantic_private__` заполняется так же, как при валидации.
- Для полей с значениями по умолчанию они подставляются, если аргумент не передан.
- Словари во вложенные модели не преобразуются — при необходимости делайте это сами.

Поведение при лишних данных:

- При `extra='forbid'` вызов `model_construct()` с лишними данными не вызывает ошибку, данные игнорируются.
- При `extra='ignore'` лишние данные не попадают в `__pydantic_extra__` и `__dict__`.
- При `extra='allow'` лишние данные сохраняются в `__pydantic_extra__` и в `__dict__`.

### Собственный __init__()

У моделей Pydantic есть реализация `__init__()` по умолчанию (вызывается только из конструктора, не из `model_validate_*()`), которая делегирует валидацию в pydantic-core.

Можно задать свой `__init__()`. Тогда он будет вызываться из всех методов валидации без выполнения валидации (в реализации нужно вызывать `super().__init__(**kwargs)`).

Собственный `__init__()` не рекомендуется: теряются параметры валидации (строгость, лишние данные, контекст). Для действий после инициализации используйте after [field](https://docs.pydantic.dev/latest/concepts/validators/#field-after-validator) или [model](https://docs.pydantic.dev/latest/concepts/validators/#model-after-validator) валидаторы либо [model_post_init()](https://docs.pydantic.dev/latest/api/base_model/#pydantic.BaseModel.model_post_init):

```python
import logging
from typing import Any

from pydantic import BaseModel


class MyModel(BaseModel):
    id: int

    def model_post_init(self, context: Any) -> None:
        logging.info("Model initialized with id %d", self.id)
```

## Обработка ошибок (Error handling)

При ошибках в данных Pydantic выбрасывает [ValidationError](https://docs.pydantic.dev/latest/api/pydantic_core/#pydantic_core.ValidationError). Одно исключение содержит информацию обо всех найденных ошибках. Подробнее: [Error Handling](https://docs.pydantic.dev/latest/errors/errors/).

Пример:

```python
from pydantic import BaseModel, ValidationError


class Model(BaseModel):
    list_of_ints: list[int]
    a_float: float


data = {'list_of_ints': ['1', 2, 'bad'], 'a_float': 'not a float'}

try:
    Model(**data)
except ValidationError as e:
    print(e)
    """
    2 validation errors for Model
    list_of_ints.2
      Input should be a valid integer, unable to parse string as an integer [type=int_parsing, input_value='bad', input_type=str]
    a_float
      Input should be a valid number, unable to parse string as a number [type=float_parsing, input_value='not a float', input_type=str]
    """
```

## Произвольные экземпляры классов (Arbitrary class instances)

(Ранее «ORM Mode» / `from_orm()`.)

При использовании [model_validate()](https://docs.pydantic.dev/latest/api/base_model/#pydantic.BaseModel.model_validate) Pydantic может валидировать произвольные объекты, читая атрибуты по именам полей. Часто это используют для интеграции с ORM.

Включение: конфиг [from_attributes](https://docs.pydantic.dev/latest/api/config/#pydantic.config.ConfigDict.from_attributes) или параметр `from_attributes` в `model_validate()`. Пример с [SQLAlchemy](https://www.sqlalchemy.org/):

```python
from typing import Annotated

from sqlalchemy import ARRAY, String
from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column

from pydantic import BaseModel, ConfigDict, StringConstraints


class Base(DeclarativeBase):
    pass


class CompanyOrm(Base):
    __tablename__ = 'companies'
    id: Mapped[int] = mapped_column(primary_key=True, nullable=False)
    public_key: Mapped[str] = mapped_column(String(20), index=True, nullable=False, unique=True)
    domains: Mapped[list[str]] = mapped_column(ARRAY(String(255)))


class CompanyModel(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    id: int
    public_key: Annotated[str, StringConstraints(max_length=20)]
    domains: list[Annotated[str, StringConstraints(max_length=255)]]


co_orm = CompanyOrm(id=123, public_key='foobar', domains=['example.com', 'foobar.com'])
co_model = CompanyModel.model_validate(co_orm)
# id=123 public_key='foobar' domains=['example.com', 'foobar.com']
```

### Вложенные атрибуты

Экземпляры моделей создаются и из верхнеуровневых, и из вложенных атрибутов. Пример:

```python
from pydantic import BaseModel, ConfigDict


class PetCls:
    def __init__(self, *, name: str) -> None:
        self.name = name


class PersonCls:
    def __init__(self, *, name: str, pets: list[PetCls]) -> None:
        self.name = name
        self.pets = pets


class Pet(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    name: str


class Person(BaseModel):
    model_config = ConfigDict(from_attributes=True)
    name: str
    pets: list[Pet]


bones = PetCls(name='Bones')
orion = PetCls(name='Orion')
anna = PersonCls(name='Anna', pets=[bones, orion])
anna_model = Person.model_validate(anna)
print(anna_model)
#> name='Anna' pets=[Pet(name='Bones'), Pet(name='Orion')]
```

## Копирование модели (Model copy)

[model_copy()](https://docs.pydantic.dev/latest/api/base_model/#pydantic.BaseModel.model_copy) создаёт копию модели (с опциональными обновлениями), удобно для замороженных моделей.

```python
from pydantic import BaseModel


class BarModel(BaseModel):
    whatever: int


class FooBarModel(BaseModel):
    banana: float
    foo: str
    bar: BarModel


m = FooBarModel(banana=3.14, foo='hello', bar={'whatever': 123})

print(m.model_copy(update={'banana': 0}))
#> banana=0 foo='hello' bar=BarModel(whatever=123)

# обычная копия — тот же объект для bar
print(id(m.bar) == id(m.model_copy().bar))
#> True
# глубокая копия — новый объект для bar
print(id(m.bar) == id(m.model_copy(deep=True).bar))
#> False
```

## Дженерик-модели (Generic models)

Pydantic поддерживает дженерик-модели для переиспользования общей структуры. Поддерживаются и [новый синтаксис параметров типов](https://docs.python.org/3/reference/compound_stmts.html#type-params) (PEP 695, Python 3.12), и [старый](https://docs.python.org/3/library/typing.html#building-generic-types-and-type-aliases).

Пример обёртки ответа API (Python 3.9+):

```python
from typing import Generic, TypeVar

from pydantic import BaseModel, ValidationError

DataT = TypeVar('DataT')


class DataModel(BaseModel):
    number: int


class Response(BaseModel, Generic[DataT]):
    data: DataT


print(Response[int](data=1))
#> data=1
print(Response[str](data='value'))
#> data='value'
print(Response[str](data='value').model_dump())
#> {'data': 'value'}

data = DataModel(number=1)
print(Response[DataModel](data=data).model_dump())
#> {'data': {'number': 1}}
try:
    Response[int](data='value')
except ValidationError as e:
    print(e)
```

Python 3.12+ (новый синтаксис): `class Response[DataT](BaseModel):` с type parameters.

Наследование от дженерик-модели с сохранением дженерика — подкласс должен наследовать `Generic`:

```python
from typing import Generic, TypeVar
from pydantic import BaseModel

TypeX = TypeVar('TypeX')

class BaseClass(BaseModel, Generic[TypeX]):
    X: TypeX

class ChildClass(BaseClass[TypeX], Generic[TypeX]):
    pass

print(ChildClass[int](X=1))
#> X=1
```

Подкласс может частично или полностью подставлять type variables:

```python
class ChildClass(BaseClass[int, TypeY], Generic[TypeY, TypeZ]):
    z: TypeZ
# ChildClass[str, int](x='1', y='y', z='3') -> x=1 y='y' z=3
```

Переопределение имени параметризованного класса — [model_parametrized_name()](https://docs.pydantic.dev/latest/api/base_model/#pydantic.BaseModel.model_parametrized_name): например, `IntResponse(data=1)`, `StrResponse(data='a')`. Параметризованные дженерик-модели можно использовать как типы полей в других моделях (`Order` с `product: ResponseModel[Product]`). Один и тот же type variable во вложенных моделях связывает типы на разных уровнях (`InnerT[T]`, `OuterT[T]`).

Для наследования от дженерик-модели с сохранением дженерика подкласс должен наследовать [Generic](https://docs.python.org/3/library/typing.html#typing.Generic). Имя параметризованных подклассов можно задать через [model_parametrized_name()](https://docs.pydantic.dev/latest/api/base_model/#pydantic.BaseModel.model_parametrized_name).

> При параметризации модели конкретным типом Pydantic не проверяет, что тип [совместим с верхней границей](https://typing.readthedocs.io/en/latest/spec/generics.html#type-variables-with-an-upper-bound) type variable.
>
> Предупреждение

Конфигурация, валидаторы и логика сериализации дженерик-модели применяются к параметризованным классам. Внутри Pydantic создаёт подклассы дженерик-модели при параметризации и кэширует их.

**Валидация непараметризованных type variables:** для неограниченных переменных используется [Any](https://docs.python.org/3/library/typing.html#typing.Any); при default (PEP 696) или bound/constraints — соответствующий тип. Пример с `T`, `U` (bound=int), `V` (default=str): `Model(t='t', u=1, v='v')` валидно; при `ItemHolder` с `ItemT` bound на `ItemBase` без параметризации поле `item` валидируется как `ItemBase()` и теряет поля подтипа — нужно явно `ItemHolder[IntItem](**loaded_data)`.

**Сериализация непараметризованных type variables:** при upper bound без параметризации валидация идёт по верхней границе, сериализация — как Any (все поля сохраняются). При явной параметризации `Error[ErrorDetails]` в вывод попадают только поля объявленного типа. При constraints или default тип по умолчанию используется и для валидации, и для сериализации; переопределить можно через [SerializeAsAny](https://docs.pydantic.dev/latest/concepts/serialization/#serializeasany-annotation).

> Не используйте параметризованные дженерики в [isinstance()](https://docs.python.org/3/library/functions.html#isinstance) (например, `isinstance(my_model, MyGenericModel[int])`). Для проверки типа создайте подкласс: `class MyIntModel(MyGenericModel[int]): ...` и проверяйте `isinstance(my_model, MyIntModel)`.
>
> Предупреждение

## Динамическое создание моделей (Dynamic model creation)

[create_model()](https://docs.pydantic.dev/latest/api/base_model/#pydantic.create_model) создаёт модель по полям, заданным в runtime:

```python
from pydantic import BaseModel, create_model

DynamicFoobarModel = create_model('DynamicFoobarModel', foo=str, bar=(int, 123))
# Эквивалент классу с полями foo: str и bar: int = 123
```

Поля задаются именованными аргументами: либо пара (тип, значение по умолчанию или Field()), либо один тип. Расширенный пример с `Field`, `PrivateAttr`:

```python
from typing import Annotated

from pydantic import BaseModel, Field, PrivateAttr, create_model

DynamicModel = create_model(
    'DynamicModel',
    foo=(str, Field(alias='FOO')),
    bar=Annotated[str, Field(description='Bar field')],
    _private=(int, PrivateAttr(default=1)),
)
```

Специальные аргументы `__config__` и `__base__` настраивают модель; `__base__` задаёт базовую модель:

```python
from pydantic import BaseModel, create_model

class FooModel(BaseModel):
    foo: str
    bar: int = 123

BarModel = create_model(
    'BarModel',
    apple=(str, 'russet'),
    banana=(str, 'yellow'),
    __base__=FooModel,
)
print(BarModel.model_fields.keys())
#> dict_keys(['foo', 'bar', 'apple', 'banana'])
```

Валидаторы передаются в `__validators__`:

```python
from pydantic import ValidationError, create_model, field_validator

def alphanum(cls, v):
    assert v.isalnum(), 'must be alphanumeric'
    return v

validators = {'username_validator': field_validator('username')(alphanum)}
UserModel = create_model('UserModel', username=(str, ...), __validators__=validators)
user = UserModel(username='scolvin')
# UserModel(username='scolvi%n') вызовет ValidationError
```

> Для pickle динамической модели нужны `__module__` и глобальное определение модели.
>
> Примечание

> Функция может выполнять произвольный код в аннотациях при разрешении строковых ссылок. См. [Security implications of introspecting annotations](https://docs.python.org/3/library/annotationlib.html#annotationlib-security).
>
> Предупреждение

См. также [пример динамических моделей](https://docs.pydantic.dev/latest/examples/dynamic_models/).

## RootModel и пользовательский корневой тип

Модели с «пользовательским корневым типом» задаются наследованием от [pydantic.RootModel](https://docs.pydantic.dev/latest/api/root_model/#pydantic.root_model.RootModel). Корневой тип задаётся дженерик-параметром; корневое значение передаётся первым (и единственным) аргументом в `__init__` или [model_validate](https://docs.pydantic.dev/latest/api/base_model/#pydantic.BaseModel.model_validate).

Пример:

```python
from pydantic import RootModel

Pets = RootModel[list[str]]
PetsByName = RootModel[dict[str, str]]

print(Pets(['dog', 'cat']))
#> root=['dog', 'cat']
print(Pets(['dog', 'cat']).model_dump_json())
#> ["dog","cat"]
print(Pets.model_validate(['dog', 'cat']))
print(Pets.model_json_schema())
# {'items': {'type': 'string'}, 'title': 'RootModel[list[str]]', 'type': 'array'}

print(PetsByName({'Otis': 'dog', 'Milo': 'cat'}))
#> root={'Otis': 'dog', 'Milo': 'cat'}
print(PetsByName.model_validate({'Otis': 'dog', 'Milo': 'cat'}))
```

Для доступа к элементам `root` по индексу или итерации реализуйте `__iter__` и `__getitem__`:

```python
from pydantic import RootModel


class Pets(RootModel):
    root: list[str]

    def __iter__(self):
        return iter(self.root)

    def __getitem__(self, item):
        return self.root[item]


pets = Pets.model_validate(['dog', 'cat'])
print(pets[0])
#> dog
print([pet for pet in pets])
#> ['dog', 'cat']
```

Подкласс параметризованной RootModel:

```python
from pydantic import RootModel

class Pets(RootModel[list[str]]):
    def describe(self) -> str:
        return f'Pets: {", ".join(self.root)}'

my_pets = Pets.model_validate(['dog', 'cat'])
print(my_pets.describe())
#> Pets: dog, cat
```

## Псевдо-неизменяемость (Faux immutability)

Неизменяемость задаётся через `model_config['frozen'] = True`. Попытка изменить атрибут приведёт к ошибке. В V1 использовалось `allow_mutation = False` (устарело в V2).

> В Python неизменяемость не гарантируется на уровне языка; изменяемые вложенные объекты (например, dict) по-прежнему можно менять.
>
> Предупреждение

```python
from pydantic import BaseModel, ConfigDict, ValidationError


class FooBarModel(BaseModel):
    model_config = ConfigDict(frozen=True)
    a: str
    b: dict


foobar = FooBarModel(a='hello', b={'apple': 'pear'})

try:
    foobar.a = 'different'
except ValidationError as e:
    print(e)
    # Instance is frozen

print(foobar.a)
#> hello
foobar.b['apple'] = 'grape'  # dict изменяем, изменение допустимо
print(foobar.b)
#> {'apple': 'grape'}
```

## Абстрактные базовые классы (Abstract base classes)

Модели Pydantic можно сочетать с [ABC](https://docs.python.org/3/library/abc.html): наследуйте модель от `abc.ABC` и помечайте методы `@abc.abstractmethod`.

## Порядок полей (Field ordering)

Порядок полей сохраняется при [сериализации](https://docs.pydantic.dev/latest/concepts/serialization/#serializing-data), в ошибках валидации и в [JSON Schema](https://docs.pydantic.dev/latest/concepts/json_schema/) модели.

```python
from pydantic import BaseModel, ValidationError

class Model(BaseModel):
    a: int
    b: int = 2
    c: int = 1
    d: int = 0
    e: float

print(Model.model_fields.keys())
#> dict_keys(['a', 'b', 'c', 'd', 'e'])
m = Model(e=2, a=1)
print(m.model_dump())
#> {'a': 1, 'b': 2, 'c': 1, 'd': 0, 'e': 2.0}
try:
    Model(a='x', b='x', c='x', d='x', e='x')
except ValidationError as err:
    error_locations = [e['loc'] for e in err.errors()]
print(error_locations)
#> [('a',), ('b',), ('c',), ('d',), ('e',)]
```

## Автоматически исключаемые атрибуты

### Переменные класса

Атрибуты с аннотацией [ClassVar](https://docs.python.org/3/library/typing.html#typing.ClassVar) считаются переменными класса и не становятся полями экземпляра.

```python
from typing import ClassVar

from pydantic import BaseModel

class Model(BaseModel):
    x: ClassVar[int] = 1
    y: int = 2

m = Model()
print(m)
#> y=2
print(Model.x)
#> 1
```

### Приватные атрибуты модели

Атрибуты с ведущим подчёркиванием не считаются полями и не входят в схему. Они превращаются в «приватные атрибуты» и не валидируются и не устанавливаются в `__init__`/`model_validate`. Задаются через [PrivateAttr](https://docs.pydantic.dev/latest/api/fields/#pydantic.fields.PrivateAttr). Имена с двойным подчёркиванием (`__attr__`) не поддерживаются и игнорируются.

```python
from datetime import datetime
from random import randint
from typing import Any

from pydantic import BaseModel, PrivateAttr

class TimeAwareModel(BaseModel):
    _processed_at: datetime = PrivateAttr(default_factory=datetime.now)
    _secret_value: str

    def model_post_init(self, context: Any) -> None:
        self._secret_value = randint(1, 5)

m = TimeAwareModel()
print(m._processed_at)
print(m._secret_value)
```

## Сигнатура модели (Model signature)

Сигнатура модели генерируется по полям и подходит для интроспекции и библиотек вроде FastAPI и hypothesis.

```python
import inspect
from pydantic import BaseModel, Field

class FooModel(BaseModel):
    id: int
    name: str = None
    description: str = 'Foo'
    apple: int = Field(alias='pear')

print(inspect.signature(FooModel))
#> (*, id: int, name: str = None, description: str = 'Foo', pear: int) -> None
```

Учитывается и кастомный `__init__`. В сигнатуру попадают alias или имя поля, если это валидный идентификатор; при `model_config['extra'] == 'allow'` в сигнатуре всегда есть `**data`.

## Сопоставление структур (Structural pattern matching)

Поддерживается структурное сопоставление образцов (PEP 636, Python 3.10):

```python
from pydantic import BaseModel

class Pet(BaseModel):
    name: str
    species: str

a = Pet(name='Bones', species='dog')

match a:
    case Pet(species='dog', name=dog_name):
        print(f'{dog_name} is a dog')
#> Bones is a dog
    case _:
        print('No dog matched')
```

Это синтаксический сахар для доступа к атрибутам и сравнения/присваивания, не создаёт новую модель.

## Копирование атрибутов (Attribute copies)

Во многих случаях аргументы конструктора копируются для валидации и приведения типов.

```python
from pydantic import BaseModel

class C1:
    arr = []
    def __init__(self, in_arr):
        self.arr = in_arr

class C2(BaseModel):
    arr: list[int]

arr_orig = [1, 9, 10, 3]
c1 = C1(arr_orig)
c2 = C2(arr=arr_orig)
print(f'{id(c1.arr) == id(c2.arr)=}')
#> id(c1.arr) == id(c2.arr)=False
```

В части случаев (например, при передаче моделей) копирование не выполняется; переопределить можно через [model_config['revalidate_instances'] = 'always'](https://docs.pydantic.dev/latest/api/config/#pydantic.config.ConfigDict).

---

[← Добро пожаловать в Pydantic](welcome-to-pydantic.md) | [К содержанию](README.md)
