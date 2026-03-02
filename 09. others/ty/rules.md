# Правила

Правила — это отдельные проверки, которые ty выполняет для выявления типичных проблем в коде: несовместимые присваивания, отсутствующие импорты или некорректные аннотации типов. Каждое правило отвечает за определённый паттерн и может быть включено или отключено в зависимости от потребностей проекта.

Tip

Полный перечень поддерживаемых правил см. в [справочнике по правилам](https://docs.astral.sh/ty/reference/rules/).

## Уровни правил

У каждого правила настраивается уровень:

- `ignore`: правило отключено
- `warn`: нарушения сообщаются как предупреждения. В зависимости от настроек ty завершается с кодом 0, если есть только предупреждения (по умолчанию), или с кодом 1 при использовании `--error-on-warning`
- `error`: нарушения считаются ошибками, и ty завершается с кодом 1 при наличии любых таких нарушений

Уровень для каждого правила можно задать в командной строке флагами `--warn`, `--error` и `--ignore`. Например:

```bash
ty check \
  --warn unused-ignore-comment \        # Сделать `unused-ignore-comment` предупреждением
  --ignore redundant-cast \             # Отключить `redundant-cast`
  --error possibly-missing-attribute \  # Ошибка для `possibly-missing-attribute`
  --error possibly-missing-import       # Ошибка для `possibly-missing-import`
```

Опции можно повторять. Более поздние опции переопределяют более ранние.

Уровни правил также можно менять в секции [rules](https://docs.astral.sh/ty/reference/configuration/#rules) [конфигурационного файла](https://docs.astral.sh/ty/configuration/).

Например, следующая конфигурация эквивалентна приведённой выше команде:

pyproject.toml

```toml
[tool.ty.rules]
unused-ignore-comment = "warn"
redundant-cast = "ignore"
possibly-missing-attribute = "error"
possibly-missing-import = "error"
```

Уровень можно задать и для всех правил сразу.

В командной строке можно использовать `--error all`, `--warn all` или `--ignore all`. Например:

```bash
ty check --error all
```

То же самое можно указать в секции [rules](https://docs.astral.sh/ty/reference/configuration/#rules) [конфигурационного файла](https://docs.astral.sh/ty/configuration/).

Например, следующая конфигурация эквивалентна приведённой выше команде:

pyproject.toml

```toml
[tool.ty.rules]
all = "error"
```
