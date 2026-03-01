# Отладка DAG в Airflow

В этом руководстве описано, как находить и устранять типичные проблемы с DAG в Airflow, а также куда обращаться, если решения не нашлось. Основной фокус — локальная разработка, но большая часть применима и к продакшену.

> Рекомендуется наладить систематическое тестирование DAG, чтобы предотвращать типичные ошибки. См. [Test Airflow DAGs](../04.%20astronomer-advanced/testing-airflow.md).

## Необходимая база

Полезно понимать:

- Основы DAG в Airflow. См. [Введение в DAG](../01.%20astronomer-basic/dags.md).
- Основы Airflow. См. [Get started with Airflow tutorial](../01.%20astronomer-basic/README.md).

## Общий подход к отладке

Чтобы повысить шансы найти и исправить ошибку, задайте себе следующие вопросы:

- Воспроизводится ли проблема в новом локальном инстансе Airflow (например, через [Astro CLI](https://www.astronomer.io/docs/astro/cli/overview))?
- Какие версии Airflow и провайдеров используются? Сверьтесь с подходящей версией [документации Airflow](https://airflow.apache.org/docs/apache-airflow/stable/index.html).
- Можете ли собрать нужные логи? Где они лежат и как настраиваются: [Airflow logging](../04.%20astronomer-advanced/logging.md).
- Проблема со всеми DAG или только с одним?
- Корректно ли настроены [подключения Airflow](../01.%20astronomer-basic/connections.md) (учётные данные)? См. раздел «Отладка подключений» ниже.
- Есть ли у Airflow доступ ко всем нужным файлам? Особенно важно в контейнерах (например, [Astro CLI](https://www.astronomer.io/docs/astro/cli/overview)).
- В каком состоянии [компоненты Airflow](../03.%20astronomer-infra/airflow-components.md)? Проверьте логи каждого компонента и при необходимости перезапустите окружение.
- Проблема в Airflow или во внешней системе? Проверьте, выполняется ли действие во внешней системе без Airflow.

Ответы помогут сузить круг причин и выбрать следующие шаги.

## Airflow не запускается на Astro CLI

Чаще всего Airflow локально запускают через [Astro CLI](https://www.astronomer.io/docs/astro/cli/install-cli), [standalone](https://airflow.apache.org/docs/apache-airflow/stable/start.html) или [Docker](https://airflow.apache.org/docs/apache-airflow/stable/howto/docker-compose/index.html). Ниже — типичные проблемы именно с Astro CLI.

Частые причины:

- Компоненты Airflow уходят в crash-loop из-за ошибок в кастомных плагинах или XCom backend. Логи планировщика: `astro dev logs -s`.
- Ошибки из-за кастомных команд в Dockerfile или конфликтов зависимостей в `packages.txt` и `requirements.txt`.
- Astro CLI установлен некорректно. Проверьте: `astro version`. При наличии новой версии обновите CLI.

По проблемам инфраструктуры на других платформах (Docker, Kubernetes с [Helm Chart](https://airflow.apache.org/docs/helm-chart/stable/index.html), managed-сервисы) см. соответствующую документацию и поддержку.

Подробнее об [тестировании и отладке локально](https://www.astronomer.io/docs/astro/cli/test-your-astro-project-locally) с Astro CLI — в документации Astro.

## Типичные проблемы с DAG

### DAG не отображаются в UI

Если DAG не появляется в UI, чаще всего Airflow не может его распарсить. В UI при этом будет видна ошибка импорта (Import Error). Текст ошибки подсказывает, что исправить.

Чтобы посмотреть ошибки импорта в терминале: с Astro CLI — `astro dev run dags list-import-errors`, с Airflow CLI — `airflow dags list-import-errors`.

Если сообщения об ошибке импорта нет, но DAG в UI по-прежнему нет:

- Если в UI указано, что планировщик не запущен, проверьте логи планировщика — возможно, падение из-за ошибки в DAG. В Astro CLI: `astro dev logs -s`, затем перезапуск.
- Перезапустите планировщик: `astro dev restart`.
- Убедитесь, что DAG зарегистрирован в БД метаданных: `astro dev run dags list` или `airflow dags list`. Если DAG есть в списке, но не в UI, перезапустите Airflow: `astro dev restart`.
- Проверьте права доступа к DAG и права на просмотр DAG.
- Airflow сканирует каталог `dags` с интервалом [`dag_dir_list_interval`](https://airflow.apache.org/docs/apache-airflow/stable/configurations-ref.html#dag-dir-list-interval) (по умолчанию 5 минут). Принудительный перепарсинг: `astro dev run dags reserialize` / `airflow dags reserialize`.
- Убедитесь, что все DAG лежат в каталоге `dags`.

На уровне кода проверьте, что:

- DAG, определённый через декоратор `@dag`, **вызывается** (см. [Декораторы Airflow](airflow-decorators.md)).
- У DAG уникальный `dag_id`.

Через плагин можно настроить [Airflow listener](../04.%20astronomer-advanced/airflow-plugins.md), который выполняет код при появлении новой ошибки импорта (`on_new_dag_import_error`) или при повторной обработке известной ошибки (`on_existing_dag_import_error`).

### Ошибки импорта из-за зависимостей

Частая причина ошибок импорта — отсутствие нужных пакетов в окружении Airflow: нет [provider packages](https://registry.astronomer.io/providers) для используемых операторов/хуков или нет Python-пакетов, которые вызываются в задачах.

В проекте Astro пакеты уровня ОС добавляют в `packages.txt`, Python-пакеты (в том числе провайдеры) — в `requirements.txt`. Если нужен другой менеджер пакетов, можно добавить команду в Dockerfile.

Чтобы избежать несовместимости при выходе новых версий, рекомендуется **фиксировать версии** в проекте. Например, `apache-airflow-providers-amazon==9.6.0` в `requirements.txt` не даст подтянуться несовместимой новой версии. Без фиксации Airflow будет использовать последнюю доступную версию.

В Astro CLI пакеты ставятся в контейнер планировщика. Проверить установку пакета:

```sh
astro dev bash --scheduler "pip freeze | grep <package-name>"
```

При конфликте версий или необходимости разных версий Python задачи можно выполнять в отдельных окружениях:

- [PythonVirtualEnvOperator](https://registry.astronomer.io/providers/apache-airflow/modules/pythonvirtualenvoperator): задача во временном виртуальном окружении.
- [ExternalPythonOperator](../04.%20astronomer-advanced/isolated-environments.md): задача в заранее созданном виртуальном окружении.
- [KubernetesPodOperator](../04.%20astronomer-advanced/kubernetes-pod-operator.md): задача в отдельном Pod в Kubernetes.

Если у многих задач общий набор пакетов и версий, часто используют два и более отдельных развёртывания Airflow.

### DAG не запускаются или ведут себя не так

Если DAG не запускаются или работают не так, как задумано, проверьте:

- Если не запускается ни один DAG — состояние планировщика: `astro dev logs -s`.
- Протестировать DAG: `astro dev dags test <dag_id> <execution_date>` или `airflow dags test <dag_id> <execution_date>`.
- У DAG должен быть **start_date в прошлом**. При start_date в будущем DAG run будет успешным, но без запуска задач. Не используйте `datetime.now()` в качестве start_date.
- У каждого DAG должен быть **уникальный dag_id**. При двух DAG с одним id планировщик раз в 30 секунд случайно выбирает один из них для отображения.
- DAG должны быть **включены** (не на паузе), чтобы запускаться по расписанию. Включить можно переключателем в UI или через [Airflow CLI](https://airflow.apache.org/docs/apache-airflow/stable/cli-and-env-variables-ref.html#unpause). Чтобы все DAG по умолчанию были включены, задайте в конфиге [`dags_are_paused_at_creation=False`](https://airflow.apache.org/docs/apache-airflow/stable/configurations-ref.html#dag-dir-list-interval).

Если DAG запускается, но не по ожидаемому расписанию, см. [Планирование в Airflow](../01.%20astronomer-basic/scheduling.md). При кастомном timetable убедитесь, что data interval DAG run не раньше start_date DAG.

## Типичные проблемы с задачами

Если не работает весь DAG, см. раздел «DAG не запускаются или ведут себя не так» выше.

> Между Airflow 2 и Airflow 3 существенно изменилась архитектура (безопасность, remote execution и др.). Для авторов DAG важно: **прямой доступ к БД метаданных из задач больше недоступен**. Подробнее: [Upgrade from Airflow 2 to 3](https://www.astronomer.io/docs/learn/airflow-upgrade-2-3) и [Release notes](https://airflow.apache.org/docs/apache-airflow/stable/release_notes.html).

### Задачи не запускаются или ведут себя не так

DAG может стартовать, но задачи остаются в разных состояниях или выполняются не в нужном порядке. Что проверить:

- При CeleryExecutor в версиях Airflow до 2.6, если задачи застревают в `queued`, можно включить [`stalled_task_timeout`](https://airflow.apache.org/docs/apache-airflow/stable/configurations-ref.html#stalled-task-timeout).
- Параметр [`task_queued_timeout`](https://airflow.apache.org/docs/apache-airflow/stable/configurations-ref.html#task-queued-timeout) задаёт, сколько задача может находиться в очереди перед повтором или пометкой failed. По умолчанию 600 секунд.
- Проверьте зависимости задач и [trigger rules](../01.%20astronomer-basic/trigger-rules.md). См. [Управление зависимостями](../01.%20astronomer-basic/task-dependencies.md). Можно временно пересобрать DAG из EmptyOperator, чтобы убедиться в правильной структуре зависимостей.
- При использовании декораторов задач убедитесь, что **задачи вызываются** (см. [Декораторы Airflow](airflow-decorators.md)).
- При большом числе экземпляров задач или DAG учитывайте параметры масштабирования и лимиты по умолчанию. Подробнее: [Scaling Airflow](../03.%20astronomer-infra/scaling-airflow.md).
- Если у задачи `depends_on_past=True`, новые задачи не запустятся, пока не задано состояние предыдущих run этой задачи.
- Если задачи остаются в `scheduled` или `queued`, проверьте работу планировщика. При необходимости перезапустите его или увеличьте ресурсы.
- Ещё раз проверьте, что **start_date** DAG в прошлом. start_date в будущем даёт успешный DAG run без выполнения задач.

### Задачи падают

Большинство падений задач относится к одному из трёх типов:

- Проблемы во внешней системе.
- Ошибки внутри оператора.
- Некорректные параметры оператора.

Упавшие задачи отображаются красными квадратами в Grid view; оттуда же открываются логи задачи. В логах видна причина ошибки.

Для быстрого реагирования на падения настройте уведомления. См. [Уведомления об ошибках в Airflow](airflow-notifications.md).

В новых DAG сообщения вроде `Task exited with return code Negsignal.SIGKILL` или код `-9` часто связаны с **нехваткой памяти**. Увеличьте ресурсы планировщика, webserver или pod в зависимости от типа executor (Local, Celery, Kubernetes).

> После исправления может понадобиться перезапуск DAG или задач. См. [Rerunning DAGs](rerunning-dags.md).

### Проблемы с динамически маппленными задачами

[Динамический маппинг задач](dynamic-tasks.md) позволяет менять число задач в рантайме по входным параметрам. Поддерживается и [динамический маппинг по task groups](task-groups.md).

Возможные причины проблем:

- Не все параметры поддерживают маппинг; при неподдерживаемом параметре будет ошибка импорта.
- Число одновременно выполняемых маппленных экземпляров одной задачи во всех run DAG ограничено параметром задачи `max_active_tis_per_dag`.
- Превышен лимит числа маппленных экземпляров. Лимит задаётся конфигом [`max_map_length`](https://airflow.apache.org/docs/apache-airflow/stable/configurations-ref.html#max-map-length) (по умолчанию 1024).
- Маппинг по пустому списку — маппленная задача будет пропущена.
- При `.expand_kwargs()` параметры должны быть в виде `List(Dict)`.
- В `.expand()` нужно передавать именованный аргумент (keyword argument).

При сложных сценариях с динамическим маппингом сначала соберите структуру DAG из EmptyOperator или задекорированных Python-операторов, убедитесь, что она верна, затем подставляйте нужные задачи. Примеры: [Create dynamic Airflow tasks](dynamic-tasks.md).

> Часто вывод вышестоящего оператора не совпадает по формату с тем, что нужно для маппинга. Используйте [`.map()`](dynamic-tasks.md) для преобразования элементов списка через Python-функцию.

## Отсутствующие логи

При отладке падения задачи логов может не быть: на странице логов в UI — «спиннер» или пустой файл.

Обычно логи не появляются, когда процесс в планировщике или воркере завершается и связь теряется. Что можно сделать:

- Проверьте логи планировщика и webserver на ошибки, связанные с отсутствием логов.
- Убедитесь, что логи хранятся достаточно долго. Клиентам Astronomer: [View logs](https://www.astronomer.io/docs/astro/view-logs).
- Увеличьте CPU или память для задачи.
- При Kubernetes executor, если задача падает очень быстро (менее 15 секунд), pod может завершиться до того, как webserver успеет забрать логи. По возможности добавьте небольшую задержку в задачу (в зависимости от оператора). Иначе ищите причину немедленного падения (часто — нехватка ресурсов или ошибка конфигурации).
- Увеличьте ресурсы воркеров (Celery) или планировщика (Local executor).
- Увеличьте параметр `log_fetch_timeout_sec` выше значения по умолчанию (5 секунд). Он задаёт время ожидания webserver при первом контакте с воркером при получении логов.
- Попробуйте перезапустить задачу через [clearing task instance](rerunning-dags.md) — иногда логи появляются при повторном запуске.

## Отладка подключений

Подключения в Airflow обычно нужны для обмена с внешними системами. Большинство хуков и операторов ожидают заданный параметр подключения. Поэтому некорректно заданные подключения — одна из самых частых причин отладки при первых шагах с DAG.

Конкретная ошибка может быть разной, но в логах задачи часто фигурирует слово «connection». Если подключение не задано, может быть сообщение вида `'connection_abc' is not defined`.

Что проверить:

- Работают ли учётные данные при прямом обращении к API внешней системы.
- Задавайте подключения через переменные окружения Airflow, а не только в UI. Не задавайте одно и то же подключение в нескольких местах; при дублировании приоритет у переменной окружения.
- Измените подключение `_default` на нужные данные или создайте новое подключение с другим именем и передайте его в хук или оператор.
- Установлены ли нужные provider packages для данного типа подключения.
- Как устроены подключения: [Управление подключениями в Airflow](../01.%20astronomer-basic/connections.md).

Чтобы узнать, какие параметры нужны для конкретного типа подключения:

- Посмотрите исходный код хука, который использует оператор.
- Изучите документацию внешней системы по аутентификации.
- В [Astronomer Registry](https://registry.astronomer.io/providers?page=1) откройте документацию провайдера; у популярных провайдеров обычно описаны типы подключений (например, Azure).

## Нужна дополнительная помощь

Этого руководства обычно достаточно для типичных случаев. Если вашей проблемы в нём нет:

- При баге в Airflow или основных провайдерах создайте issue в [репозитории Airflow](https://github.com/apache/airflow/issues). Для open source инструментов Astronomer — в [репозиториях Astronomer](https://github.com/astronomer).
- [Apache Airflow Slack](https://apache-airflow-slack.herokuapp.com/): каналы `#newbie-questions` или `#troubleshooting` — удобное место для сложных вопросов по Airflow.
- [Stack Overflow](https://stackoverflow.com/) с тегами `airflow` и другими по используемым инструментам. Удобно, когда непонятно, в каком компоненте ошибка — вопрос увидят специалисты по разным системам.
- Клиенты Astronomer: [поддержка](https://support.astronomer.io/).

Чтобы получить более точный ответ, включите в вопрос или issue:

- Что изменилось в окружении, когда появилась проблема.
- Что именно вы хотите сделать (максимально подробно).
- Полный код DAG, если ошибка в нём.
- Полный текст ошибки и traceback.
- Версию Airflow и версии задействованных провайдеров.
- Как запускается Airflow (Astro CLI, standalone, Docker, managed-сервис).

---

[← Версионирование](dag-versioning.md) | [К содержанию](README.md) | [Лучшие практики →](dag-best-practices.md)
