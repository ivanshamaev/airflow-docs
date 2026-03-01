# Веб- и API-запросы (Web and API Requests)

Модели Pydantic удобны для валидации и сериализации данных запросов и ответов. Pydantic широко используется во многих веб-фреймворках и библиотеках: FastAPI, Django, Flask, HTTPX и др.

## Запросы через httpx

[httpx](https://www.python-httpx.org/) — HTTP-клиент для Python 3 с синхронным и асинхронным API. В примере ниже запрашиваются данные пользователя из [JSONPlaceholder API](https://jsonplaceholder.typicode.com/) и проверяются моделью Pydantic.

```python
import httpx

from pydantic import BaseModel, EmailStr


class User(BaseModel):
    id: int
    name: str
    email: EmailStr


url = 'https://jsonplaceholder.typicode.com/users/1'

response = httpx.get(url)
response.raise_for_status()

user = User.model_validate(response.json())
print(repr(user))
#> User(id=1, name='Leanne Graham', email='[email protected]')
```

При работе с HTTP-запросами часто полезен [TypeAdapter](https://docs.pydantic.dev/latest/api/type_adapter/#pydantic.type_adapter.TypeAdapter) из Pydantic. Пример валидации списка пользователей:

```python
from pprint import pprint

import httpx

from pydantic import BaseModel, EmailStr, TypeAdapter


class User(BaseModel):
    id: int
    name: str
    email: EmailStr


url = 'https://jsonplaceholder.typicode.com/users/'  # (1)!

response = httpx.get(url)
response.raise_for_status()

users_list_adapter = TypeAdapter(list[User])

users = users_list_adapter.validate_python(response.json())
pprint([u.name for u in users])
"""
['Leanne Graham',
 'Ervin Howell',
 'Clementine Bauch',
 'Patricia Lebsack',
 'Chelsey Dietrich',
 'Mrs. Dennis Schulist',
 'Kurtis Weissnat',
 'Nicholas Runolfsdottir V',
 'Glenna Reichert',
 'Clementina DuBuque']
"""
```

1. Здесь запрашивается эндпоинт `/users/`, чтобы получить список пользователей.
