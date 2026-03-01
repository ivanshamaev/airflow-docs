# Базы данных (Databases)

Pydantic хорошо подходит для описания моделей при работе с ORM (object relational mapping). ORM связывают объекты с таблицами в базе данных и наоборот.

## SQLAlchemy

Pydantic можно сочетать с SQLAlchemy: с его помощью задают схему моделей базы данных.

**Дублирование кода**

При использовании Pydantic с SQLAlchemy может возникать дублирование кода. В таком случае можно рассмотреть [SQLModel](https://sqlmodel.tiangolo.com/), который объединяет Pydantic и SQLAlchemy и во многом убирает это дублирование.

Если нужен именно «чистый» Pydantic с SQLAlchemy, модели Pydantic лучше держать рядом с моделями SQLAlchemy, как в примере ниже. Здесь используется алиас в Pydantic, чтобы назвать колонку так же, как зарезервированное поле SQLAlchemy, и избежать конфликта.

```python
import sqlalchemy as sa
from sqlalchemy.orm import declarative_base

from pydantic import BaseModel, ConfigDict, Field


class MyModel(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    metadata: dict[str, str] = Field(alias='metadata_')


Base = declarative_base()


class MyTableModel(Base):
    __tablename__ = 'my_table'
    id = sa.Column('id', sa.Integer, primary_key=True)
    # 'metadata' зарезервировано в SQLAlchemy, поэтому '_'
    metadata_ = sa.Column('metadata', sa.JSON)


sql_model = MyTableModel(metadata_={'key': 'val'}, id=1)
pydantic_model = MyModel.model_validate(sql_model)

print(pydantic_model.model_dump())
#> {'metadata': {'key': 'val'}}
print(pydantic_model.model_dump(by_alias=True))
#> {'metadata_': {'key': 'val'}}
```

> В примере выше всё работает, потому что при заполнении полей алиасы имеют приоритет над именами полей. Обращение к атрибуту `metadata` у модели SQLAlchemy привело бы к `ValidationError`.
>
> Примечание
