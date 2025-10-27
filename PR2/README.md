# Основы обработки данных с помощью R и Dplyr
KTMUSIC22682@yandex.ru

## Цель работы

1.  Развить практические навыки использования языка программирования R
    для обработки данных
2.  Закрепить знания базовых типов данных языка R
3.  Развить практические навыки использования функций обработки данных
    пакета dplyr – функции select(), filter(), mutate(), arrange(),
    group_by()

### Установка dplyr

install.packages(“dplyr”) also installing the dependencies ‘utf8’,
‘pkgconfig’, ‘generics’, ‘pillar’, ‘tibble’, ‘tidyselect’ trying URL
‘https://mirror.truenetwork.ru/CRAN/bin/macosx/big-sur-arm64/contrib/4.5/utf8_1.2.6.tgz’
trying URL
‘https://mirror.truenetwork.ru/CRAN/bin/macosx/big-sur-arm64/contrib/4.5/pkgconfig_2.0.3.tgz’
trying URL
‘https://mirror.truenetwork.ru/CRAN/bin/macosx/big-sur-arm64/contrib/4.5/generics_0.1.4.tgz’
trying URL
‘https://mirror.truenetwork.ru/CRAN/bin/macosx/big-sur-arm64/contrib/4.5/pillar_1.11.1.tgz’
trying URL
‘https://mirror.truenetwork.ru/CRAN/bin/macosx/big-sur-arm64/contrib/4.5/tibble_3.3.0.tgz’
trying URL
‘https://mirror.truenetwork.ru/CRAN/bin/macosx/big-sur-arm64/contrib/4.5/tidyselect_1.2.1.tgz’
trying URL
‘https://mirror.truenetwork.ru/CRAN/bin/macosx/big-sur-arm64/contrib/4.5/dplyr_1.1.4.tgz’

The downloaded binary packages are in
/var/folders/ns/z016wxgs06n20ddlstgmcgmw0000gn/T//RtmpzMTynq/downloaded_packages
trying URL
‘https://mirror.truenetwork.ru/CRAN/bin/macosx/big-sur-arm64/contrib/4.5/dplyr_1.1.4.tgz’
Content type ‘application/octet-stream’ length 1614595 bytes (1.5 MB)
================================================== downloaded 1.5 MB

### Подключение dplyr

``` r
library(dplyr)
```


    Attaching package: 'dplyr'

    The following objects are masked from 'package:stats':

        filter, lag

    The following objects are masked from 'package:base':

        intersect, setdiff, setequal, union

### Сколько строк в датафрейме?

``` r
nrow(starwars)
```

    [1] 87

### Сколько столбцов в датафрейме?

``` r
ncol(starwars)
```

    [1] 14

### Как просмотреть примерный вид датафрейма?

``` r
glimpse(starwars)
```

    Rows: 87
    Columns: 14
    $ name       <chr> "Luke Skywalker", "C-3PO", "R2-D2", "Darth Vader", "Leia Or…
    $ height     <int> 172, 167, 96, 202, 150, 178, 165, 97, 183, 182, 188, 180, 2…
    $ mass       <dbl> 77.0, 75.0, 32.0, 136.0, 49.0, 120.0, 75.0, 32.0, 84.0, 77.…
    $ hair_color <chr> "blond", NA, NA, "none", "brown", "brown, grey", "brown", N…
    $ skin_color <chr> "fair", "gold", "white, blue", "white", "light", "light", "…
    $ eye_color  <chr> "blue", "yellow", "red", "yellow", "brown", "blue", "blue",…
    $ birth_year <dbl> 19.0, 112.0, 33.0, 41.9, 19.0, 52.0, 47.0, NA, 24.0, 57.0, …
    $ sex        <chr> "male", "none", "none", "male", "female", "male", "female",…
    $ gender     <chr> "masculine", "masculine", "masculine", "masculine", "femini…
    $ homeworld  <chr> "Tatooine", "Tatooine", "Naboo", "Tatooine", "Alderaan", "T…
    $ species    <chr> "Human", "Droid", "Droid", "Human", "Human", "Human", "Huma…
    $ films      <list> <"A New Hope", "The Empire Strikes Back", "Return of the J…
    $ vehicles   <list> <"Snowspeeder", "Imperial Speeder Bike">, <>, <>, <>, "Imp…
    $ starships  <list> <"X-wing", "Imperial shuttle">, <>, <>, "TIE Advanced x1",…

### Сколько уникальных рас персонажей (species) представлено в данных?

``` r
starwars %>% summarise(количество_уникальных_рас = n_distinct(species))
```

    # A tibble: 1 × 1
      количество_уникальных_рас
                          <int>
    1                        38

### Найти самого высокого персонажа.

``` r
starwars %>%
      filter(height == max(height, na.rm = TRUE))
```

    # A tibble: 1 × 14
      name      height  mass hair_color skin_color eye_color birth_year sex   gender
      <chr>      <int> <dbl> <chr>      <chr>      <chr>          <dbl> <chr> <chr> 
    1 Yarael P…    264    NA none       white      yellow            NA male  mascu…
    # ℹ 5 more variables: homeworld <chr>, species <chr>, films <list>,
    #   vehicles <list>, starships <list>

### Найти всех персонажей ниже 170

``` r
starwars %>% filter(height < 170)
```

    # A tibble: 22 × 14
       name     height  mass hair_color skin_color eye_color birth_year sex   gender
       <chr>     <int> <dbl> <chr>      <chr>      <chr>          <dbl> <chr> <chr> 
     1 C-3PO       167    75 <NA>       gold       yellow           112 none  mascu…
     2 R2-D2        96    32 <NA>       white, bl… red               33 none  mascu…
     3 Leia Or…    150    49 brown      light      brown             19 fema… femin…
     4 Beru Wh…    165    75 brown      light      blue              47 fema… femin…
     5 R5-D4        97    32 <NA>       white, red red               NA none  mascu…
     6 Yoda         66    17 white      green      brown            896 male  mascu…
     7 Mon Mot…    150    NA auburn     fair       blue              48 fema… femin…
     8 Wicket …     88    20 brown      brown      brown              8 male  mascu…
     9 Nien Nu…    160    68 none       grey       black             NA male  mascu…
    10 Watto       137    NA black      blue, grey yellow            NA male  mascu…
    # ℹ 12 more rows
    # ℹ 5 more variables: homeworld <chr>, species <chr>, films <list>,
    #   vehicles <list>, starships <list>

### Подсчитать ИМТ (индекс массы тела) для всех персонажей.

``` r
starwars %>%
     mutate(height_m = height / 100,bmi = mass / (height_m)^2) %>%
     select(name, mass, height, bmi)
```

    # A tibble: 87 × 4
       name                mass height   bmi
       <chr>              <dbl>  <int> <dbl>
     1 Luke Skywalker        77    172  26.0
     2 C-3PO                 75    167  26.9
     3 R2-D2                 32     96  34.7
     4 Darth Vader          136    202  33.3
     5 Leia Organa           49    150  21.8
     6 Owen Lars            120    178  37.9
     7 Beru Whitesun Lars    75    165  27.5
     8 R5-D4                 32     97  34.0
     9 Biggs Darklighter     84    183  25.1
    10 Obi-Wan Kenobi        77    182  23.2
    # ℹ 77 more rows

### Найти 10 самых “вытянутых” персонажей.“Вытянутость” оценить по отношению массы (mass) к росту (height) персонажей.

``` r
 starwars %>% 
     mutate(stretch_ratio = mass / height) %>% 
     arrange(desc(stretch_ratio)) %>% 
     slice_head(n = 10)
```

    # A tibble: 10 × 15
       name     height  mass hair_color skin_color eye_color birth_year sex   gender
       <chr>     <int> <dbl> <chr>      <chr>      <chr>          <dbl> <chr> <chr> 
     1 Jabba D…    175  1358 <NA>       green-tan… orange         600   herm… mascu…
     2 Grievous    216   159 none       brown, wh… green, y…       NA   male  mascu…
     3 IG-88       200   140 none       metal      red             15   none  mascu…
     4 Owen La…    178   120 brown, gr… light      blue            52   male  mascu…
     5 Darth V…    202   136 none       white      yellow          41.9 male  mascu…
     6 Jek Ton…    180   110 brown      fair       blue            NA   <NA>  <NA>  
     7 Bossk       190   113 none       green      red             53   male  mascu…
     8 Tarfful     234   136 brown      brown      blue            NA   male  mascu…
     9 Dexter …    198   102 none       brown      yellow          NA   male  mascu…
    10 Chewbac…    228   112 brown      unknown    blue           200   male  mascu…
    # ℹ 6 more variables: homeworld <chr>, species <chr>, films <list>,
    #   vehicles <list>, starships <list>, stretch_ratio <dbl>

### Найти средний возраст персонажей каждой расы вселенной Звездных войн.

``` r
starwars %>%
     mutate(current_year = 100,age = current_year + birth_year) %>%
     filter(!is.na(age) & !is.na(species)) %>%
     group_by(species) %>%
     summarise(average_age = mean(age),count = n()) %>%
     arrange(desc(average_age))
```

    # A tibble: 15 × 3
       species        average_age count
       <chr>                <dbl> <int>
     1 Yoda's species        996      1
     2 Hutt                  700      1
     3 Wookiee               300      1
     4 Cerean                192      1
     5 Zabrak                154      1
     6 Human                 154.    26
     7 Droid                 153.     3
     8 Trandoshan            153      1
     9 Gungan                152      1
    10 Mirialan              149      2
    11 Twi'lek               148      1
    12 Rodian                144      1
    13 Mon Calamari          141      1
    14 Kel Dor               122      1
    15 Ewok                  108      1

### Найти самый распространенный цвет глаз персонажей вселенной Звездных войн.

``` r
starwars %>% 
     filter(!is.na(eye_color)) %>% 
     count(eye_color, sort = TRUE) %>% 
     slice(1)
```

    # A tibble: 1 × 2
      eye_color     n
      <chr>     <int>
    1 brown        21

### Подсчитать среднюю длину имени в каждой расе вселенной Звездных войн.

``` r
starwars %>%
     filter(!is.na(name) & !is.na(species)) %>%
     group_by(species) %>%
     summarise(average_name_length = mean(nchar(name)))
```

    # A tibble: 37 × 2
       species   average_name_length
       <chr>                   <dbl>
     1 Aleena                  12   
     2 Besalisk                15   
     3 Cerean                  12   
     4 Chagrian                10   
     5 Clawdite                10   
     6 Droid                    4.83
     7 Dug                      7   
     8 Ewok                    21   
     9 Geonosian               17   
    10 Gungan                  11.7 
    # ℹ 27 more rows

## Оценка результатов

В данной практической работе мы применили знания полученные в результате
первой практической работы для обработки массива данных.

## Вывод

Таким образом, мы развили практические навыки использования функций
обработки данных пакета dplyr – функции select(), filter(), mutate(),
arrange(), group_by()
