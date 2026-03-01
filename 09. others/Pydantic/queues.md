# Очереди (Queues)

Pydantic удобен для валидации данных, которые попадают в очереди и извлекаются из них. Ниже — как валидировать и сериализовать данные при работе с разными системами очередей.

## Очередь Redis

Redis — популярное хранилище структур данных в памяти.

Чтобы запустить пример локально, нужно [установить Redis](https://redis.io/docs/latest/operate/oss_and_stack/install/install-redis/) и запустить сервер.

Простой пример использования Pydantic:

1. Десериализация и валидация данных при извлечении из очереди
2. Сериализация данных перед помещением в очередь

```python
import redis

from pydantic import BaseModel, EmailStr


class User(BaseModel):
    id: int
    name: str
    email: EmailStr


r = redis.Redis(host='localhost', port=6379, db=0)
QUEUE_NAME = 'user_queue'


def push_to_queue(user_data: User) -> None:
    serialized_data = user_data.model_dump_json()
    r.rpush(QUEUE_NAME, serialized_data)
    print(f'Added to queue: {serialized_data}')


user1 = User(id=1, name='John Doe', email='[email protected]')
user2 = User(id=2, name='Jane Doe', email='[email protected]')

push_to_queue(user1)
#> Added to queue: {"id":1,"name":"John Doe","email":"[email protected]"}

push_to_queue(user2)
#> Added to queue: {"id":2,"name":"Jane Doe","email":"[email protected]"}


def pop_from_queue() -> None:
    data = r.lpop(QUEUE_NAME)

    if data:
        user = User.model_validate_json(data)
        print(f'Validated user: {repr(user)}')
    else:
        print('Queue is empty')


pop_from_queue()
#> Validated user: User(id=1, name='John Doe', email='[email protected]')

pop_from_queue()
#> Validated user: User(id=2, name='Jane Doe', email='[email protected]')

pop_from_queue()
#> Queue is empty
```

## RabbitMQ

RabbitMQ — популярный брокер сообщений, реализующий протокол AMQP.

Чтобы запустить пример локально, нужно [установить RabbitMQ](https://www.rabbitmq.com/download.html) и запустить сервер.

Простой пример использования Pydantic:

1. Десериализация и валидация данных при извлечении из очереди
2. Сериализация данных перед помещением в очередь

Сначала скрипт отправителя:

```python
import pika

from pydantic import BaseModel, EmailStr


class User(BaseModel):
    id: int
    name: str
    email: EmailStr


connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
channel = connection.channel()
QUEUE_NAME = 'user_queue'
channel.queue_declare(queue=QUEUE_NAME)


def push_to_queue(user_data: User) -> None:
    serialized_data = user_data.model_dump_json()
    channel.basic_publish(
        exchange='',
        routing_key=QUEUE_NAME,
        body=serialized_data,
    )
    print(f'Added to queue: {serialized_data}')


user1 = User(id=1, name='John Doe', email='[email protected]')
user2 = User(id=2, name='Jane Doe', email='[email protected]')

push_to_queue(user1)
#> Added to queue: {"id":1,"name":"John Doe","email":"[email protected]"}

push_to_queue(user2)
#> Added to queue: {"id":2,"name":"Jane Doe","email":"[email protected]"}

connection.close()
```

Скрипт получателя:

```python
import pika

from pydantic import BaseModel, EmailStr


class User(BaseModel):
    id: int
    name: str
    email: EmailStr


def main():
    connection = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
    channel = connection.channel()
    QUEUE_NAME = 'user_queue'
    channel.queue_declare(queue=QUEUE_NAME)

    def process_message(
        ch: pika.channel.Channel,
        method: pika.spec.Basic.Deliver,
        properties: pika.spec.BasicProperties,
        body: bytes,
    ):
        user = User.model_validate_json(body)
        print(f'Validated user: {repr(user)}')
        ch.basic_ack(delivery_tag=method.delivery_tag)

    channel.basic_consume(queue=QUEUE_NAME, on_message_callback=process_message)
    channel.start_consuming()


if __name__ == '__main__':
    try:
        main()
    except KeyboardInterrupt:
        pass
```

Как проверить пример:

1. В одном терминале запустите скрипт отправителя, чтобы отправить сообщения.
2. В другом терминале запустите скрипт получателя, чтобы запустить потребителя.

## ARQ

ARQ — быстрая очередь задач на Redis для Python. Построена поверх Redis и даёт простой способ обрабатывать фоновые задачи.

Чтобы запустить пример локально, нужно [установить Redis](https://redis.io/docs/latest/operate/oss_and_stack/install/install-redis/) и запустить сервер.

Простой пример использования Pydantic с ARQ:

1. Валидация и десериализация данных при обработке задач
2. Сериализация данных при постановке задач в очередь
3. Модель для данных задачи

```python
import asyncio
from typing import Any

from arq import create_pool
from arq.connections import RedisSettings

from pydantic import BaseModel, EmailStr


class User(BaseModel):
    id: int
    name: str
    email: EmailStr


REDIS_SETTINGS = RedisSettings()


async def process_user(ctx: dict[str, Any], user_data: dict[str, Any]) -> None:
    user = User.model_validate(user_data)
    print(f'Processing user: {repr(user)}')


async def enqueue_jobs(redis):
    user1 = User(id=1, name='John Doe', email='[email protected]')
    user2 = User(id=2, name='Jane Doe', email='[email protected]')

    await redis.enqueue_job('process_user', user1.model_dump())
    print(f'Enqueued user: {repr(user1)}')

    await redis.enqueue_job('process_user', user2.model_dump())
    print(f'Enqueued user: {repr(user2)}')


class WorkerSettings:
    functions = [process_user]
    redis_settings = REDIS_SETTINGS


async def main():
    redis = await create_pool(REDIS_SETTINGS)
    await enqueue_jobs(redis)


if __name__ == '__main__':
    asyncio.run(main())
```

Скрипт самодостаточен: его можно запускать и для постановки задач в очередь, и для их обработки.
