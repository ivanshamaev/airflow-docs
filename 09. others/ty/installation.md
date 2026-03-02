# Установка ty

## Запуск ty без установки

Используйте [uvx](https://docs.astral.sh/uv/guides/tools/), чтобы быстро начать работу с ty:

```
uvx ty

```

## Способы установки

### Добавление ty в проект

Tip

Добавление ty в зависимости гарантирует, что все разработчики проекта используют одну и ту же версию ty.

Используйте [uv](https://github.com/astral-sh/uv) (или другой менеджер пакетов) для добавления ty в качестве dev-зависимости:

```
uv add --dev ty

```

Затем вызывайте ty через `uv run`:

```
uv run ty

```

Для обновления ty используйте `--upgrade-package`:

```
uv lock --upgrade-package ty

```

### Глобальная установка через uv

Установите ty глобально с помощью uv:

```
uv tool install ty@latest

```

Для обновления ty используйте `uv tool upgrade`:

```
uv tool upgrade ty

```

### Установка через standalone-инсталлятор

ty поставляется со standalone-инсталлятором.

macOS и Linux / Windows

Используйте `curl` для загрузки скрипта и выполнения его через `sh`:

```
$ curl -LsSf https://astral.sh/ty/install.sh | sh

```

Если в системе нет `curl`, можно использовать `wget`:

```
$ wget -qO- https://astral.sh/ty/install.sh | sh

```

Чтобы запросить конкретную версию, укажите её в URL:

```
$ curl -LsSf https://astral.sh/ty/0.0.20/install.sh | sh

```

Используйте `irm` для загрузки скрипта и выполнения его через `iex`:

```
PS> powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/ty/install.ps1 | iex"

```

Изменение [политики выполнения](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_execution_policies?view=powershell-7.4#powershell-execution-policies) позволяет запускать скрипты из интернета.

Чтобы запросить конкретную версию, укажите её в URL:

```
PS> powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/ty/0.0.20/install.ps1 | iex"

```

Tip

Скрипт установки можно просмотреть перед использованием:

macOS и Linux / Windows

```
$ curl -LsSf https://astral.sh/ty/install.sh | less

```

```
PS> powershell -c "irm https://astral.sh/ty/install.ps1 | more"

```

Альтернативно, инсталлятор или бинарные файлы можно скачать напрямую с GitHub.

### Установка из GitHub Releases

Артефакты релизов ty можно скачать с [GitHub Releases](https://github.com/astral-sh/ty/releases).

На каждой странице релиза есть бинарные файлы для всех поддерживаемых платформ, а также инструкции по использованию standalone-инсталлятора через `github.com` вместо `astral.sh`.

### Глобальная установка через pipx

Установите ty глобально с помощью pipx:

```
pipx install ty

```

Для обновления ty используйте `pipx upgrade`:

```
pipx upgrade ty

```

### Установка через pip

Установите ty в текущее Python-окружение с помощью pip:

```
pip install ty

```

### Глобальная установка через mise

Установите ty глобально с помощью [mise](https://github.com/jdx/mise):

```
mise install ty

```

Чтобы задать его глобально:

```
mise use --global ty

```

### Установка в Docker

Установите ty в Docker, скопировав бинарный файл из официального образа:

Dockerfile

```
COPY --from=ghcr.io/astral-sh/ty:latest /ty /bin/

```

Доступны следующие теги:

- `ghcr.io/astral-sh/ty:{major}.{minor}`, например `ghcr.io/astral-sh/ty:0.0` (последняя patch-версия)
- `ghcr.io/astral-sh/ty:{major}.{minor}.{patch}`, например `ghcr.io/astral-sh/ty:0.0.20`
- `ghcr.io/astral-sh/ty:latest`

### Использование ty с Bazel

[aspect_rules_lint](https://registry.bazel.build/docs/aspect_rules_lint#function-lint_ty_aspect) предоставляет Bazel lint aspect для запуска ty. Инструкции по настройке см. в его документации.

## Добавление ty в редактор

См. руководство по [интеграции с редакторами](https://docs.astral.sh/ty/editors/), чтобы добавить ty в ваш редактор.

## Автодополнение в оболочке

Tip

Команда `echo $SHELL` поможет определить вашу оболочку.

Чтобы включить автодополнение команд ty в оболочке, выполните одну из следующих команд:

Bash / Zsh / fish / Elvish / PowerShell / pwsh

```
echo 'eval "$(ty generate-shell-completion bash)"' >> ~/.bashrc

```

```
echo 'eval "$(ty generate-shell-completion zsh)"' >> ~/.zshrc

```

```
echo 'ty generate-shell-completion fish | source' > ~/.config/fish/completions/ty.fish

```

```
echo 'eval (ty generate-shell-completion elvish | slurp)' >> ~/.elvish/rc.elv

```

```
if (!(Test-Path -Path $PROFILE)) {
  New-Item -ItemType File -Path $PROFILE -Force
}
Add-Content -Path $PROFILE -Value '(& ty generate-shell-completion powershell) | Out-String | Invoke-Expression'

```

Затем перезапустите оболочку или выполните `source` для файла конфигурации оболочки.
