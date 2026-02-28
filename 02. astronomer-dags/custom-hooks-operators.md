# Кастомные хуки и операторы (Custom hooks and operators)

Когда встроенных операторов или хуков недостаточно, можно написать свои и подключать их в DAG или через плагины.

## Кастомный хук

- Наследуйтесь от [BaseHook](https://github.com/apache/airflow/blob/main/airflow-core/src/airflow/hooks/base.py). В `__init__` принимайте `*args, **kwargs` и передавайте в `super().__init__()`.
- Подключение к внешней системе обычно настраивается по **connection_id** (получение из метаданных через базовый класс).
- Реализуйте методы для нужных операций (get, push, list и т.д.). Хук можно использовать внутри задачи (в `@task` или в callable PythonOperator) или внутри кастомного оператора.

Хуки размещают в папке `plugins/` (Airflow подхватывает их при наличии подклассов Hook) или импортируют напрямую в DAG-файле.

## Кастомный оператор

- Наследуйтесь от [BaseOperator](https://airflow.apache.org/docs/apache-airflow/stable/_api/airflow/models/baseoperator.html) (или от оператора провайдера). В `__init__` сохраняйте нужные параметры и вызывайте `super().__init__(*args, **kwargs)`.
- Реализуйте метод **execute(self, context)**. В него передаётся контекст задачи; здесь вызывается логика (в т.ч. использование хука) и при необходимости возвращается значение (оно попадёт в XCom).
- При необходимости задайте **template_fields**, **template_ext**, **ui_color**, **ui_fg_color** и т.д.

Оператор можно положить в `plugins/` (как часть плагина) или импортировать из своего пакета (например, из `include/` или установленного пакета).

## Плагины Airflow

В каталоге **plugins/** можно разместить Python-модули с классами, наследующими `AirflowPlugin`. В плагине регистрируют кастомные операторы, хуки, макросы, пункты меню UI и т.д. После добавления плагина классы становятся доступны в DAG без явного импорта из файла (операторы/хуки появляются в списках, если плагин их регистрирует).

Подробнее: [Custom hooks and operators](https://www.astronomer.io/docs/learn/airflow-importing-custom-hooks-operators), [Airflow Plugins](https://airflow.apache.org/docs/apache-airflow/stable/plugins.html).

---

[← К содержанию](README.md) | [Хуки →](../01. astronomer-basic/hooks.md) | [Операторы →](../01. astronomer-basic/operators.md)
