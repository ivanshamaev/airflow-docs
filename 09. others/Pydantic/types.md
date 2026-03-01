# Типы (Types)

Pydantic использует типы, чтобы задать, как выполнять валидацию и сериализацию. [Встроенные и стандартные типы](https://docs.pydantic.dev/latest/api/standard_library_types/) (например [int](https://docs.python.org/3/library/functions.html#int), [str](https://docs.python.org/3/library/stdtypes.html#str), [date](https://docs.python.org/3/library/datetime.html#datetime.date)) можно использовать как есть. Для них можно задавать [строгость](https://docs.pydantic.dev/latest/concepts/strict_mode/) и ограничения.

Помимо этого Pydantic предоставляет дополнительные типы — [в самой библиотеке](https://docs.pydantic.dev/latest/api/types/) (например [SecretStr](https://docs.pydantic.dev/latest/api/types/#pydantic.types.SecretStr)) или во внешней [pydantic-extra-types](https://github.com/pydantic/pydantic-extra-types). Они реализованы по паттернам из раздела о кастомных типах. Строгость и ограничения к ним не применяются.

В [документации по встроенным и стандартным типам](https://docs.pydantic.dev/latest/api/standard_library_types/) описаны поддерживаемые типы: допустимые значения, ограничения валидации и возможность настройки [строгости](https://docs.pydantic.dev/latest/concepts/strict_mode/).

См. также [таблицу преобразований](https://docs.pydantic.dev/latest/concepts/conversion_table/) — сводку допустимых значений по типам.

Ниже речь пойдёт о том, как определять собственные кастомные типы.

## Кастомные типы (Custom Types)

Задать кастомный тип можно несколькими способами.

### Аннотированный паттерн (Using the annotated pattern)

[Аннотированный паттерн](https://docs.pydantic.dev/latest/concepts/fields/#the-annotated-pattern) позволяет делать типы переиспользуемыми. Например, тип «положительное целое»:

```python
from typing import Annotated

from pydantic import Field, TypeAdapter, ValidationError

PositiveInt = Annotated[int, Field(gt=0)]  # (1)!

ta = TypeAdapter(PositiveInt)

print(ta.validate_python(1))
#> 1

try:
    ta.validate_python(-1)
except ValidationError as exc:
    print(exc)
    """
    1 validation error for constrained-int
      Input should be greater than 0 [type=greater_than, input_value=-1, input_type=int]
    """
```

Ограничения можно брать из библиотеки [annotated-types](https://github.com/annotated-types/annotated-types), чтобы не привязываться к Pydantic:

```python
from annotated_types import Gt

PositiveInt = Annotated[int, Gt(0)]
```

1. Доступ к имени поля — см. раздел Access to field name ниже.

#### Добавление валидации и сериализации (Adding validation and serialization)

Валидацию, сериализацию и JSON Schema для произвольного типа можно добавлять или переопределять маркерами Pydantic:

```python
from typing import Annotated

from pydantic import (
    AfterValidator,
    PlainSerializer,
    TypeAdapter,
    WithJsonSchema,
)

TruncatedFloat = Annotated[
    float,
    AfterValidator(lambda x: round(x, 1)),
    PlainSerializer(lambda x: f'{x:.1e}', return_type=str),
    WithJsonSchema({'type': 'string'}, mode='serialization'),
]


ta = TypeAdapter(TruncatedFloat)

input = 1.02345
assert input != 1.0

assert ta.validate_python(input) == 1.0

assert ta.dump_json(input) == b'"1.0e+00"'

assert ta.json_schema(mode='validation') == {'type': 'number'}
assert ta.json_schema(mode='serialization') == {'type': 'string'}
```

#### Дженерики (Generics)

[Переменные типов](https://docs.python.org/3/library/typing.html#typing.TypeVar) можно использовать внутри [Annotated](https://docs.python.org/3/library/typing.html#typing.Annotated):

```python
from typing import Annotated, TypeVar

from annotated_types import Gt, Len

from pydantic import TypeAdapter, ValidationError

T = TypeVar('T')


ShortList = Annotated[list[T], Len(max_length=4)]


ta = TypeAdapter(ShortList[int])

v = ta.validate_python([1, 2, 3, 4])
assert v == [1, 2, 3, 4]

try:
    ta.validate_python([1, 2, 3, 4, 5])
except ValidationError as exc:
    print(exc)
    """
    1 validation error for list[int]
      List should have at most 4 items after validation, not 5 [type=too_long, input_value=[1, 2, 3, 4, 5], input_type=list]
    """


PositiveList = list[Annotated[T, Gt(0)]]

ta = TypeAdapter(PositiveList[float])

v = ta.validate_python([1.0])
assert type(v[0]) is float


try:
    ta.validate_python([-1.0])
except ValidationError as exc:
    print(exc)
    """
    1 validation error for list[constrained-float]
    0
      Input should be greater than 0 [type=greater_than, input_value=-1.0, input_type=float]
    """
```

### Именованные алиасы типов (Named type aliases)

В примерах выше использовались неявные алиасы типов (присвоенные переменной). В runtime Pydantic не знает имя этой переменной, из‑за чего:

- в большинстве случаев рекурсивные алиасы типов не работают;
- [JSON Schema](https://docs.pydantic.dev/latest/concepts/json_schema/) алиаса не превращается в [definition](https://json-schema.org/understanding-json-schema/structuring#defs) (удобно, когда алиас используется в модели больше одного раза).

С помощью [type statement](https://typing.readthedocs.io/en/latest/spec/aliases.html#type-statement) ([PEP 695](https://peps.python.org/pep-0695/)) алиасы можно задать так:

**Python 3.9+** (TypeAliasType из typing_extensions):

```python
from typing import Annotated

from annotated_types import Gt
from typing_extensions import TypeAliasType

from pydantic import BaseModel

PositiveIntList = TypeAliasType('PositiveIntList', list[Annotated[int, Gt(0)]])


class Model(BaseModel):
    x: PositiveIntList
    y: PositiveIntList


print(Model.model_json_schema())  # (1)!
"""
{
    '$defs': {
        'PositiveIntList': {
            'items': {'exclusiveMinimum': 0, 'type': 'integer'},
            'type': 'array',
        }
    },
    'properties': {
        'x': {'$ref': '#/$defs/PositiveIntList'},
        'y': {'$ref': '#/$defs/PositiveIntList'},
    },
    'required': ['x', 'y'],
    'title': 'Model',
    'type': 'object',
}
"""
```

1. Если бы `PositiveIntList` был неявным алиасом, его определение дублировалось бы в `'x'` и `'y'`.

**Python 3.12+** (новый синтаксис):

```python
from typing import Annotated

from annotated_types import Gt

from pydantic import BaseModel

type PositiveIntList = list[Annotated[int, Gt(0)]]


class Model(BaseModel):
    x: PositiveIntList
    y: PositiveIntList


print(Model.model_json_schema())  # (1)!
"""
{
    '$defs': {
        'PositiveIntList': {
            'items': {'exclusiveMinimum': 0, 'type': 'integer'},
            'type': 'array',
        }
    },
    'properties': {
        'x': {'$ref': '#/$defs/PositiveIntList'},
        'y': {'$ref': '#/$defs/PositiveIntList'},
    },
    'required': ['x', 'y'],
    'title': 'Model',
    'type': 'object',
}
"""
```

1. Если бы `PositiveIntList` был неявным алиасом, его определение дублировалось бы в `'x'` и `'y'`.

**Когда использовать именованные алиасы**

Для статических анализаторов (именованные) алиасы по PEP 695 и неявные алиасы эквивалентны, но Pydantic не обрабатывает метаданные уровня поля внутри именованных алиасов. То есть метаданные вроде `alias`, `default`, `deprecated` использовать нельзя:

**Python 3.9+**:

```python
from typing import Annotated

from typing_extensions import TypeAliasType

from pydantic import BaseModel, Field

MyAlias = TypeAliasType('MyAlias', Annotated[int, Field(default=1)])


class Model(BaseModel):
    x: MyAlias  # Так нельзя
```

**Python 3.12+**:

```python
from typing import Annotated

from pydantic import BaseModel, Field

type MyAlias = Annotated[int, Field(default=1)]


class Model(BaseModel):
    x: MyAlias  # Так нельзя
```

Допускаются только метаданные, применимые к самому аннотированному типу (например [ограничения валидации](https://docs.pydantic.dev/latest/concepts/fields/#field-constraints) и JSON-метаданные). Поддержка метаданных уровня поля потребовала бы ранней интроспекции [__value__](https://docs.python.org/3/library/typing.html#typing.TypeAliasType.__value__) алиаса, и тогда алиас нельзя было бы хранить как определение в JSON Schema.

> Как и в неявных алиасах, внутри именованного дженерик-алиаса можно использовать [переменные типов](https://docs.python.org/3/library/typing.html#typing.TypeVar):
>
> Примечание

**Python 3.9+**: `ShortList = TypeAliasType('ShortList', Annotated[list[T], Len(max_length=4)], type_params=(T,))`

**Python 3.12+**: `type ShortList[T] = Annotated[list[T], Len(max_length=4)]`

#### Именованные рекурсивные типы (Named recursive types)

Именованные алиасы типов нужны, когда вы определяете рекурсивные алиасы типов (1).

1. По ряду причин Pydantic не поддерживает неявные рекурсивные алиасы — например, не может разрешать [forward annotations](https://docs.pydantic.dev/latest/concepts/forward_annotations/) между модулями.

Пример типа JSON:

**Python 3.9+**:

```python
from typing import Union

from typing_extensions import TypeAliasType

from pydantic import TypeAdapter

Json = TypeAliasType(
    'Json',
    'Union[dict[str, Json], list[Json], str, int, float, bool, None]',  # (1)!
)

ta = TypeAdapter(Json)
print(ta.json_schema())
"""
{
    '$defs': {
        'Json': {
            'anyOf': [
                {
                    'additionalProperties': {'$ref': '#/$defs/Json'},
                    'type': 'object',
                },
                {'items': {'$ref': '#/$defs/Json'}, 'type': 'array'},
                {'type': 'string'},
                {'type': 'integer'},
                {'type': 'number'},
                {'type': 'boolean'},
                {'type': 'null'},
            ]
        }
    },
    '$ref': '#/$defs/Json',
}
"""
```

1. Аннотацию нужно брать в кавычки, так как она вычисляется сразу (а `Json` ещё не определён).

**Python 3.12+**:

```python
from pydantic import TypeAdapter

type Json = dict[str, Json] | list[Json] | str | int | float | bool | None  # (1)!

ta = TypeAdapter(Json)
print(ta.json_schema())
"""
{
    '$defs': {
        'Json': {
            'anyOf': [
                ...
            ]
        }
    },
    '$ref': '#/$defs/Json',
}
"""
```

1. Значение именованного алиаса вычисляется лениво, поэтому forward annotations не нужны.

> В Pydantic есть удобный тип [JsonValue](https://docs.pydantic.dev/latest/api/types/#pydantic.types.JsonValue).
>
> Совет

### Настройка валидации через __get_pydantic_core_schema__ (Customizing validation with __get_pydantic_core_schema__)

Для более глубокой настройки обработки кастомных классов (особенно когда класс доступен или от него можно наследоваться) можно реализовать специальный метод `__get_pydantic_core_schema__`, чтобы задать, как Pydantic строит схему для `pydantic-core`.

Pydantic использует `pydantic-core` внутри для валидации и сериализации; это новый API в Pydantic V2, его могут менять в будущем. По возможности лучше опираться на встроенные конструкции: `annotated-types`, `pydantic.Field`, `BeforeValidator` и т.п.

`__get_pydantic_core_schema__` можно реализовать и на кастомном типе, и на метаданных для `Annotated`. В обоих случаях API похож на цепочку middleware и на «wrap»-валидаторы: есть `source_type` (не обязательно совпадающий с классом, особенно для дженериков) и `handler`, который можно вызвать с типом — либо для перехода к следующему элементу в `Annotated`, либо к внутренней генерации схемы Pydantic.

Минимальная реализация без побочных эффектов: вызвать `handler` с переданным типом и вернуть результат. Можно также изменить тип до вызова handler, изменить core schema после вызова или не вызывать handler вовсе.

#### Как метод кастомного типа (As a method on a custom type)

Пример типа с `__get_pydantic_core_schema__` для настройки валидации (аналог `__get_validators__` в Pydantic V1):

```python
from typing import Any

from pydantic_core import CoreSchema, core_schema

from pydantic import GetCoreSchemaHandler, TypeAdapter


class Username(str):
    @classmethod
    def __get_pydantic_core_schema__(
        cls, source_type: Any, handler: GetCoreSchemaHandler
    ) -> CoreSchema:
        return core_schema.no_info_after_validator_function(cls, handler(str))


ta = TypeAdapter(Username)
res = ta.validate_python('abc')
assert isinstance(res, Username)
assert res == 'abc'
```

Подробнее о настройке JSON Schema для кастомных типов: [JSON Schema](https://docs.pydantic.dev/latest/concepts/json_schema/).

#### Как аннотация (As an annotation)

Часто нужно параметризовать кастомный тип не только дженерик-параметрами (это делается через систему типов). Или не требуется (или не хочется) создавать экземпляр подкласса — нужен исходный тип с дополнительной валидацией.

Например, собственная реализация аналога `pydantic.AfterValidator` (см. Adding validation and serialization):

**Python 3.9+**:

```python
from dataclasses import dataclass
from typing import Annotated, Any, Callable

from pydantic_core import CoreSchema, core_schema

from pydantic import BaseModel, GetCoreSchemaHandler


@dataclass(frozen=True)  # (1)!
class MyAfterValidator:
    func: Callable[[Any], Any]

    def __get_pydantic_core_schema__(
        self, source_type: Any, handler: GetCoreSchemaHandler
    ) -> CoreSchema:
        return core_schema.no_info_after_validator_function(
            self.func, handler(source_type)
        )


Username = Annotated[str, MyAfterValidator(str.lower)]


class Model(BaseModel):
    name: Username


assert Model(name='ABC').name == 'abc'  # (2)!
```

1. Анализаторы типов не будут ругаться на присваивание `'ABC'` типу `Username`, так как не считают `Username` отдельным типом от `str`.
2. `frozen=True` делает `MyAfterValidator` хешируемым; иначе union вроде `Username | None` приведёт к ошибке.

**Python 3.10+** (то же с `collections.abc.Callable`).

#### Работа со сторонними типами (Handling third-party types)

Ещё один случай — интеграция со сторонними типами:

```python
from typing import Annotated, Any

from pydantic_core import core_schema

from pydantic import (
    BaseModel,
    GetCoreSchemaHandler,
    GetJsonSchemaHandler,
    ValidationError,
)
from pydantic.json_schema import JsonSchemaValue


class ThirdPartyType:
    """
    Тип из сторонней библиотеки без интеграции с Pydantic,
    без pydantic_core.CoreSchema и т.п.
    """
    x: int

    def __init__(self):
        self.x = 0


class _ThirdPartyTypePydanticAnnotation:
    @classmethod
    def __get_pydantic_core_schema__(
        cls,
        _source_type: Any,
        _handler: GetCoreSchemaHandler,
    ) -> core_schema.CoreSchema:
        """
        Возвращаем pydantic_core.CoreSchema, где:
        * int парсятся в ThirdPartyType с этим int в атрибуте x
        * экземпляры ThirdPartyType проходят без изменений
        * остальное не проходит валидацию
        * сериализация всегда возвращает int
        """
        def validate_from_int(value: int) -> ThirdPartyType:
            result = ThirdPartyType()
            result.x = value
            return result

        from_int_schema = core_schema.chain_schema([
            core_schema.int_schema(),
            core_schema.no_info_plain_validator_function(validate_from_int),
        ])

        return core_schema.json_or_python_schema(
            json_schema=from_int_schema,
            python_schema=core_schema.union_schema([
                core_schema.is_instance_schema(ThirdPartyType),
                from_int_schema,
            ]),
            serialization=core_schema.plain_serializer_function_ser_schema(
                lambda instance: instance.x
            ),
        )

    @classmethod
    def __get_pydantic_json_schema__(
        cls, _core_schema: core_schema.CoreSchema, handler: GetJsonSchemaHandler
    ) -> JsonSchemaValue:
        return handler(core_schema.int_schema())


PydanticThirdPartyType = Annotated[
    ThirdPartyType, _ThirdPartyTypePydanticAnnotation
]


class Model(BaseModel):
    third_party_type: PydanticThirdPartyType


m_int = Model(third_party_type=1)
assert isinstance(m_int.third_party_type, ThirdPartyType)
assert m_int.third_party_type.x == 1
assert m_int.model_dump() == {'third_party_type': 1}

instance = ThirdPartyType()
instance.x = 10
m_instance = Model(third_party_type=instance)
assert m_instance.model_dump() == {'third_party_type': 10}

try:
    Model(third_party_type='a')
except ValidationError as e:
    print(e)

assert Model.model_json_schema() == {
    'properties': {'third_party_type': {'title': 'Third Party Type', 'type': 'integer'}},
    'required': ['third_party_type'],
    'title': 'Model',
    'type': 'object',
}
```

Так можно задать поведение для типов Pandas, NumPy и т.п.

#### GetPydanticSchema для сокращения кода (Using GetPydanticSchema to reduce boilerplate)

Документация API: [pydantic.types.GetPydanticSchema](https://docs.pydantic.dev/latest/api/types/#pydantic.types.GetPydanticSchema)

Маркер-класс из примеров выше даёт много шаблонного кода. В простых случаях его можно сократить с помощью `pydantic.GetPydanticSchema`:

```python
from typing import Annotated

from pydantic_core import core_schema

from pydantic import BaseModel, GetPydanticSchema


class Model(BaseModel):
    y: Annotated[
        str,
        GetPydanticSchema(
            lambda tp, handler: core_schema.no_info_after_validator_function(
                lambda x: x * 2, handler(tp)
            )
        ),
    ]


assert Model(y='ab').y == 'abab'
```

#### Итог (Summary)

1. Для кастомного типа можно реализовать `__get_pydantic_core_schema__` на самом типе.
2. Под капотом используется `pydantic-core`; к нему можно подключаться через `GetPydanticSchema` или маркер-класс с `__get_pydantic_core_schema__`.
3. У Pydantic есть высокоуровневые хуки для типов через `Annotated` — `AfterValidator`, `Field` и др. Ими стоит пользоваться, когда возможно.

### Работа с кастомными дженерик-классами (Handling custom generic classes)

> Это продвинутый приём; на первых порах он может не понадобиться. Часто достаточно обычных моделей Pydantic.
>
> Предупреждение

[Дженерик-классы](https://docs.python.org/3/library/typing.html#typing.Generic) можно использовать как типы полей и настраивать валидацию по «параметрам типа» (подтипам) через `__get_pydantic_core_schema__`.

Если у дженерик-класса есть класс-метод `__get_pydantic_core_schema__`, для его работы не нужен [arbitrary_types_allowed](https://docs.pydantic.dev/latest/api/config/#pydantic.config.ConfigDict.arbitrary_types_allowed).

Параметр `source_type` не совпадает с `cls`; параметры дженерика можно извлечь через `typing.get_args` (или `typing_extensions.get_args`). Затем по ним строят схему, вызывая `handler.generate_schema`. Важно не делать что-то вроде `handler(get_args(source_type)[0])`, а именно `handler.generate_schema(...)` — чтобы схема для параметра типа не зависела от текущего контекста метаданных `Annotated`.

**Python 3.9+** / **Python 3.10+** (импорт `get_args`, `get_origin` из `typing_extensions` или `typing`):

```python
from dataclasses import dataclass
from typing import Any, Generic, TypeVar

from pydantic_core import CoreSchema, core_schema
from typing_extensions import get_args, get_origin

from pydantic import (
    BaseModel,
    GetCoreSchemaHandler,
    ValidationError,
    ValidatorFunctionWrapHandler,
)

ItemType = TypeVar('ItemType')


@dataclass
class Owner(Generic[ItemType]):
    name: str
    item: ItemType

    @classmethod
    def __get_pydantic_core_schema__(
        cls, source_type: Any, handler: GetCoreSchemaHandler
    ) -> CoreSchema:
        origin = get_origin(source_type)
        if origin is None:
            origin = source_type
            item_tp = Any
        else:
            item_tp = get_args(source_type)[0]
        item_schema = handler.generate_schema(item_tp)

        def val_item(v: Owner[Any], handler: ValidatorFunctionWrapHandler) -> Owner[Any]:
            v.item = handler(v.item)
            return v

        python_schema = core_schema.chain_schema([
            core_schema.is_instance_schema(cls),
            core_schema.no_info_wrap_validator_function(val_item, item_schema),
        ])

        return core_schema.json_or_python_schema(
            json_schema=core_schema.chain_schema([
                core_schema.typed_dict_schema({
                    'name': core_schema.typed_dict_field(core_schema.str_schema()),
                    'item': core_schema.typed_dict_field(item_schema),
                }),
                core_schema.no_info_before_validator_function(
                    lambda data: Owner(name=data['name'], item=data['item']),
                    python_schema,
                ),
            ]),
            python_schema=python_schema,
        )


class Car(BaseModel):
    color: str


class House(BaseModel):
    rooms: int


class Model(BaseModel):
    car_owner: Owner[Car]
    home_owner: Owner[House]


model = Model(
    car_owner=Owner(name='John', item=Car(color='black')),
    home_owner=Owner(name='James', item=House(rooms=3)),
)

# При неверных подтипах — ValidationError
# Аналогично при model_validate_json с перепутанными item
```

#### Дженерик-контейнеры (Generic containers)

Та же идея для дженерик-контейнеров (например, кастомный тип в духе `Sequence`):

```python
from collections.abc import Sequence
from typing import Any, TypeVar

from pydantic_core import ValidationError, core_schema
from typing import get_args  # или typing_extensions.get_args

from pydantic import BaseModel, GetCoreSchemaHandler

T = TypeVar('T')


class MySequence(Sequence[T]):
    def __init__(self, v: Sequence[T]):
        self.v = v

    def __getitem__(self, i):
        return self.v[i]

    def __len__(self):
        return len(self.v)

    @classmethod
    def __get_pydantic_core_schema__(
        cls, source: Any, handler: GetCoreSchemaHandler
    ) -> core_schema.CoreSchema:
        instance_schema = core_schema.is_instance_schema(cls)
        args = get_args(source)
        if args:
            sequence_t_schema = handler.generate_schema(Sequence[args[0]])
        else:
            sequence_t_schema = handler.generate_schema(Sequence)
        non_instance_schema = core_schema.no_info_after_validator_function(
            MySequence, sequence_t_schema
        )
        return core_schema.union_schema([instance_schema, non_instance_schema])


class M(BaseModel):
    model_config = dict(validate_default=True)
    s1: MySequence = [3]

m = M()
print(m.s1.v)  # [3]

class M(BaseModel):
    s1: MySequence[int]

M(s1=[1])
# M(s1=['a']) -> ValidationError
```

### Доступ к имени поля (Access to field name)

> В Pydantic V2 до V2.3 этого не было, [возвращено](https://github.com/pydantic/pydantic/pull/7542) в V2.4.
>
> Примечание

Начиная с Pydantic V2.4, внутри `__get_pydantic_core_schema__` доступно имя поля через `handler.field_name`; оно попадает в `info.field_name`.

```python
from typing import Any

from pydantic_core import core_schema

from pydantic import BaseModel, GetCoreSchemaHandler, ValidationInfo


class CustomType:
    """Кастомный тип, сохраняющий имя поля, в котором использован."""

    def __init__(self, value: int, field_name: str):
        self.value = value
        self.field_name = field_name

    def __repr__(self):
        return f'CustomType<{self.value} {self.field_name!r}>'

    @classmethod
    def validate(cls, value: int, info: ValidationInfo):
        return cls(value, info.field_name)

    @classmethod
    def __get_pydantic_core_schema__(
        cls, source_type: Any, handler: GetCoreSchemaHandler
    ) -> core_schema.CoreSchema:
        return core_schema.with_info_after_validator_function(
            cls.validate, handler(int)
        )


class MyModel(BaseModel):
    my_field: CustomType


m = MyModel(my_field=1)
print(m.my_field)
#> CustomType<1 'my_field'>
```

К `field_name` можно обращаться и из маркеров в `Annotated`, например [AfterValidator](https://docs.pydantic.dev/latest/api/functional_validators/#pydantic.functional_validators.AfterValidator):

```python
from typing import Annotated

from pydantic import AfterValidator, BaseModel, ValidationInfo


def my_validators(value: int, info: ValidationInfo):
    return f'<{value} {info.field_name!r}>'


class MyModel(BaseModel):
    my_field: Annotated[int, AfterValidator(my_validators)]


m = MyModel(my_field=1)
print(m.my_field)
#> <1 'my_field'>
```
