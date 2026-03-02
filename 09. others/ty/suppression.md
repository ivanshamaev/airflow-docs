# Подавление

Правила также можно отключать в конкретных местах кода (вместо полного отключения правила), чтобы скрывать ложные срабатывания или допустимые нарушения.

Note

Чтобы отключить правило полностью, установите для него уровень `ignore`, как описано в разделе [уровни правил](https://docs.astral.sh/ty/rules/#rule-levels).

## Комментарии подавления ty

Чтобы подавить нарушение правила в строке, добавьте в её конец комментарий `# ty: ignore[]`:

```python
a = 10 + "test"  # ty: ignore[unsupported-operator]
```

Нарушения, занимающие несколько строк, можно подавить, добавив комментарий в конец первой или последней строки нарушения:

```python
def sum_three_numbers(a: int, b: int, c: int) -> int: ...

# в первой строке

sum_three_numbers(  # ty: ignore[missing-argument]
    3,
    2
)

# или в последней строке

sum_three_numbers(
    3,
    2
)  # ty: ignore[missing-argument]
```

Чтобы подавить несколько нарушений в одной строке, перечислите правила через запятую:

```python
sum_three_numbers("one", 5)  # ty: ignore[missing-argument, invalid-argument-type]
```

Note

Указание имён правил (например, `[rule1, rule2]`) необязательно. Тем не менее мы настоятельно рекомендуем указывать конкретные правила, чтобы случайно не подавить другие ошибки.

## Стандартные комментарии подавления

ty поддерживает стандартный формат комментария [type: ignore](https://typing.python.org/en/latest/spec/directives.html#type-ignore-comments), введённый в PEP 484.

ty обрабатывает их аналогично комментариям `ty: ignore`, но подавляет все нарушения в этой строке, даже при использовании `type: ignore[code]`.

```python
# Игнорировать все ошибки типизации в следующей строке
sum_three_numbers("one", 5)  # type: ignore
```

## Несколько комментариев подавления

Чтобы подавить ошибку типизации в строке, где уже есть комментарий подавления от другого инструмента, добавьте комментарий `# ty: ignore` в ту же строку.

Например, чтобы подавить ошибку типов и отключить форматирование для конкретной строки:

```python
result = calculate()  # ty: ignore[invalid-argument-type]  # fmt: skip

# или
result = calculate()  # fmt: off  # ty: ignore[invalid-argument-type]
```

## Неиспользуемые комментарии подавления

Если включено правило [unused-ignore-comment](https://docs.astral.sh/ty/reference/rules/#unused-ignore-comment), ty будет сообщать о неиспользуемых комментариях `ty: ignore` и `type: ignore`.

Нарушения `unused-ignore-comment` можно подавить только с помощью `# ty: ignore[unused-ignore-comment]`. Их нельзя подавить с помощью `# ty: ignore` без кода правила или `# type: ignore`.

## Директива @no_type_check

ty поддерживает декоратор [@no_type_check](https://typing.python.org/en/latest/spec/directives.html#no-type-check) для подавления всех нарушений внутри функции.

```python
from typing import no_type_check

def sum_three_numbers(a: int, b: int, c: int) -> int:
    return a + b + c

@no_type_check
def main():
    sum_three_numbers(1, 2)  # ошибки из-за отсутствующего аргумента не будет
```

Использование декоратора `@no_type_check` для класса не поддерживается.
