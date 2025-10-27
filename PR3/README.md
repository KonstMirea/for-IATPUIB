# Основы обработки данных с помощью R и Dplyr
KTMUSIC22682@yandex.ru

## Цель работы

1.  Развить практические навыки использования языка программирования R
    для обработки данных
2.  Закрепить знания базовых типов данных языка R
3.  Развить практические навыки использования функций обработки данных
    пакета dplyr – функции select(), filter(), mutate(), arrange(),
    group_by()

## Исходные данные

1.  Программное обеспечение ОС Windows 11 Pro
2.  RStudio
3.  Интерпретатор языка R 4.5.1

## План

1.  Установть пакет nycflights13.

2.  Проанализировать встроенный в пакет dplyr набор данных starwars с
    помощью языка R и ответить на вопросы:

    1.  Сколько встроенных в пакет nycflights13 датафреймов?

    2.  Сколько строк в каждом датафрейме?

    3.  Сколько столбцов в каждом датафрейме?

    4.  Как просмотреть примерный вид датафрейма?

    5.  Сколько компаний-перевозчиков (carrier) учитывают эти наборы
        данных (представлено в наборах дан- ных)?

    6.  Сколько рейсов принял аэропорт John F Kennedy Intl в мае?

    7.  Какой самый северный аэропорт?

    8.  Какой аэропорт самый высокогорный (находится выше всех над
        уровнем моря)?

    9.  Какие бортовые номера у самых старых самолетов?

    10. Какая средняя температура воздуха была в сентябре в аэропорту
        John F Kennedy Intl (в градусах Цельсия).

    11. Самолеты какой авиакомпании совершили больше всего вылетов в
        июне? Данные i2z1.ddslab.ru 3

    12. Самолеты какой авиакомпании задерживались чаще других в 2013
        году?

### Установка nycflights13

> install.packages(‘nycflights13’) trying URL
> ‘https://mirror.truenetwork.ru/CRAN/bin/macosx/big-sur-arm64/contrib/4.5/nycflights13_1.0.2.tgz’
> Content type ‘application/octet-stream’ length 4504016 bytes (4.3 MB)
> ================================================== downloaded 4.3 MB

The downloaded binary packages are in
/var/folders/ns/z016wxgs06n20ddlstgmcgmw0000gn/T//RtmpwexTuw/downloaded_packages

### Сколько встроенных в пакет nycflights13 датафреймов?

> length(ls(“package:nycflights13”)) \[1\] 5

### Сколько строк в каждом датафрейме?

> sapply(ls(“package:nycflights13”), function(df) nrow(get(df,
> “package:nycflights13”))) airlines airports flights planes weather 16
> 1458 336776 3322 26115

### Сколько столбцов в каждом датафрейме?

> sapply(ls(“package:nycflights13”), function(df) ncol(get(df,
> “package:nycflights13”))) airlines airports flights planes weather 2 8
> 19 9 15

### Как просмотреть примерный вид датафрейма?

> glimpse(airlines) Rows: 16 Columns: 2 $ carrier <chr> “9E”, “AA”,
> “AS”, “B6”, “DL”, “EV”, “F9”, “FL”, “HA”, “MQ”, “OO”, “UA”, “US”,
> “VX”, “WN”, “YV” $ name <chr> “Endeavor Air Inc.”, “American Airlines
> Inc.”, “Alaska Airlines Inc.”, “JetBlue Airways”, “Delta Air Lines
> Inc.”, “Exp…

### Сколько компаний-перевозчиков (carrier) учитывают эти наборы данных (представлено в наборах данных)?

> length(unique(nycflights13::flights$carrier)) \[1\] 16

### Сколько рейсов принял аэропорт John F Kennedy Intl в мае?

> library(dplyr) nycflights13::flights %\>% + filter(dest == “JFK”,
> month == 5) %\>% + summarise(flights_count = n()) \# A tibble: 1 × 1
> flights_count <int> 1 0

### Какой самый северный аэропорт?

> library(dplyr) nycflights13::airports %\>% + filter(lat == max(lat,
> na.rm = TRUE)) \# A tibble: 1 × 8 faa name lat lon alt tz dst tzone
> <chr> <chr> <dbl> <dbl> <dbl> <dbl> <chr> <chr> 1 EEN Dillant Hopkins
> Airport 72.3 42.9 149 -5 A NA
