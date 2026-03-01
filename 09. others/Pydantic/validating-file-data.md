# Валидация данных из файлов (Validating File Data)

С помощью `pydantic` удобно проверять данные из разных источников. Ниже — как валидировать данные из файлов разных форматов.

> Если вы разбираете конфигурацию или настройки из таких форматов, имеет смысл обратить внимание на библиотеку [pydantic-settings](https://docs.pydantic.dev/latest/api/pydantic_settings/#pydantic_settings), в которой есть встроенная поддержка разбора таких данных.
>
> Примечание

## Данные JSON

Файлы `.json` часто используют для хранения пар ключ/значение в удобном для человека виде. Пример содержимого `.json`:

```json
{
    "name": "John Doe",
    "age": 30,
    "email": "[email protected]"
}
```

Валидация таких данных моделью Pydantic:

```python
import pathlib

from pydantic import BaseModel, EmailStr, PositiveInt


class Person(BaseModel):
    name: str
    age: PositiveInt
    email: EmailStr


json_string = pathlib.Path('person.json').read_text()
person = Person.model_validate_json(json_string)
print(person)
#> name='John Doe' age=30 email='[email protected]'
```

Если данные в файле невалидны, `pydantic` выбрасывает [ValidationError](https://docs.pydantic.dev/latest/api/pydantic_core/#pydantic_core.ValidationError). Например, в файле:

```json
{
    "age": -30,
    "email": "not-an-email-address"
}
```

ошибок три:

1. Поле `email` — не корректный email.
2. Поле `age` — отрицательное число.
3. Отсутствует обязательное поле `name`.

При валидации Pydantic поднимает [ValidationError](https://docs.pydantic.dev/latest/api/pydantic_core/#pydantic_core.ValidationError) со всеми этими замечаниями:

```python
import pathlib

from pydantic import BaseModel, EmailStr, PositiveInt, ValidationError


class Person(BaseModel):
    name: str
    age: PositiveInt
    email: EmailStr


json_string = pathlib.Path('person.json').read_text()
try:
    person = Person.model_validate_json(json_string)
except ValidationError as err:
    print(err)
    """
    3 validation errors for Person
    name
    Field required [type=missing, input_value={'age': -30, 'email': 'not-an-email-address'}, input_type=dict]
        For further information visit https://errors.pydantic.dev/2.10/v/missing
    age
    Input should be greater than 0 [type=greater_than, input_value=-30, input_type=int]
        For further information visit https://errors.pydantic.dev/2.10/v/greater_than
    email
    value is not a valid email address: An email address must have an @-sign. [type=value_error, input_value='not-an-email-address', input_type=str]
    """
```

Нередко в `.json` лежит массив однотипных записей, например список людей:

```json
[
    {
        "name": "John Doe",
        "age": 30,
        "email": "[email protected]"
    },
    {
        "name": "Jane Doe",
        "age": 25,
        "email": "[email protected]"
    }
]
```

В таком случае данные можно валидировать типом `list[Person]`:

```python
import pathlib

from pydantic import BaseModel, EmailStr, PositiveInt, TypeAdapter


class Person(BaseModel):
    name: str
    age: PositiveInt
    email: EmailStr


person_list_adapter = TypeAdapter(list[Person])  # (1)!

json_string = pathlib.Path('people.json').read_text()
people = person_list_adapter.validate_json(json_string)
print(people)
#> [Person(name='John Doe', age=30, email='[email protected]'), Person(name='Jane Doe', age=25, email='[email protected]')]
```

1. [TypeAdapter](https://docs.pydantic.dev/latest/api/type_adapter/#pydantic.type_adapter.TypeAdapter) используется для валидации списка объектов `Person`. TypeAdapter — конструкция Pydantic для валидации данных по одному типу.

## Файлы JSON Lines

По аналогии со списком объектов из `.json` можно валидировать данные из `.jsonl`. В `.jsonl` каждая строка — отдельный JSON-объект.

Пример `.jsonl`:

```
{"name": "John Doe", "age": 30, "email": "[email protected]"}
{"name": "Jane Doe", "age": 25, "email": "[email protected]"}
```

Валидация тем же подходом, что и для `.json`:

```python
import pathlib

from pydantic import BaseModel, EmailStr, PositiveInt


class Person(BaseModel):
    name: str
    age: PositiveInt
    email: EmailStr


json_lines = pathlib.Path('people.jsonl').read_text().splitlines()
people = [Person.model_validate_json(line) for line in json_lines]
print(people)
#> [Person(name='John Doe', age=30, email='[email protected]'), Person(name='Jane Doe', age=25, email='[email protected]')]
```

## Файлы CSV

CSV — один из самых распространённых форматов табличных данных. Чтобы валидировать данные из CSV, можно загрузить их стандартным модулем `csv` и проверить моделью Pydantic.

Пример CSV:

```
name,age,email
John Doe,30,[email protected]
Jane Doe,25,[email protected]
```

Валидация:

```python
import csv

from pydantic import BaseModel, EmailStr, PositiveInt


class Person(BaseModel):
    name: str
    age: PositiveInt
    email: EmailStr


with open('people.csv') as f:
    reader = csv.DictReader(f)
    people = [Person.model_validate(row) for row in reader]

print(people)
#> [Person(name='John Doe', age=30, email='[email protected]'), Person(name='Jane Doe', age=25, email='[email protected]')]
```

## Файлы TOML

TOML часто используют для конфигурации из‑за простоты и читаемости.

Пример TOML:

```toml
name = "John Doe"
age = 30
email = "[email protected]"
```

Валидация:

```python
import tomllib

from pydantic import BaseModel, EmailStr, PositiveInt


class Person(BaseModel):
    name: str
    age: PositiveInt
    email: EmailStr


with open('person.toml', 'rb') as f:
    data = tomllib.load(f)

person = Person.model_validate(data)
print(person)
#> name='John Doe' age=30 email='[email protected]'
```

## Файлы YAML

YAML (YAML Ain't Markup Language) — удобный для человека формат сериализации данных, часто используется для конфигов.

Пример YAML:

```yaml
name: John Doe
age: 30
email: [email protected]
```

Валидация:

```python
import yaml

from pydantic import BaseModel, EmailStr, PositiveInt


class Person(BaseModel):
    name: str
    age: PositiveInt
    email: EmailStr


with open('person.yaml') as f:
    data = yaml.safe_load(f)

person = Person.model_validate(data)
print(person)
#> name='John Doe' age=30 email='[email protected]'
```

## Файлы XML

XML (eXtensible Markup Language) — язык разметки с правилами кодирования документов в формате, удобном и для людей, и для программ.

Пример XML:

```xml
<?xml version="1.0"?>
<person>
    <name>John Doe</name>
    <age>30</age>
    <email>[email protected]</email>
</person>
```

Валидация:

```python
import xml.etree.ElementTree as ET

from pydantic import BaseModel, EmailStr, PositiveInt


class Person(BaseModel):
    name: str
    age: PositiveInt
    email: EmailStr


tree = ET.parse('person.xml').getroot()
data = {child.tag: child.text for child in tree}
person = Person.model_validate(data)
print(person)
#> name='John Doe' age=30 email='[email protected]'
```

## Файлы INI

INI — простой формат конфигурации с секциями и парами ключ–значение. Часто встречается в приложениях под Windows и в старом ПО.

Пример INI:

```ini
[PERSON]
name = John Doe
age = 30
email = [email protected]
```

Валидация:

```python
import configparser

from pydantic import BaseModel, EmailStr, PositiveInt


class Person(BaseModel):
    name: str
    age: PositiveInt
    email: EmailStr


config = configparser.ConfigParser()
config.read('person.ini')
person = Person.model_validate(config['PERSON'])
print(person)
#> name='John Doe' age=30 email='[email protected]'
```
