# Установка uv

## Способы установки

Установите uv с помощью автономного установщика или любого удобного пакетного менеджера.

### Автономный установщик

uv предоставляет автономный установщик для загрузки и установки:

**macOS и Linux**

Скачайте скрипт через `curl` и выполните его через `sh`:

```bash
$ curl -LsSf https://astral.sh/uv/install.sh | sh
```

Если в системе нет `curl`, можно использовать `wget`:

```bash
$ wget -qO- https://astral.sh/uv/install.sh | sh
```

Чтобы запросить конкретную версию, укажите её в URL:

```bash
$ curl -LsSf https://astral.sh/uv/0.10.7/install.sh | sh
```

**Windows**

Скачайте скрипт через `irm` и выполните его через `iex`:

```powershell
PS> powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

Изменение [политики выполнения](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_execution_policies?view=powershell-7.4#powershell-execution-policies) позволяет запускать скрипты из интернета.

Чтобы запросить конкретную версию, укажите её в URL:

```powershell
PS> powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/0.10.7/install.ps1 | iex"
```

**Совет:** Скрипт установки можно просмотреть перед запуском:

**macOS и Linux:**

```bash
$ curl -LsSf https://astral.sh/uv/install.sh | less
```

**Windows:**

```powershell
PS> powershell -c "irm https://astral.sh/uv/install.ps1 | more"
```

Либо установщик или бинарники можно скачать напрямую с GitHub.

Подробнее о настройке установки uv см. в [справочной документации по установщику](https://docs.astral.sh/uv/reference/installer/).

### PyPI

uv публикуется на [PyPI](https://pypi.org/project/uv/) для удобства.

При установке из PyPI рекомендуется ставить uv в изолированное окружение, например через `pipx`:

```bash
$ pipx install uv
```

Можно также использовать `pip`:

```bash
$ pip install uv
```

**Примечание:** uv поставляется с предсобранными дистрибутивами (wheels) для многих платформ; если wheel для вашей платформы недоступен, uv будет собран из исходников, для чего нужна цепочка инструментов Rust. Подробнее о сборке uv из исходников см. в [руководстве по настройке для разработки](https://github.com/astral-sh/uv/blob/main/CONTRIBUTING.md#setup).

### Homebrew

uv доступен в основных пакетах Homebrew.

```bash
$ brew install uv
```

### MacPorts

uv доступен через [MacPorts](https://ports.macports.org/port/uv/).

```bash
$ sudo port install uv
```

### WinGet

uv доступен через [WinGet](https://winstall.app/apps/astral-sh.uv).

```powershell
$ winget install --id=astral-sh.uv  -e
```

### Scoop

uv доступен через [Scoop](https://scoop.sh/#/apps?q=uv).

```powershell
$ scoop install main/uv
```

### Docker

uv предоставляет Docker-образ в [ghcr.io/astral-sh/uv](https://github.com/astral-sh/uv/pkgs/container/uv).

Подробнее см. в руководстве [использование uv в Docker](https://docs.astral.sh/uv/guides/integration/docker/).

### GitHub Releases

Артефакты релизов uv можно скачать напрямую с [GitHub Releases](https://github.com/astral-sh/uv/releases).

На странице каждого релиза есть бинарники для всех поддерживаемых платформ и инструкции по использованию автономного установщика через `github.com` вместо `astral.sh`.

### Cargo

uv доступен через [crates.io](https://crates.io/).

```bash
$ cargo install --locked uv
```

**Примечание:** Этот способ собирает uv из исходников и требует совместимой цепочки инструментов Rust.

## Обновление uv

При установке через автономный установщик uv может обновляться по запросу:

```bash
$ uv self update
```

**Совет:** Обновление uv перезапускает установщик и может изменить профили оболочки. Чтобы отключить это, задайте `UV_NO_MODIFY_PATH=1`.

При установке другим способом самообновление отключено. Используйте способ обновления вашего пакетного менеджера. Например, для `pip`:

```bash
$ pip install --upgrade uv
```

## Автодополнение в оболочке

**Совет:** Команда `echo $SHELL` подскажет, какая у вас оболочка.

Чтобы включить автодополнение команд uv в оболочке, выполните одну из следующих команд:

**Bash:**

```bash
echo 'eval "$(uv generate-shell-completion bash)"' >> ~/.bashrc
```

**Zsh:**

```bash
echo 'eval "$(uv generate-shell-completion zsh)"' >> ~/.zshrc
```

**fish:**

```bash
echo 'uv generate-shell-completion fish | source' > ~/.config/fish/completions/uv.fish
```

**Elvish:**

```bash
echo 'eval (uv generate-shell-completion elvish | slurp)' >> ~/.elvish/rc.elv
```

**PowerShell / pwsh:**

```powershell
if (!(Test-Path -Path $PROFILE)) {
  New-Item -ItemType File -Path $PROFILE -Force
}
Add-Content -Path $PROFILE -Value '(& uv generate-shell-completion powershell) | Out-String | Invoke-Expression'
```

Чтобы включить автодополнение для uvx, выполните одну из следующих команд:

**Bash:**

```bash
echo 'eval "$(uvx --generate-shell-completion bash)"' >> ~/.bashrc
```

**Zsh:**

```bash
echo 'eval "$(uvx --generate-shell-completion zsh)"' >> ~/.zshrc
```

**fish:**

```bash
echo 'uvx --generate-shell-completion fish | source' > ~/.config/fish/completions/uvx.fish
```

**Elvish:**

```bash
echo 'eval (uvx --generate-shell-completion elvish | slurp)' >> ~/.elvish/rc.elv
```

**PowerShell / pwsh:**

```powershell
if (!(Test-Path -Path $PROFILE)) {
  New-Item -ItemType File -Path $PROFILE -Force
}
Add-Content -Path $PROFILE -Value '(& uvx --generate-shell-completion powershell) | Out-String | Invoke-Expression'
```

После этого перезапустите оболочку или выполните `source` для файла конфигурации оболочки.

## Удаление

Чтобы удалить uv с системы, выполните следующие шаги.

**Очистка сохранённых данных (по желанию):**

```bash
$ uv cache clean
$ rm -r "$(uv python dir)"
$ rm -r "$(uv tool dir)"
```

**Совет:** Перед удалением бинарников можно удалить данные, которые сохранял uv. Подробнее о расположении данных см. в [справочнике по хранилищу](https://docs.astral.sh/uv/reference/storage/).

**Удаление бинарников uv, uvx и uvw:**

**macOS и Linux:**

```bash
$ rm ~/.local/bin/uv ~/.local/bin/uvx
```

**Windows:**

```powershell
PS> rm $HOME\.local\bin\uv.exe
PS> rm $HOME\.local\bin\uvx.exe
PS> rm $HOME\.local\bin\uvw.exe
```

**Примечание:** До версии 0.5.0 uv устанавливался в `~/.cargo/bin`. Бинарники можно удалить оттуда для полного удаления. Обновление со старой версии не удаляет бинарники из `~/.cargo/bin` автоматически.

## Дальнейшие шаги

См. [первые шаги](https://docs.astral.sh/uv/getting-started/first-steps/) или перейдите к [руководствам](https://docs.astral.sh/uv/guides/), чтобы начать использовать uv.
