# Интеграция с редакторами

ty можно интегрировать с различными редакторами, чтобы обеспечить более удобный процесс разработки.

Больше о возможностях ty в редакторах см. в документации по [языковому серверу](https://docs.astral.sh/ty/features/language-server/).

## VS Code

Команда Astral поддерживает официальный плагин для VS Code.

Установите [расширение ty](https://marketplace.visualstudio.com/items?itemName=astral-sh.ty) из VS Code Marketplace.

Расширение автоматически отключает языковой сервер из [Python extension](https://marketplace.visualstudio.com/items?itemName=ms-python.python), чтобы не запускать два Python-языковых сервера одновременно. Это делается путём установки параметра [python.languageServer](https://code.visualstudio.com/docs/python/settings-reference#_intellisense-engine-settings) в значение `"None"` в настройках по умолчанию.

Если вы хотите использовать ty только для проверки типов и при этом использовать другой языковой сервер для таких возможностей, как подсказки при наведении, автодополнение и т. п., вы можете переопределить это поведение, явно задав значения [python.languageServer](https://code.visualstudio.com/docs/python/settings-reference#_intellisense-engine-settings) и [ty.disableLanguageServices](https://docs.astral.sh/ty/reference/editor-settings/#disablelanguageservices) в вашем файле [settings.json](https://code.visualstudio.com/docs/configure/settings#_settings-json-file):

```json
{
  "python.languageServer": "Pylance",
  "ty.disableLanguageServices": true,
}
```

## Neovim

Рекомендуемый способ использовать ty с Neovim — через расширение [nvim-lspconfig](https://github.com/neovim/nvim-lspconfig) (если вы не хотите устанавливать расширение, можно просто скопировать [конфигурацию ty](https://github.com/neovim/nvim-lspconfig/blob/master/lsp/ty.lua) вручную). После установки nvim-lspconfig нужно включить языковой сервер (и при желании можно настроить дополнительные параметры). Для Neovim версии >= 0.11 вы можете добавить в конфигурационный файл следующий фрагмент:

```lua
-- Необязательно: нужно только в том случае, если вы хотите настраивать параметры языкового сервера
vim.lsp.config('ty', {
  settings = {
    ty = {
      -- настройки языкового сервера ty указываются здесь
    }
  }
})

-- Обязательно: включение языкового сервера
vim.lsp.enable('ty')
```

Для Neovim версии < 0.11 вместо этого используется конфигурация ниже (обратите внимание, что [возможно, придётся установить более старую версию nvim-lspconfig](https://github.com/neovim/nvim-lspconfig?tab=readme-ov-file#important-%EF%B8%8F)):

```lua
require('lspconfig').ty.setup({
  settings = {
    ty = {
      -- настройки языкового сервера ty указываются здесь
    }
  }
})
```

## Zed

ty включён в Zed «из коробки» (дополнительное расширение не требуется), хотя основным LSP по умолчанию для Python является basedpyright.

Вы можете включить ty и отключить basedpyright, добавив следующее в ваш файл `settings.json`:

```json
{
  "languages": {
    "Python": {
      "language_servers": [
        // Отключить basedpyright и включить ty, а все остальные
        // настройки оставить по умолчанию.
        "ty",
        "!basedpyright",
        "..."
      ]
    }
  }
}
```

Вы можете переопределить исполняемый файл `ty`, который использует Zed, через настройку `lsp.ty.binary`:

```json
{
  "lsp": {
    "ty": {
      "binary": {
        "path": "/home/user/.local/bin/ty",
        "arguments": ["server"]
      }
    }
  }
}
```

Подробнее см. в [документации Zed](https://zed.dev/docs/languages/python#configure-python-language-servers-in-zed).

## PyCharm

Начиная с версии 2025.3, пользователи PyCharm могут включить встроенную поддержку ty в настройках:

Откройте раздел Python | Tools | ty в диалоговом окне Settings.

Установите флажок Enable.

В параметре Execution mode выберите, как PyCharm должен искать исполняемый файл:

- Режим Interpreter: PyCharm ищет исполняемый файл, установленный в вашем интерпретаторе. Чтобы установить пакет ty для выбранного интерпретатора, нажмите Install ty.
- Режим Path: PyCharm ищет исполняемый файл в `$PATH`. Если исполняемый файл не найден, вы можете указать путь вручную, нажав кнопку Browse....

Выберите, какие опции должны быть включены.

Дополнительные сведения см. в [документации PyCharm](https://www.jetbrains.com/help/pycharm/lsp-tools.html#ty).

## Другие редакторы

ty можно использовать с любым редактором, поддерживающим [протокол языкового сервера](https://microsoft.github.io/language-server-protocol/).

Чтобы запустить языковой сервер, используйте подкоманду `server`:

```bash
ty server
```

Обратитесь к документации вашего редактора, чтобы узнать, как подключиться к LSP-серверу.

## Настройки

Подробнее о настройке языкового сервера см. в [справочнике по настройкам редакторов](https://docs.astral.sh/ty/reference/editor-settings/).

