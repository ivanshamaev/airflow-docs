# Установка Ruff (Installing Ruff)

Ruff доступен на PyPI как пакет [ruff](https://pypi.org/project/ruff/).

Ruff можно запускать напрямую через [uvx](https://docs.astral.sh/uv/):

```bash
uvx ruff check   # Проверить линтером все файлы в текущем каталоге.
uvx ruff format  # Отформатировать все файлы в текущем каталоге.
```

Или установить через `uv` (рекомендуется), `pip` или `pipx`:

```bash
# Установка Ruff глобально.
uv tool install ruff@latest

# Или добавить Ruff в проект.
uv add --dev ruff

# Через pip.
pip install ruff

# Через pipx.
pipx install ruff
```

После установки Ruff можно вызывать из командной строки:

```bash
ruff check   # Проверить линтером все файлы в текущем каталоге.
ruff format  # Отформатировать все файлы в текущем каталоге.
```

Начиная с версии `0.5.0`, Ruff также можно установить автономными установщиками:

```bash
# macOS и Linux.
curl -LsSf https://astral.sh/ruff/install.sh | sh

# Windows.
powershell -c "irm https://astral.sh/ruff/install.ps1 | iex"

# Конкретная версия.
curl -LsSf https://astral.sh/ruff/0.5.0/install.sh | sh
powershell -c "irm https://astral.sh/ruff/0.5.0/install.ps1 | iex"
```

Пользователи **Homebrew** (macOS и Linuxbrew) могут установить Ruff из [formula ruff](https://formulae.brew.sh/formula/ruff):

```bash
brew install ruff
```

Пользователи **Conda** — из канала [conda-forge](https://anaconda.org/conda-forge/ruff):

```bash
conda install -c conda-forge ruff
```

Пользователи **pkgx** — из [реестра pkgx](https://pkgx.dev/pkgs/github.com/charliermarsh/ruff/):

```bash
pkgx install ruff
```

Пользователи **Arch Linux** — из [официальных репозиториев](https://archlinux.org/packages/extra/x86_64/ruff/):

```bash
pacman -S ruff
```

Пользователи **Alpine** — из [тестовых репозиториев](https://pkgs.alpinelinux.org/package/edge/testing/x86_64/ruff):

```bash
apk add ruff
```

Пользователи **openSUSE Tumbleweed** — из репозитория дистрибутива:

```bash
sudo zypper install python3-ruff
```

**Docker**: образ публикуется как `ghcr.io/astral-sh/ruff` с тегами для каждого релиза и тегом `latest` для последней версии.

```bash
docker run -v .:/io --rm ghcr.io/astral-sh/ruff check
docker run -v .:/io --rm ghcr.io/astral-sh/ruff:0.3.0 check

# Для Podman на SELinux.
docker run -v .:/io:Z --rm ghcr.io/astral-sh/ruff check
```
