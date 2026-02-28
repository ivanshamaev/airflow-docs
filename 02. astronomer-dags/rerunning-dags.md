# Повторный запуск DAG и задач (Rerunning, Backfill)

## Ручной запуск (Trigger)

- В UI: открыть DAG → кнопка **Trigger DAG** (или **Trigger DAG w/ config**). Создаётся один DAG run (по умолчанию с текущей логической датой или без неё, в зависимости от настроек).
- Через CLI: `airflow dags trigger <dag_id>`.
- Через REST API: endpoint запуска DAG с опциональными params и config.

## Backfill

**Backfill** — создание нескольких DAG run за прошлые периоды (по расписанию за выбранный диапазон дат). Используется для «догоняющей» обработки после добавления DAG или после сбоев.

- В UI: DAG → **Trigger** → в диалоге выбрать **Backfill**, указать диапазон дат и опции (reprocessing behavior, направление, max active runs и т.д.).
- CLI: `airflow dags backfill -s <start_date> -e <end_date> <dag_id>` (и другие флаги).
- При backfill создаётся отдельный DAG run на каждый интервал (например, на каждый день); они могут выполняться параллельно в пределах **max_active_runs**.

## Очистка и перезапуск задач

- **Clear** — сброс состояния выбранных экземпляров задач (в UI: выбрать задачу/задачи в Grid → Clear). После сброса задачи снова планируются и выполняются. Можно очистить один task instance, все задачи одного run или несколько run.
- **Mark failed / success** — ручная установка состояния (для разблокирования downstream при необходимости).
- **Retry** — повторная попытка для упавшей задачи (если остались retries). Иногда проще сделать Clear выбранных задач.

Перезапуск только части задач (например, с определённой даты) делается комбинацией backfill (при необходимости) и clear выбранных экземпляров.

Подробнее: [Rerunning DAGs](https://www.astronomer.io/docs/learn/rerunning-dags), [Backfill](https://www.astronomer.io/docs/learn/rerunning-dags#backfill).

---

[← К содержанию](README.md) | [DAG →](../01. astronomer-basic/dags.md) | [Отладка →](debugging-dags.md)
