# VS Code: локальная разработка

Настройка [VS Code](https://code.visualstudio.com/) для локальной разработки DAG с [Astro CLI](https://www.astronomer.io/docs/astro/cli/overview): автодополнение, подсветка ошибок, работа внутри контейнера.

## Требования

- Astro-проект, запущенный локально (`astro dev start`).
- Astro CLI.
- Расширение [Dev Containers](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers).
- VS Code.

## Шаги

1. Открыть папку Astro-проекта в VS Code.
2. В левом нижнем углу нажать на иконку контейнеров → **Open Folder in Container...**.
3. Выбрать **From 'Dockerfile'** — откроется новое окно с подключением к контейнеру (в углу будет индикатор).
4. Установить расширение **Python** (Microsoft) в контейнере: Extensions → Python → Install in Container.
5. Открыть файл из `dags/` (например, `example_dag_basic.py`) — интерпретатор должен определиться автоматически.

![Open Folder in Container](images/vscode_open_folder.png)

![Автодополнение в VS Code](images/vscode_code.png)

После настройки VS Code показывает предупреждения и автодополнение для Airflow API.

Подробнее: [VS Code local development](https://www.astronomer.io/docs/learn/vscode-local-dev), [Debug with dag.test()](../astronomer-advanced/testing-airflow.md).

---

[← SQL check operators](sql-check-operators.md) | [К содержанию](README.md)
