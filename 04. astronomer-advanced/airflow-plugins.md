# Плагины Airflow (Airflow plugins)

Плагины позволяют расширять установку Airflow: кастомный UI, API, макросы. В Airflow 3.1 добавлена поддержка плагинов через plugin manager (внешние представления, React-приложения, FastAPI, макросы). Компоненты на базе Flask AppBuilder в Airflow 3 устарели; новые плагины используют [External views](https://www.astronomer.io/docs/learn/using-airflow-plugins#external-views), [React apps](https://www.astronomer.io/docs/learn/using-airflow-plugins#react-apps), [FastAPI apps](https://www.astronomer.io/docs/learn/using-airflow-plugins#fastapi-apps), [Middlewares](https://www.astronomer.io/docs/learn/using-airflow-plugins#middlewares).

Плагин — класс, наследующий **AirflowPlugin**, с атрибутами: `external_views`, `react_apps`, `macros`, `fastapi_apps`, `fastapi_root_middlewares`, `global_operator_extra_links`, `operator_extra_links`, `timetables`, `listeners`. Регистрация: Python-файл в папке `plugins`. Загруженные плагины: Admin → Plugins. После изменений нужен перезапуск API server (или `AIRFLOW__CORE__LAZY_LOAD_PLUGINS=False` для авто-перезагрузки).

Компоненты: **External views** — дополнительные страницы/вкладки (iframe или FastAPI); **React apps** — вложение React-приложения (dashboard, dag_overview, task_overview или nav); **Macros** — функции для Jinja в шаблонируемых полях; **FastAPI apps** — свои эндпоинты; **Middlewares** — обработка запросов/ответов API; **Operator extra links** — кнопки в Details задачи (глобальные или для конкретного оператора). На Astro для ссылок на FastAPI используйте относительные пути в `href` (без ведущего `/`).

Подробнее: [Using Airflow plugins](https://www.astronomer.io/docs/learn/using-airflow-plugins), [Airflow plugins](https://airflow.apache.org/docs/apache-airflow/stable/plugins.html).

---

[← MLOps](airflow-mlops.md) | [К содержанию](README.md) | [Пуллы →](airflow-pools.md)
