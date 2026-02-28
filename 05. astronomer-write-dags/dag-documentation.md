# Документирование DAG (DAG documentation)

Документация DAG отображается в Airflow UI (вкладка Details, блок с описанием). Два основных способа:

## doc_md

Параметр **doc_md** у DAG принимает строку в формате Markdown или путь к `.md`-файлу. Содержимое показывается в UI вместо стандартного описания.

```python
@dag(doc_md=__doc__)
def my_dag():
    ...

# или
@dag(doc_md="/path/to/readme.md")
def my_dag():
    ...
```

Если передать `__doc__`, используется docstring модуля/функции DAG.

## docstring

Docstring у функции DAG (при использовании `@dag`) или у объекта DAG по умолчанию отображается как описание. Для более богатого форматирования удобнее `doc_md` с Markdown.

Рекомендации: кратко описать назначение DAG, расписание, входы/выходы, контакты владельца. Это помогает новым членам команды и при отладке.

Подробнее: [Create DAG documentation in Apache Airflow](https://www.astronomer.io/docs/learn/custom-airflow-ui-docs-tutorial), [DAG parameters](../02. astronomer-dags/dag-parameters.md).

---

[← Object Storage](airflow-objectstorage.md) | [К содержанию](README.md) | [DAG Factory →](dag-factory.md)
