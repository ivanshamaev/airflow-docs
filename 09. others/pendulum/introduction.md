# Документация Pendulum

###### Версия 3.0

# Документация

# Установка

Установить `pendulum` очень просто:

```bash
$ pip install pendulum
```

или, если вы используете [poetry](https://python-poetry.org/):

```bash
$ poetry add pendulum
```

## Дополнительные возможности

Pendulum предоставляет опциональные возможности, которые нужно явно подключать при установке.

Доступные опции:

- `test`: набор вспомогательных средств для тестирования, позволяющих управлять течением времени.

Установить их можно, указав при установке Pendulum:

```bash
$ pip install pendulum[test]
```

или, если вы используете [poetry](https://python-poetry.org/):

```bash
$ poetry add pendulum[test]
```

# Введение

Pendulum — это Python-пакет для удобной работы с датой и временем.

Он предоставляет классы, которые являются прямой заменой встроенных (и наследуются от них).

Особое внимание уделено корректной работе с часовыми поясами на основе реализации `tzinfo`. Например, все сравнения выполняются в UTC или в часовом поясе используемой даты и времени.

```python
>>> import pendulum

>>> dt_toronto = pendulum.datetime(2012, 1, 1, tz='America/Toronto')
>>> dt_vancouver = pendulum.datetime(2012, 1, 1, tz='America/Vancouver')

>>> print(dt_vancouver.diff(dt_toronto).in_hours())
3
```

Часовой пояс по умолчанию (кроме метода `now()`) всегда UTC.

# Создание экземпляров

Есть несколько способов создать новый экземпляр `DateTime`.

Основной — вспомогательная функция `datetime()`.

```python
>>> import pendulum

>>> dt = pendulum.datetime(2015, 2, 5)
>>> isinstance(dt, datetime)
True
>>> dt.timezone.name
'UTC'
```

`datetime()` устанавливает время в `00:00:00`, если оно не задано, и часовой пояс (аргумент `tz`) в UTC. Иначе можно передать экземпляр `Timezone` или строку с названием часового пояса.

```python
>>> import pendulum

>>> pendulum.datetime(2015, 2, 5, tz='Europe/Paris')
>>> tz = pendulum.timezone('Europe/Paris')
>>> pendulum.datetime(2015, 2, 5, tz=tz)
```

Note

Поддерживаются строки часовых поясов из [базы данных IANA time zone](https://www.iana.org/time-zones).

Также поддерживается специальная строка `local` — она соответствует вашей текущей временной зоне.

Warning

В отличие от версии 1.x, аргумент `tz` передаётся только по имени.

Функция `local()` похожа на `datetime()`, но автоматически подставляет локальный часовой пояс.

```python
>>> import pendulum

>>> dt = pendulum.local(2015, 2, 5)
>>> print(dt.timezone.name)
'America/Toronto'
```

Note

`local()` — это псевдоним для `datetime(..., tz='local')`.

Есть также метод `now()`.

```python
>>> import pendulum

>>> now = pendulum.now()

>>> now_in_london_tz = pendulum.now('Europe/London')
>>> now_in_london_tz.timezone_name
'Europe/London'
```

Наряду с `now()` есть статические функции для создания известных моментов времени. Важно: `today()`, `tomorrow()` и `yesterday()` принимают параметр часового пояса, а время у них устанавливается в `00:00:00`.

```python
>>> now = pendulum.now()
>>> print(now)
'2016-06-28T16:51:45.978473-05:00'

>>> today = pendulum.today()
>>> print(today)
'2016-06-28T00:00:00-05:00'

>>> tomorrow = pendulum.tomorrow('Europe/London')
>>> print(tomorrow)
'2016-06-29T00:00:00+01:00'

>>> yesterday = pendulum.yesterday()
>>> print(yesterday)
'2016-06-27T00:00:00-05:00'
```

Pendulum использует даты и время с учётом часового пояса; это рекомендуемый способ работы с библиотекой. Если нужен «наивный» объект `DateTime`, используйте функцию `naive()`.

```python
>>> import pendulum

>>> naive = pendulum.naive(2015, 2, 5)
>>> naive.timezone
None
```

Функция `from_format()` аналогична встроенной `datetime.strptime()`, но использует собственные токены для создания экземпляра `DateTime`.

```python
>>> dt = pendulum.from_format('1975-05-21 22', 'YYYY-MM-DD HH')
>>> print(dt)
'1975-05-21T22:00:00+00:00'
```

Note

Все доступные токены описаны в разделе Formatter.

Можно передать аргумент `tz` для указания часового пояса:

```python
>>> dt = pendulum.from_format('1975-05-21 22', 'YYYY-MM-DD HH', tz='Europe/London')
'1975-05-21T22:00:00+01:00'
```

Для работы с Unix-временными метками используется `from_timestamp()`: создаётся экземпляр `DateTime` по заданной метке, при необходимости задаётся часовой пояс (по умолчанию UTC).

```python
>>> dt = pendulum.from_timestamp(-1)
>>> print(dt)
'1969-12-31T23:59:59+00:00'

>>> dt  = pendulum.from_timestamp(-1, tz='Europe/London')
>>> print(dt)
'1970-01-01T00:59:59+01:00'
```

Если у вас уже есть экземпляр `datetime.datetime`, из него можно создать `DateTime` с помощью функции `instance()`.

```python
>>> dt = datetime(2008, 1, 1)
>>> p = pendulum.instance(dt)
>>> print(p)
'2008-01-01T00:00:00+00:00'
```

# Разбор строк

Библиотека поддерживает формат RFC 3339, большинство форматов ISO 8601 и ряд других распространённых форматов.

```python
>>> import pendulum

>>> dt = pendulum.parse('1975-05-21T22:00:00')
>>> print(dt)
'1975-05-21T22:00:00+00:00'

# Часовой пояс можно задать аргументом tz
>>> dt = pendulum.parse('1975-05-21T22:00:00', tz='Europe/Paris')
>>> print(dt)
'1975-05-21T22:00:00+01:00'

# Не по ISO 8601, но часто встречается
>>> dt = pendulum.parse('1975-05-21 22:00:00')
```

При передаче нестандартной или сложной строки будет выброшено исключение; в таких случаях лучше использовать `from_format()`.

Если нужно использовать парсер [dateutil](https://dateutil.readthedocs.io/), передайте `strict=False`.

```python
>>> import pendulum

>>> dt = pendulum.parse('31-01-01')
Traceback (most recent call last):
...
ParserError: Unable to parse string [31-01-01]

>>> dt = pendulum.parse('31-01-01', strict=False)
>>> print(dt)
'2031-01-01T00:00:00+00:00'
```

## RFC 3339

| Строка | Результат |
| --- | --- |
| 1996-12-19T16:39:57-08:00 | 1996-12-19T16:39:57-08:00 |
| 1990-12-31T23:59:59Z | 1990-12-31T23:59:59+00:00 |

## ISO 8601

### Дата и время

| Строка | Результат |
| --- | --- |
| 20161001T143028+0530 | 2016-10-01T14:30:28+05:30 |
| 20161001T14 | 2016-10-01T14:00:00+00:00 |

| Строка | Результат |
| --- | --- |
| 2012 | 2012-01-01T00:00:00+00:00 |
| 2012-05-03 | 2012-05-03T00:00:00+00:00 |
| 20120503 | 2012-05-03T00:00:00+00:00 |
| 2012-05 | 2012-05-01T00:00:00+00:00 |

### Порядковый день года

| Строка | Результат |
| --- | --- |
| 2012-007 | 2012-01-07T00:00:00+00:00 |
| 2012007 | 2012-01-07T00:00:00+00:00 |

### Номер недели

| Строка | Результат |
| --- | --- |
| 2012-W05 | 2012-01-30T00:00:00+00:00 |
| 2012W05 | 2012-01-30T00:00:00+00:00 |
| 2012-W05-5 | 2012-02-03T00:00:00+00:00 |
| 2012W055 | 2012-02-03T00:00:00+00:00 |

### Время

При передаче только времени дата по умолчанию — сегодня.

| Строка | Результат |
| --- | --- |
| 00:00 | 2016-12-17T00:00:00+00:00 |
| 12:04:23 | 2016-12-17T12:04:23+00:00 |
| 120423 | 2016-12-17T12:04:23+00:00 |
| 12:04:23.45 | 2016-12-17T12:04:23.450000+00:00 |

### Интервалы

| Строка | Результат |
| --- | --- |
| 2007-03-01T13:00:00Z/2008-05-11T15:30:00Z | 2007-03-01T13:00:00+00:00 -> 2008-05-11T15:30:00+00:00 |
| 2008-05-11T15:30:00Z/P1Y2M10DT2H30M | 2008-05-11T15:30:00+00:00 -> 2009-07-21T18:00:00+00:00 |
| P1Y2M10DT2H30M/2008-05-11T15:30:00Z | 2007-03-01T13:00:00+00:00 -> 2008-05-11T15:30:00+00:00 |

Note

В `parse()` можно передать аргумент `exact=True`, чтобы получить тип, соответствующий строке:

```python
>>> import pendulum

>>> pendulum.parse('2012-05-03', exact=True)
Date(2012, 05, 03)

>>> pendulum.parse('12:04:23', exact=True)
Time(12, 04, 23)
```

# Локализация

Локализация применяется в методе `format()` через аргумент `locale`.

```python
>>> import pendulum

>>> dt = pendulum.datetime(1975, 5, 21)
>>> dt.format('dddd DD MMMM YYYY', locale='de')
'Mittwoch 21 Mai 1975'

>>> dt.format('dddd DD MMMM YYYY')
'Wednesday 21 May 1975'
```

Метод `diff_for_humans()` тоже локализуется; глобальную локаль можно задать через `pendulum.set_locale()`.

```python
>>> import pendulum

>>> pendulum.set_locale('de')
>>> pendulum.now().add(years=1).diff_for_humans()
'in 1 Jahr'
>>> pendulum.set_locale('en')
```

Локаль для одного вызова можно задать аргументом `locale` в `diff_for_humans()`.

```python
>>> pendulum.set_locale('de')
>>> dt = pendulum.now().add(years=1)
>>> dt.diff_for_humans(locale='fr')
'dans 1 an'
```

# Атрибуты и свойства

У Pendulum больше атрибутов и свойств, чем у стандартного класса `datetime`.

```python
>>> import pendulum

>>> dt = pendulum.parse('2012-09-05T23:26:11.123789')

# Эти свойства возвращают целые числа
>>> dt.year
2012
>>> dt.month
9
>>> dt.day
5
>>> dt.hour
23
>>> dt.minute
26
>>> dt.second
11
>>> dt.microsecond
123789
>>> dt.day_of_week
3
>>> dt.day_of_year
248
>>> dt.week_of_month
1
>>> dt.week_of_year
36
>>> dt.days_in_month
30
>>> dt.timestamp()
1346887571.123789
>>> dt.float_timestamp
1346887571.123789
>>> dt.int_timestamp
1346887571

>>> pendulum.datetime(1975, 5, 21).age
41  # вычислено относительно now() в том же поясе
>>> dt.quarter
3

# Разница с UTC в секундах (int), со знаком
>>> pendulum.from_timestamp(0).offset
0
>>> pendulum.from_timestamp(0, 'America/Toronto').offset
-18000

# Разница с UTC в часах (float), со знаком
>>> pendulum.from_timestamp(0, 'America/Toronto').offset_hours
-5.0
>>> pendulum.from_timestamp(0, 'Australia/Adelaide').offset_hours
9.5

# Экземпляр часового пояса
>>> pendulum.now().timezone
>>> pendulum.now().tz

# Название часового пояса
>>> pendulum.now().timezone_name

# Действует ли летнее время
>>> dt = pendulum.datetime(2012, 1, 1, tz='America/Toronto')
>>> dt.is_dst()
False
>>> dt = pendulum.datetime(2012, 9, 1, tz='America/Toronto')
>>> dt.is_dst()
True

# Совпадает ли часовой пояс с локальным
>>> pendulum.now().is_local()
True
>>> pendulum.now('Europe/London').is_local()
False

# Находится ли экземпляр в UTC
>>> pendulum.now().is_utc()
False
>>> pendulum.now('Europe/London').is_local()
False
>>> pendulum.now('UTC').is_utc()
True
```

# Fluent-помощники

Pendulum предоставляет методы, возвращающие новый экземпляр с изменёнными атрибутами. Ни один из них (кроме явной установки часового пояса) не меняет часовой пояс исходного экземпляра. В частности, установка timestamp не переводит часовой пояс в UTC.

```python
>>> import pendulum

>>> dt = pendulum.now()

>>> dt.set(year=1975, month=5, day=21).to_datetime_string()
'1975-05-21 13:45:18'

>>> dt.set(hour=22, minute=32, second=5).to_datetime_string()
'2016-11-16 22:32:05'
```

Дату и время можно менять методами `on()` и `at()`:

```python
>>> dt.on(1975, 5, 21).at(22, 32, 5).to_datetime_string()
'1975-05-21 22:32:05'

>>> dt.at(10).to_datetime_string()
'2016-11-16 10:00:00'

>>> dt.at(10, 30).to_datetime_string()
'2016-11-16 10:30:00'
```

Часовой пояс тоже можно изменить:

```python
>>> dt.set(tz='Europe/London')
```

`set(tz=...)` меняет только информацию о поясе без пересчёта времени, а `in_timezone()` (или `in_tz()`) переводит время в указанный пояс.

```python
>>> import pendulum

>>> dt = pendulum.datetime(2013, 3, 31, 2, 30)
>>> print(dt)
'2013-03-31T02:30:00+00:00'

>>> dt = dt.set(tz='Europe/Paris')
>>> print(dt)
'2013-03-31T03:30:00+02:00'

>>> dt = dt.in_tz('Europe/Paris')
>>> print(dt)
'2013-03-31T04:30:00+02:00'

>>> dt = dt.set(tz='Europe/Paris').set(tz='UTC')
>>> print(dt)
'2013-03-31T03:30:00+00:00'

>>> dt = dt.in_tz('Europe/Paris').in_tz('UTC')
>>> print(dt)
'2013-03-31T02:30:00+00:00'
```

# Форматирование в строку

Метод `__str__` определён так, что экземпляры `DateTime` выводятся в читаемом виде в строковом контексте. По умолчанию используется тот же формат, что и у `isoformat()`.

```python
>>> import pendulum

>>> dt = pendulum.datetime(1975, 12, 25, 14, 15, 16)
>>> print(dt)
'1975-12-25T14:15:16+00:00'

>>> dt.to_date_string()
'1975-12-25'

>>> dt.to_formatted_date_string()
'Dec 25, 1975'

>>> dt.to_time_string()
'14:15:16'

>>> dt.to_datetime_string()
'1975-12-25 14:15:16'

>>> dt.to_day_datetime_string()
'Thu, Dec 25, 1975 2:15 PM'

# Можно использовать метод format()
>>> dt.format('dddd Do [of] MMMM YYYY HH:mm:ss A')
'Thursday 25th of December 1975 02:15:16 PM'

# Метод strftime по-прежнему доступен
>>> dt.strftime('%A %-d%t of %B %Y %I:%M:%S %p')
'Thursday 25th of December 1975 02:15:16 PM'
```

Note

Локализация описана в разделе «Локализация».

## Распространённые форматы

Методы вывода в стандартных форматах:

```python
>>> import pendulum

>>> dt = pendulum.now()

>>> dt.to_atom_string()
>>> dt.to_cookie_string()
>>> dt.to_iso8601_string()
>>> dt.to_rfc822_string()
>>> dt.to_rfc850_string()
>>> dt.to_rfc1036_string()
>>> dt.to_rfc1123_string()
>>> dt.to_rfc2822_string()
>>> dt.to_rfc3339_string()
>>> dt.to_rss_string()
>>> dt.to_w3c_string()
```

## Formatter

При использовании метода `format()` Pendulum применяет собственный формат. Он интуитивнее, чем у `strftime()`, и поддерживает больше директив.

```python
>>> import pendulum

>>> dt = pendulum.datetime(1975, 12, 25, 14, 15, 16)
>>> dt.format('YYYY-MM-DD HH:mm:ss')
'1975-12-25 14:15:16'
```

### Токены

Поддерживаемые токены:

| Категория | Токен | Пример вывода |
| --- | --- | --- |
| Год | YYYY | 2000, 2001 ... 2012, 2013 |
| | YY | 00, 01 ... 12, 13 |
| | Y | 2000, 2001 ... 2012, 2013 |
| Квартал | Q | 1 2 3 4 |
| | Qo | 1st 2nd 3rd 4th |
| Месяц | MMMM | January, February ... |
| | MMM | Jan, Feb, Mar ... |
| | MM | 01, 02 ... 11, 12 |
| | M | 1, 2 ... 11, 12 |
| | Mo | 1st 2nd ... 11th 12th |
| День года | DDDD | 001, 002 ... 364, 365 |
| | DDD | 1, 2, 3 ... 364, 365 |
| День месяца | DD | 01, 02 ... 30, 31 |
| | D | 1, 2 ... 30, 31 |
| | Do | 1st, 2nd ... 30th, 31st |
| День недели | dddd | Monday, Tuesday ... |
| | ddd | Mon, Tue, Wed ... |
| | dd | Mo, Tu, We ... |
| | d | 0, 1, 2 ... 6 |
| Дни недели ISO | E | 1, 2 ... 7 |
| Час | HH | 00, 01 ... 23 |
| | H | 0, 1 ... 23 |
| | hh | 01, 02 ... 11, 12 |
| | h | 1, 2 ... 11, 12 |
| Минута | mm | 00, 01 ... 58, 59 |
| | m | 0, 1 ... 58, 59 |
| Секунда | ss | 00, 01 ... 58, 59 |
| | s | 0, 1 ... 58, 59 |
| Доли секунды | S, SS, SSS, SSSS... | 0, 00, 000, 0000... |
| AM/PM | A | AM, PM |
| Часовой пояс | Z | -07:00 ... +07:00 |
| | ZZ | -0700 ... +0700 |
| | z | Asia/Baku, Europe/Warsaw ... |
| | zz | EST CST ... MST PST |
| Метка времени | X | секунды | x | миллисекунды |

### Локализованные форматы

| Токен | Пример |
| --- | --- |
| LT | 8:30 PM |
| LTS | 8:30:25 PM |
| L | 09/04/1986 |
| LL | September 4 1986 |
| LLL | September 4 1986 8:30 PM |
| LLLL | Thursday, September 4 1986 8:30 PM |

### Экранирование

Символы в строках формата экранируются квадратными скобками.

```python
>>> dt.format('[today] dddd')
'today Sunday'
```

# Сравнение

Сравнение выполняется обычными операторами. Сравнение идёт в UTC, поэтому результат не всегда очевиден.

```python
>>> first = pendulum.datetime(2012, 9, 5, 23, 26, 11, 0, tz='America/Toronto')
>>> second = pendulum.datetime(2012, 9, 5, 20, 26, 11, 0, tz='America/Vancouver')

>>> first == second
True
>>> first != second
False
>>> first > second
False
>>> first >= second
True
```

Есть вспомогательные методы для типичных случаев (например, сравнение с `now()`: `is_today()`, `is_past()`, `is_leap_year()`, `is_birthday()` и т.д.). Для методов, сравнивающих с `now()`, момент `now()` создаётся в том же часовом поясе, что и экземпляр.

# Сложение и вычитание

Используйте методы `add()` и `subtract()`; каждый возвращает новый экземпляр `DateTime`.

```python
>>> dt = pendulum.datetime(2012, 1, 31)
>>> dt = dt.add(years=5)
>>> dt = dt.add(months=1)
>>> dt = dt.subtract(days=29)
>>> dt = dt.add(weeks=3, hours=24, minutes=61, seconds=61)
>>> dt = dt.add(years=3, months=2, days=6, hours=12, minutes=31, seconds=43)
```

Note

В `add()` можно передавать отрицательные значения — это эквивалентно `subtract()`.

# Разница (Difference)

Метод `diff()` возвращает экземпляр Period — длительность между двумя `DateTime`. Эту длительность можно выразить в разных единицах. Методы интервала возвращают полную разницу в запрошенных единицах; значения обрезаются, без округления.

Первый параметр `diff()` — экземпляр `DateTime` для сравнения или `None` (сравнение с `now()`). Второй параметр — возвращать ли абсолютное значение или относительное (со знаком минус, если переданная дата меньше текущего экземпляра). По умолчанию `True` — абсолютное значение.

```python
>>> dt_ottawa = pendulum.datetime(2000, 1, 1, tz='America/Toronto')
>>> dt_vancouver = pendulum.datetime(2000, 1, 1, tz='America/Vancouver')

>>> dt_ottawa.diff(dt_vancouver).in_hours()
3
>>> dt_vancouver.diff(dt_ottawa, False).in_hours()
-3
```

## Разница «по-человечески» (diff_for_humans)

`diff_for_humans()` добавляет фразу после значения разницы. Варианты: «5 months ago», «1 hour from now», «5 months before», «1 hour after». Второй параметр `True` убирает модификаторы «ago», «from now» и т.п.

```python
>>> pendulum.now().subtract(days=1).diff_for_humans()
'1 day ago'
>>> pendulum.now().add(seconds=5).diff_for_humans()
'5 seconds from now'
>>> pendulum.now().subtract(days=24).diff_for_humans(absolute=True)
'3 weeks'
```

Локаль задаётся глобально через `pendulum.set_locale('fr')` или аргументом `locale` в вызове.

# Модификаторы

Методы `start_of()`, `next()` и `previous()` устанавливают время в `00:00:00`, `end_of()` — в `23:59:59.999999`. Метод `average()` возвращает среднюю дату между двумя экземплярами.

```python
>>> dt.start_of('day')
>>> dt.end_of('day')
>>> dt.start_of('month'), dt.end_of('month')
>>> dt.start_of('year'), dt.end_of('year')
>>> dt.start_of('week'), dt.end_of('week')  # ISO8601: понедельник — воскресенье
>>> dt.next(pendulum.WEDNESDAY)
>>> dt.previous(pendulum.WEDNESDAY)
>>> start.average(end)
```

Поддерживаются единицы: day, month, quarter, year, decade, century, week. Аналогично работают `first_of()`, `last_of()`, `nth_of()`.

# Часовые пояса

Часовые пояса — важная часть библиотеки; Pendulum стремится обрабатывать их корректно.

Note

Система поясов лучше всего работает внутри экосистемы Pendulum, но может использоваться и со стандартным `datetime` с ограничениями (см. «Using the timezone library directly»).

## Нормализация

При создании `DateTime` библиотека нормализует момент для заданного пояса (переход на летнее/зимнее время).

```python
>>> pendulum.datetime(2013, 3, 31, 2, 30, tz='Europe/Paris')
# 2:30 31 марта 2013 не существует — возвращается 3:30+02:00
'2013-03-31T03:30:00+02:00'

>>> pendulum.datetime(2013, 10, 27, 2, 30, tz='Europe/Paris')
# 2:30 встречается дважды — считается, что переход уже произошёл
'2013-10-27T02:30:00+01:00'
```

Поведение при нормализации задаётся параметром `dst_rule`: `pendulum.PRE_TRANSITION`, `pendulum.TRANSITION_ERROR` и т.д.

## Сдвиг к моменту перехода

При добавлении времени к `DateTime` в момент перехода Pendulum учитывает контекст и применяет переход корректно (например, 01:59:59.999999 + 1 микросекунда → 03:00:00+02:00).

## Смена часового пояса

Используйте `in_timezone()` или кратко `in_tz()`.

```python
>>> in_paris = pendulum.datetime(2016, 8, 7, 22, 24, 30, tz='Europe/Paris')
>>> in_paris.in_timezone('America/New_York')
>>> in_paris.in_tz('Asia/Tokyo')
```

## Использование библиотеки часовых поясов напрямую

Warning

В Python < 3.6 использовать библиотеку поясов не рекомендуется: Pendulum опирается на атрибут `fold`, появившийся в 3.6.

С стандартным `datetime` можно использовать `pendulum.timezone`, но с ограничениями при сложении/вычитании времени вокруг переходов. По умолчанию используется атрибут `fold`; можно передать `dst_rule` в `convert()`. После арифметики с `timedelta` рекомендуется снова вызвать `tz.convert(dt)` для корректного результата.

# Duration

Класс `Duration` наследуется от `timedelta` и расширяет его.

Note

Нормализация ведёт себя иначе, чем у нативного `timedelta`, чтобы поведение было более предсказуемым.

Создание: `pendulum.duration(days=1177, seconds=7284, microseconds=1234)`. Поддерживаются также `years` и `months` (с приближением для нативной совместимости). Свойства: `years`, `months`, `weeks`, `days`, `remaining_days`, `hours`, `minutes`, `seconds`, `remaining_seconds`, `microseconds`. Методы: `total_weeks()`, `total_days()`, `in_weeks()`, `in_days()` и т.д., а также `in_words()` для текстового представления.

# Interval

При вычитании двух `DateTime` или вызове `diff()` возвращается экземпляр `Interval` (наследуется от Duration, хранит ссылки на исходные экземпляры). Создание: `pendulum.interval(start, end)`; можно передать `absolute=True` для положительного интервала при обратном порядке дат.

Метод `range('days')` (или `range('days', 2)` с шагом) позволяет итерировать по интервалу. Поддерживаются единицы: years, months, weeks, days, hours, minutes, seconds, microseconds. Проверка вхождения: `dt in interval`.

Warning

Большинство арифметических операций с Interval возвращают `Duration`, а не `Interval`.

# Тестирование

Pendulum предоставляет помощники для управления временем в тестах (доступны при установке с опцией `test`).

Warning

В Pendulum 3 удалены `set_test_now()` и `test()` из версии 2.

- **Относительное перемещение:** `pendulum.travel(minutes=5)` или `pendulum.travel(minutes=5, freeze=True)` — время останавливается.
- **Абсолютное перемещение:** `pendulum.travel_to(pendulum.yesterday())` или с `freeze=True`.
- **Возврат в настоящее:** `pendulum.travel_back()` или использование контекстного менеджера: `with pendulum.travel(minutes=5, freeze=True): ...`

# Ограничения

`DateTime` наследуется от `datetime`, но в редких случаях не может его полностью заменить.

- **sqlite3:** зарегистрировать адаптер: `register_adapter(pendulum.DateTime, lambda val: val.isoformat(' '))`.
- **mysqlclient / PyMySQL:** прописать конвертеры в `MySQLdb.converters.conversions` и `pymysql.converters.conversions`.
- **Django:** `isoformat()` всегда возвращает смещение (timezone-aware), что может вызвать ошибку для MySQL; можно создать свой `DateTimeField` или использовать обход для MySQLdb (возвращать `value.format('YYYY-MM-DD HH:mm:ss')` для `pendulum.DateTime`).
