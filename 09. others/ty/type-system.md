# Система типов

В целом можно ожидать, что ty поддерживает все возможности типизации, описанные и заданные в [документации по типизации Python](https://typing.python.org/en/latest/spec/index.html) (подробный обзор см. в [issue по отслеживанию возможностей системы типов](https://github.com/astral-sh/ty/issues/1889)). На этой странице выделены некоторые особенности системы типов ty.

## Повторные объявления

ty позволяет повторно использовать один и тот же символ с другим типом. В следующем примере параметр `paths` объявляется заново как список строк:

```python
def split_paths(paths: str) -> list[Path]:
    paths: list[str] = paths.split(":")
    return [Path(p) for p in paths]
```

(Полный пример в [playground](https://play.ty.dev/80a74c95-a43e-4a3d-8c26-f88e879d7dcb))

## Пересечение типов (intersection types)

ty имеет встроенную поддержку типов пересечения. В отличие от объединения `A | B` («либо A, либо B»), тип пересечения `A & B` означает «и A, и B». Сужение типов в ty основано на пересечениях. Обратите внимание, как в следующей функции можно вызвать `obj.serialize_json()` и обратиться к свойству `.version`:

```python
def output_as_json(obj: Serializable) -> str:
    if isinstance(obj, Versioned):
        reveal_type(obj)  # выводит: Serializable & Versioned

        return str({
            "data": obj.serialize_json(),
            "version": obj.version
        })
    else:
        return obj.serialize_json()
```

(Полный пример в [playground](https://play.ty.dev/39241435-5e78-4ce9-817f-ce65be73a6ed))

Пересечения также можно строить с постепенными типами вроде `Any` или неявного `Unknown`. Например, вы вызываете нетипизированный (сторонний) код, который возвращает объект типа `Unknown`. Сужение типа этого объекта через `isinstance` даёт тип пересечения `Unknown & Iterable`. Такой тип позволяет использовать `obj` как итерируемый объект, но важнее то, что по-прежнему доступны атрибуты исходного неизвестного типа (в примере — `.description`):

```python
def print_content(data: bytes):
    obj = untyped_library.deserialize(data)

    if isinstance(obj, Iterable):
        print(obj.description)
        for part in obj:
            print("*", part.description)
    else:
        print(obj.description)
```

(Полный пример в [playground](https://play.ty.dev/8f98820e-7306-4d69-b572-56d69a92b90f))

Типы пересечения также используются при сужении через `hasattr`. В примере ниже тип `Person | Animal | None` сужается с помощью `hasattr(…, "name")`. `Person` остаётся в суженном объединении, так как у него есть атрибут `name`. `Animal` пересекается с синтетическим протоколом — учитывается возможность подклассов `Animal` с членом `name`. `None` полностью исключается, так как это финальный тип без атрибута `name`:

```python
class Person:
    name: str

class Animal:
    species: str

def greet(being: Person | Animal | None):
    if hasattr(being, "name"):
        # `being` теперь имеет тип `Person | (Animal & <Protocol with members 'name'>)`

        print(f"Hello, {being.name}!")
    else:
        print("Hello there!")
```

(Полный пример в [playground](https://play.ty.dev/31f2c718-516a-4a85-80e0-2a4682b818f1))

Info

Если в такой ситуации нужно исключить из суженного типа и `Animal`, можно сделать класс `Animal` помеченным как `@final`. Тогда ty сможет вывести более точный тип для `being.name` (`str` вместо `object`).

Если ty — единственный используемый вами проверяющий типов, можно напрямую использовать типы пересечения в аннотациях, импортируя `Intersection` из специального модуля `ty_extensions`, который (пока) доступен только во время проверки типов:

```python
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from ty_extensions import Intersection

    type SerializableVersioned = Intersection[Serializable, Versioned]

def output_as_json(obj: SerializableVersioned) -> str:
    ...
```

(Полный пример в [playground](https://play.ty.dev/f003e901-0e45-4f45-9759-d6db9d5e5f66))

## Верхняя и нижняя материализации

У постепенных типов обычно есть две специальные [материализации](https://typing.python.org/en/latest/spec/concepts.html#materialization). Верхняя материализация — «наибольший» тип, к которому может свестись постепенный тип: объединение всех возможных материализаций. Например, верхняя материализация `Any` — это `object`, а верхняя материализация `Any & int` — `int`. Для инвариантных универсальных классов верхняя материализация не выражается в системе типов Python, но ty пересекает с ней при проверках `isinstance` с универсальными классами. Например, при проверке `isinstance(…, list)` ty пересекает с верхней материализацией `list[Unknown]`:

```python
@final
class Item: ...

def process(items: Item | list[Item]):
    if isinstance(items, list):
        # выводит: list[Item]
        reveal_type(items)
```

(Полный пример в [playground](https://play.ty.dev/f1306120-0b8d-4ed5-b832-1f2d379eae2b))

Info

Может возникнуть вопрос, зачем здесь `Item` объявлен как `@final`. Без декоратора `@final` выведенный тип в ветке `if` станет `(Item & Top[list[Unknown]]) | list[Item]`. Так учитывается возможность классов, наследующих и от `Item`, и от `list`! Если нужно исключить такой случай, можно проверять `isinstance` по `Item`. Тогда в ветке `else` суженный тип будет `list[Item] & ~Item`, что по сути ведёт себя как `list[Item]`.

## Достижимость на основе типов

Анализ достижимости в ty опирается на вывод типов. Это позволяет ty находить недостижимые ветки в большем числе ситуаций по сравнению с подходами, основанными на нескольких известных паттернах (например, проверках `sys.version_info >= (3, 10)`). У этого есть практическое применение. Рассмотрим код, который должен работать с двумя мажорными версиями зависимости. Следующий код успешно проверяется типом и при установленном pydantic 1.x, и при pydantic 2.x. В обоих случаях ty считает достижимой только соответствующую ветку и не выдаёт ошибок типов для другой. Это возможно, потому что `pydantic.__version__.startswith("2.")` может быть вычислено в `True` или `False` во время проверки типов:

```python
import pydantic
from pydantic import BaseModel

PYDANTIC_V2 = pydantic.__version__.startswith("2.")

class Person(BaseModel):
    name: str

def to_json(person: Person):
    if PYDANTIC_V2:
        return person.model_dump_json()  # ошибки не будет при проверке с 1.x
    else:
        return person.json()
```

(Полный пример в [playground](https://play.ty.dev/34a227bb-93d5-405e-86c3-72f57ec5642e))

## Постепенная гарантия

ty в целом стремится не выдавать ложных ошибок типов в нетипизированном коде. Следующий фрагмент не приводит к ошибкам типов при проверке ty (тогда как другие проверяющие типов могут считать, что `max_retries` имеет тип `None`, и сообщать об ошибке при присваивании атрибута):

```python
class RetryPolicy:
    max_retries = None

policy = RetryPolicy()
policy.max_retries = 1
```

(Полный пример в [playground](https://play.ty.dev/a5286db1-cdfd-45e7-af54-29649ba5c423))

Это достигается тем, что `max_retries` рассматривается как тип `Unknown | None`: тип атрибута не известен полностью, но `None` — одно из возможных значений.

Пользователи могут включить более строгую проверку, добавив аннотации типов (в данном случае — `int | None`).

Info

Также планируется режим для тех, кто предпочитает по умолчанию более строгий вывод типов в таких ситуациях. Обновления можно отслеживать в [этом issue](https://github.com/astral-sh/ty/issues/1240).

## Итерация неподвижной точки

Когда тип символа циклически зависит от самого себя, ty использует механизм итерации неподвижной точки, чтобы вывести тип этого символа. В методе `tick` ниже тип `self.value` зависит от самого `self.value`. ty сначала предполагает, что `self.value` имеет тип `Unknown | Literal[0]` (тип, выведенный из метода `__init__`), затем итерирует, пока тип не сойдётся к `Unknown | Literal[0, 1, 2, 3, 4]`. Без операции взятия по модулю объединение росло бы бесконечно. В таком случае после определённого числа итераций используется откат к типу `int`.

```python
class LoopingCounter:
    def __init__(self):
        self.value = 0

    def tick(self):
        self.value = (self.value + 1) % 5

# выводит: Unknown | Literal[0, 1, 2, 3, 4]
reveal_type(LoopingCounter().value)
```

(Полный пример в [playground](https://play.ty.dev/64400d96-ee1b-48f3-8361-b583dddddf82))
