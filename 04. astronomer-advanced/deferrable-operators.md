# Deferrable-операторы (Deferrable operators)

**Deferrable-оператор** откладывает выполнение задачи. Классический сенсор занимает слот воркера на всё время ожидания; deferrable-оператор освобождает воркер до срабатывания триггера.

![Сенсор занимает слот воркера](images/3-0_standard_sensor_slot_taking.png)  
![Процесс deferrable-оператора](images/deferrable_operator_process.png)  
![Deferrable задача в Grid](images/3-0_deferrable_grid_view.png)

Вместо того чтобы занимать воркер в ожидании (например, сенсор), задача передаёт управление **триггеру** (асинхронная функция в компоненте Triggerer). Когда условие выполняется, триггер отправляет событие, задача снова ставится в очередь и выполняется. Так экономятся ресурсы воркера и повышается масштабируемость.

Триггер наследуется от **BaseTrigger**, реализует метод `run()` (async generator, yield `TriggerEvent` при готовности). Оператор наследует **BaseOperator** и использует метод `defer()` с указанием триггера и даты/метаданных. Вместо классического сенсора с `poke_interval` используется deferrable-версия (например, `TimeSensorAsync`, провайдерские сенсоры с суффиксом Async).

Плюсы: меньше занятых слотов воркера, быстрее реакция при большом числе сенсоров. Требуется запущенный **Triggerer**. Многие сенсоры из провайдеров имеют deferrable-варианты; можно реализовать свой триггер и оператор.

Подробнее: [Deferrable operators](https://www.astronomer.io/docs/learn/deferrable-operators), [Triggerer](https://www.astronomer.io/docs/learn/airflow-components).

---

[← Custom XCom](custom-xcom-backends.md) | [К содержанию](README.md) | [Event-driven →](event-driven-scheduling.md)
