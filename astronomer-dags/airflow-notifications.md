# Уведомления об ошибках в Airflow (Error notifications)

Настройка оповещений при сбоях задач и DAG позволяет быстрее реагировать на проблемы в пайплайнах.

## Способы уведомления

- **Email:** в `default_args` DAG или на уровне задачи задаются `email_on_failure`, `email_on_retry`, `email` (список адресов). Требуется настроенная [почта](https://airflow.apache.org/docs/apache-airflow/stable/configurations-ref.html#smtp) в конфигурации Airflow.
- **Callbacks:** параметры `on_failure_callback`, `on_retry_callback`, `on_success_callback` у задачи или в `default_args` — вызываемая функция (callable), которой передаётся контекст; в ней можно отправить сообщение в Slack, PagerDuty, записать в лог и т.д.
- **Провайдеры:** операторы из провайдеров Slack, PagerDuty, Discord и др. — задача может отправлять сообщение при падении (например, через отдельную downstream-задачу с `trigger_rule="one_failed"` или внутри callback).

## Рекомендации

- Не дублируйте одни и те же оповещения на DAG и на каждой задаче — выберите уровень (DAG или критичные задачи).
- В callback используйте контекст (`context["task_instance"]`, `context["dag_run"]`) для формирования текста (имя DAG, задачи, логическая дата, ссылка на лог).
- Для продакшена предпочтительны каналы с доставкой (Slack, PagerDuty) и единый список получателей (переменные, конфиг), а не хардкод email в DAG.

Подробнее: [Error Notifications in Airflow](https://www.astronomer.io/docs/learn/error-notifications-in-airflow), [Configuration: Email](https://airflow.apache.org/docs/apache-airflow/stable/configurations-ref.html#smtp).

---

[← К содержанию](README.md) | [Параметры →](airflow-params.md)
