# Версия Python

Версия Python влияет на допустимый синтаксис, определения типов стандартной библиотеки, а также на определения типов модулей первого и стороннего уровня, зависящие от версии Python.

Например, в Python 3.10 появилась поддержка операторов `match`, а также был добавлен символ `sys.stdlib_module_names` в стандартную библиотеку. Синтаксические возможности всегда должны быть доступны в самой низкой поддерживаемой версии Python, но символы могут использоваться в ветках, зависящих от `sys.version_info`:

```python
import sys

# ошибка `invalid-syntax`, если `python-version` установлен в 3.9 или ниже:
match "echo hello".split():
    case ["echo", message]:
        print(message)
    case _:
        print("unknown command")

# ошибка `unresolved-attribute`, если `python-version` установлен в 3.9 или ниже:
print(sys.stdlib_module_names)

if sys.version_info >= (3, 10):
    # ok, потому что использование защищено проверкой версии:
    print(sys.stdlib_module_names)
```

По умолчанию в качестве целевой версии Python используется нижняя граница поля [requires-python](https://packaging.python.org/en/latest/guides/writing-pyproject-toml/#python-requires) проекта (из `pyproject.toml`), что гарантирует, что функции и символы, доступные только в более новых версиях Python, не будут использоваться.

Если поле `requires-python` отсутствует, но виртуальное окружение настроено или обнаружено, ty попытается вывести используемую версию Python из метаданных виртуального окружения.

Если виртуальное окружение отсутствует или определить версию Python по метаданным не удалось, ty вернётся к последней стабильной версии Python, поддерживаемой ty (в настоящее время 3.14).

Версию Python также можно явно указать с помощью настройки [python-version](https://docs.astral.sh/ty/reference/configuration/#python-version) или флага [--python-version](https://docs.astral.sh/ty/reference/cli/#ty-check--python-version).

