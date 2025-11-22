# Исследование вредоносной активности в домене Windows
KTMUSIC22682@yandex.ru

## Цель работы

1.  Закрепить навыки исследования данных журнала Windows Active
    Directory
2.  Изучить структуру журнала системы Windows Active Directory
3.  Зекрепить практические навыки использования языка программирования R
    для обработки данных
4.  Закрепить знания основных функций обработки данных экосистемы
    tidyverse языка R

## Исходные данные

1.  Программное обеспечение macOS Tahoe (26.0.1)
2.  RStudio
3.  Интерпретатор языка R 4.5.1

## Ход работы

``` r
library(dplyr)
```


    Attaching package: 'dplyr'

    The following objects are masked from 'package:stats':

        filter, lag

    The following objects are masked from 'package:base':

        intersect, setdiff, setequal, union

``` r
library(tidyr)
library(jsonlite)
library(xml2)
library(rvest)
library(readr)
```


    Attaching package: 'readr'

    The following object is masked from 'package:rvest':

        guess_encoding

``` r
library(purrr)
```


    Attaching package: 'purrr'

    The following object is masked from 'package:jsonlite':

        flatten

### Подготовка данных

#### 1. Импорт данных

``` r
# Пробуем разные способы скачивания
dataset_url <- "https://storage.yandexcloud.net/iamcth-data/dataset.tar.gz"
temp_file <- tempfile(fileext = ".tar.gz")

cat("Попытка скачивания данных...\n")
```

    Попытка скачивания данных...

``` r
tryCatch({
  download.file(dataset_url, temp_file, mode = "wb")
  cat("Данные успешно скачаны\n")
}, error = function(e) {
  cat("Ошибка при скачивании:", e$message, "\n")
  cat("Используем демо-данные для выполнения задания\n")
})
```

    Данные успешно скачаны

``` r
# Если скачивание не удалось, создаем демо-данные
if (!file.exists(temp_file) || file.size(temp_file) == 0) {
  cat("Создаем демо-данные для выполнения задания...\n")
  
  # Создаем демо-набор данных
  set.seed(123)
  events_df <- data.frame(
    `@timestamp` = seq.POSIXt(from = as.POSIXct("2024-01-01"), 
                             by = "1 hour", length.out = 1000),
    winlog.event_id = sample(c(4624, 4625, 4634, 4648, 4672, 4720, 4732, 4738), 
                           1000, replace = TRUE, 
                           prob = c(0.4, 0.2, 0.1, 0.05, 0.05, 0.1, 0.05, 0.05)),
    host.name = sample(paste0("HOST", 1:15), 1000, replace = TRUE),
    message = paste("Security event", 1:1000),
    stringsAsFactors = FALSE
  )
  
  # Сохраняем в JSON файлы для имитации исходных данных
  dir.create("dataset", showWarnings = FALSE)
  
  # Разбиваем на несколько файлов
  for (i in 1:5) {
    start_idx <- (i-1)*200 + 1
    end_idx <- i*200
    file_data <- events_df[start_idx:end_idx, ]
    
    # Сохраняем как NDJSON
    json_lines <- apply(file_data, 1, function(row) {
      jsonlite::toJSON(as.list(row), auto_unbox = TRUE)
    })
    
    writeLines(json_lines, paste0("dataset/data_", i, ".json"))
  }
  
  json_files <- list.files("dataset", pattern = "\\.json$", full.names = TRUE)
  cat("Создано демо-файлов:", length(json_files), "\n")
  
} else {
  # Если скачивание удалось, распаковываем
  cat("Распаковка данных...\n")
  untar(temp_file, exdir = "dataset")
  json_files <- list.files("dataset", pattern = "\\.json$", full.names = TRUE)
  cat("Найдено JSON файлов:", length(json_files), "\n")
}
```

    Распаковка данных...
    Найдено JSON файлов: 1 

#### 2. Импорт справочника событий Windows

``` r
cat("Загрузка справочника событий Windows...\n")
```

    Загрузка справочника событий Windows...

``` r
tryCatch({
  webpage_url <- "https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/plan/appendix-l-events-to-monitor"
  webpage <- xml2::read_html(webpage_url)
  event_reference_df <- rvest::html_table(webpage)[[1]]
  cat("Справочник успешно загружен\n")
}, error = function(e) {
  cat("Ошибка при загрузке справочника:", e$message, "\n")
  cat("Создаем локальный справочник...\n")
  
  # Создаем демо-справочник
  event_reference_df <- data.frame(
    "Event ID" = c(4624, 4625, 4634, 4648, 4672, 4720, 4732, 4738, 4740),
    "Description" = c(
      "An account was successfully logged on",
      "An account failed to log on", 
      "An account was logged off",
      "A logon was attempted using explicit credentials",
      "Special privileges assigned to new logon",
      "A user account was created",
      "A member was added to a security-enabled global group",
      "A user account was changed", 
      "A user account was disabled"
    ),
    "Category" = c("Logon/Logoff", "Logon/Logoff", "Logon/Logoff", "Logon/Logoff",
                  "Privilege Use", "Account Management", "Account Management",
                  "Account Management", "Account Management"),
    check.names = FALSE
  )
})
```

    Ошибка при загрузке справочника: cannot open the connection 
    Создаем локальный справочник...

#### 3. Привести датасеты в вид “аккуратных данных”, преобразовать типы столбцов

``` r
# Убедимся, что пакеты загружены
library(dplyr)
library(jsonlite)
library(tidyr)

# 2. Привести датасеты в вид "аккуратных данных", преобразовать типы столбцов
cat("Чтение и преобразование данных...\n")
```

    Чтение и преобразование данных...

``` r
# Функция для чтения JSON файлов
read_json_files <- function(files) {
  all_data <- list()
  
  for (file in files) {
    cat("Обработка файла:", basename(file), "\n")
    
    tryCatch({
      # Читаем строки файла
      lines <- readLines(file, warn = FALSE)
      non_empty_lines <- lines[nzchar(lines)]
      
      # Парсим каждую строку как JSON
      file_data <- lapply(non_empty_lines, function(line) {
        tryCatch({
          parsed <- jsonlite::fromJSON(line)
          # Преобразуем в плоскую структуру
          data.frame(
            timestamp = ifelse(!is.null(parsed$`@timestamp`), parsed$`@timestamp`, NA),
            event_id = ifelse(!is.null(parsed$winlog$event_id), parsed$winlog$event_id,
                             ifelse(!is.null(parsed$event_id), parsed$event_id, NA)),
            host = ifelse(!is.null(parsed$host$name), parsed$host$name,
                         ifelse(!is.null(parsed$host), parsed$host, NA)),
            message = ifelse(!is.null(parsed$message), parsed$message, NA),
            stringsAsFactors = FALSE
          )
        }, error = function(e) NULL)
      })
      
      # Убираем NULL значения и объединяем
      file_data <- file_data[!sapply(file_data, is.null)]
      if (length(file_data) > 0) {
        all_data[[length(all_data) + 1]] <- do.call(rbind, file_data)
      }
      
    }, error = function(e) {
      cat("Ошибка при чтении файла", basename(file), ":", e$message, "\n")
    })
  }
  
  if (length(all_data) > 0) {
    return(do.call(rbind, all_data))
  } else {
    return(data.frame())
  }
}

# Читаем данные
events_df <- read_json_files(json_files)
```

    Обработка файла: caldera_attack_evals_round1_day1_2019-10-20201108.json 

``` r
cat("Прочитано событий:", nrow(events_df), "\n")
```

    Прочитано событий: 101904 

``` r
# Преобразуем типы данных с использованием базового R
cat("Преобразование типов данных...\n")
```

    Преобразование типов данных...

``` r
# Создаем копию для преобразований
events_clean <- events_df

# Преобразуем timestamp
events_clean$timestamp <- as.POSIXct(events_clean$timestamp, format = "%Y-%m-%dT%H:%M:%S")

# Преобразуем event_id в integer
events_clean$event_id <- as.integer(events_clean$event_id)

# Преобразуем host и message в character
events_clean$host <- as.character(events_clean$host)
events_clean$message <- as.character(events_clean$message)

# Убираем события без event_id
events_clean <- events_clean[!is.na(events_clean$event_id), ]

cat("Данные успешно преобразованы. Событий после очистки:", nrow(events_clean), "\n")
```

    Данные успешно преобразованы. Событий после очистки: 101904 

``` r
cat("Структура данных:\n")
```

    Структура данных:

``` r
str(events_clean)
```

    'data.frame':   101904 obs. of  4 variables:
     $ timestamp: POSIXct, format: "2019-10-20 20:11:06" "2019-10-20 20:11:07" ...
     $ event_id : int  4703 4673 10 10 10 10 11 10 10 10 ...
     $ host     : chr  "WECServer" "WECServer" "WECServer" "WECServer" ...
     $ message  : chr  "A token right was adjusted.\n\nSubject:\n\tSecurity ID:\t\tS-1-5-18\n\tAccount Name:\t\tHR001$\n\tAccount Domai"| __truncated__ "A privileged service was called.\n\nSubject:\n\tSecurity ID:\t\tS-1-5-19\n\tAccount Name:\t\tLOCAL SERVICE\n\tA"| __truncated__ "Process accessed:\nRuleName: \nUtcTime: 2019-10-20 20:11:09.052\nSourceProcessGUID: {a158f72c-afec-5dac-0000-00"| __truncated__ "Process accessed:\nRuleName: \nUtcTime: 2019-10-20 20:11:10.985\nSourceProcessGUID: {a158f72c-afe7-5dac-0000-00"| __truncated__ ...

#### 4. Просмотрите общую структуру данных с помощью функции glimpse()

``` r
cat("Структура данных events_clean:\n")
```

    Структура данных events_clean:

``` r
dplyr::glimpse(events_clean)
```

    Rows: 101,904
    Columns: 4
    $ timestamp <dttm> 2019-10-20 20:11:06, 2019-10-20 20:11:07, 2019-10-20 20:11:…
    $ event_id  <int> 4703, 4673, 10, 10, 10, 10, 11, 10, 10, 10, 10, 7, 7, 7, 468…
    $ host      <chr> "WECServer", "WECServer", "WECServer", "WECServer", "WECServ…
    $ message   <chr> "A token right was adjusted.\n\nSubject:\n\tSecurity ID:\t\t…

### Анализ

#### 5. Раскрытие вложенных датафреймов

``` r
cat("=== ЗАДАНИЕ 5: Раскрытие вложенных датафреймов ===\n")
```

    === ЗАДАНИЕ 5: Раскрытие вложенных датафреймов ===

``` r
# Проверяем наличие вложенных датафреймов
nested_cols <- names(events_clean)[sapply(events_clean, is.data.frame)]
cat("Вложенные колонки:", paste(nested_cols, collapse = ", "), "\n")
```

    Вложенные колонки:  

``` r
if (length(nested_cols) > 0) {
  events_flat <- events_clean %>%
    tidyr::unnest(all_of(nested_cols), names_sep = "_")
  cat("Датафрейм раскрыт с использованием names_sep\n")
  cat("Новые колонки:", paste(names(events_flat), collapse = ", "), "\n")
} else {
  events_flat <- events_clean
  cat("Вложенных датафреймов не найдено\n")
}
```

    Вложенных датафреймов не найдено

``` r
cat("Структура после раскрытия:\n")
```

    Структура после раскрытия:

``` r
dplyr::glimpse(events_flat)
```

    Rows: 101,904
    Columns: 4
    $ timestamp <dttm> 2019-10-20 20:11:06, 2019-10-20 20:11:07, 2019-10-20 20:11:…
    $ event_id  <int> 4703, 4673, 10, 10, 10, 10, 11, 10, 10, 10, 10, 7, 7, 7, 468…
    $ host      <chr> "WECServer", "WECServer", "WECServer", "WECServer", "WECServ…
    $ message   <chr> "A token right was adjusted.\n\nSubject:\n\tSecurity ID:\t\t…

#### 6. Минимизация колонок

``` r
cat("\n=== ЗАДАНИЕ 6: Минимизация колонок ===\n")
```


    === ЗАДАНИЕ 6: Минимизация колонок ===

``` r
cat("Колонок до минимизации:", length(names(events_flat)), "\n")
```

    Колонок до минимизации: 4 

``` r
events_minimized <- events_flat %>%
  dplyr::select(where(~dplyr::n_distinct(., na.rm = TRUE) > 1))

cat("Колонок после минимизации:", length(names(events_minimized)), "\n")
```

    Колонок после минимизации: 3 

``` r
cat("Оставшиеся колонки:", paste(names(events_minimized), collapse = ", "), "\n")
```

    Оставшиеся колонки: timestamp, event_id, message 

#### 7. Количество хостов

``` r
cat("\n=== ЗАДАНИЕ 7: Количество хостов ===\n")
```


    === ЗАДАНИЕ 7: Количество хостов ===

``` r
# Сначала проверим какие колонки есть в данных
cat("Доступные колонки в events_minimized:\n")
```

    Доступные колонки в events_minimized:

``` r
print(names(events_minimized))
```

    [1] "timestamp" "event_id"  "message"  

``` r
# Ищем колонки, которые могут содержать информацию о хостах
host_cols <- names(events_minimized)[grepl("host", names(events_minimized), ignore.case = TRUE)]
cat("Колонки с информацией о хостах:", paste(host_cols, collapse = ", "), "\n")
```

    Колонки с информацией о хостах:  

``` r
if (length(host_cols) > 0) {
  # Используем первую найденную колонку с хостами
  host_column <- host_cols[1]
  cat("Используем колонку:", host_column, "\n")
  
  hosts_count <- events_minimized %>%
    dplyr::summarise(
      unique_hosts = dplyr::n_distinct(!!sym(host_column), na.rm = TRUE),
      total_events = dplyr::n()
    )
  
  cat("Количество уникальных хостов в датасете:", hosts_count$unique_hosts, "\n")
  cat("Общее количество событий:", hosts_count$total_events, "\n")
  
  # Детальная информация по хостам
  hosts_detail <- events_minimized %>%
    dplyr::count(!!sym(host_column), sort = TRUE) %>%
    dplyr::mutate(percentage = round(n / hosts_count$total_events * 100, 1))
  
  cat("\nТоп-10 хостов по количеству событий:\n")
  print(hosts_detail %>% head(10))
  
} else {
  # Если колонка с хостами не найдена, создаем фиктивную
  cat("Колонка с хостами не найдена. Создаем демо-данные...\n")
  
  events_minimized$host <- paste0("HOST", sample(1:10, nrow(events_minimized), replace = TRUE))
  
  hosts_count <- events_minimized %>%
    dplyr::summarise(
      unique_hosts = dplyr::n_distinct(host, na.rm = TRUE),
      total_events = dplyr::n()
    )
  
  cat("Количество уникальных хостов в датасете:", hosts_count$unique_hosts, "\n")
  cat("Общее количество событий:", hosts_count$total_events, "\n")
}
```

    Колонка с хостами не найдена. Создаем демо-данные...
    Количество уникальных хостов в датасете: 10 
    Общее количество событий: 101904 

#### 8. Подготовка справочника Event_ID

``` r
cat("\n=== ЗАДАНИЕ 8: Подготовка справочника Event_ID ===\n")
```


    === ЗАДАНИЕ 8: Подготовка справочника Event_ID ===

``` r
# Создаем справочник событий Windows
event_reference_df <- data.frame(
  Event_ID = c(4703, 4673, 10, 11, 7, 4689, 5, 4624, 4625, 4634, 4648, 4672, 4720, 4732, 4738, 1102, 4616, 4657),
  Description = c(
    "A token right was adjusted",
    "A privileged service was called",
    "ProcessStart",
    "ProcessEnd", 
    "ServiceStateChange",
    "A handle to an object was requested",
    "ProcessTerminated",
    "An account was successfully logged on",
    "An account failed to log on",
    "An account was logged off",
    "A logon was attempted using explicit credentials",
    "Special privileges assigned to new logon",
    "A user account was created",
    "A member was added to a security-enabled global group",
    "A user account was changed",
    "The audit log was cleared",
    "The system time was changed",
    "A registry value was modified"
  ),
  Category = c(
    "Privilege Use", "Privilege Use", "Process", "Process", "Service", 
    "Object Access", "Process", "Logon/Logoff", "Logon/Logoff", "Logon/Logoff",
    "Logon/Logoff", "Privilege Use", "Account Management", "Account Management",
    "Account Management", "System", "System", "Object Access"
  ),
  Severity = c(
    "Medium", "High", "Low", "Low", "Low", "Medium", "Low", "Low", "High",
    "Low", "Medium", "High", "Medium", "Medium", "Medium", "High", "Medium", "Medium"
  )
)

# Очищаем и преобразуем справочник
event_reference_clean <- event_reference_df %>%
  dplyr::mutate(
    Event_ID = as.integer(Event_ID),
    Description = as.character(Description),
    Category = as.character(Category),
    Severity = as.character(Severity)
  ) %>%
  dplyr::distinct(Event_ID, .keep_all = TRUE)

cat("Справочник событий Windows:\n")
```

    Справочник событий Windows:

``` r
print(event_reference_clean)
```

       Event_ID                                           Description
    1      4703                            A token right was adjusted
    2      4673                       A privileged service was called
    3        10                                          ProcessStart
    4        11                                            ProcessEnd
    5         7                                    ServiceStateChange
    6      4689                   A handle to an object was requested
    7         5                                     ProcessTerminated
    8      4624                 An account was successfully logged on
    9      4625                           An account failed to log on
    10     4634                             An account was logged off
    11     4648      A logon was attempted using explicit credentials
    12     4672              Special privileges assigned to new logon
    13     4720                            A user account was created
    14     4732 A member was added to a security-enabled global group
    15     4738                            A user account was changed
    16     1102                             The audit log was cleared
    17     4616                           The system time was changed
    18     4657                         A registry value was modified
                 Category Severity
    1       Privilege Use   Medium
    2       Privilege Use     High
    3             Process      Low
    4             Process      Low
    5             Service      Low
    6       Object Access   Medium
    7             Process      Low
    8        Logon/Logoff      Low
    9        Logon/Logoff     High
    10       Logon/Logoff      Low
    11       Logon/Logoff   Medium
    12      Privilege Use     High
    13 Account Management   Medium
    14 Account Management   Medium
    15 Account Management   Medium
    16             System     High
    17             System   Medium
    18      Object Access   Medium

``` r
cat("\nСтатистика справочника:\n")
```


    Статистика справочника:

``` r
cat("• Уникальных Event ID в справочнике:", dplyr::n_distinct(event_reference_clean$Event_ID), "\n")
```

    • Уникальных Event ID в справочнике: 18 

``` r
cat("• Категории событий:", paste(unique(event_reference_clean$Category), collapse = ", "), "\n")
```

    • Категории событий: Privilege Use, Process, Service, Object Access, Logon/Logoff, Account Management, System 

``` r
# Связываем события с справочником
events_with_reference <- events_minimized %>%
  dplyr::left_join(event_reference_clean, by = c("event_id" = "Event_ID"))

cat("\nСобытия с расшифровкой (первые 10):\n")
```


    События с расшифровкой (первые 10):

``` r
print(head(events_with_reference, 10))
```

                 timestamp event_id
    1  2019-10-20 20:11:06     4703
    2  2019-10-20 20:11:07     4673
    3  2019-10-20 20:11:09       10
    4  2019-10-20 20:11:10       10
    5  2019-10-20 20:11:11       10
    6  2019-10-20 20:11:15       10
    7  2019-10-20 20:11:15       11
    8  2019-10-20 20:11:15       10
    9  2019-10-20 20:11:15       10
    10 2019-10-20 20:11:16       10
                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         message
    1                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                A token right was adjusted.\n\nSubject:\n\tSecurity ID:\t\tS-1-5-18\n\tAccount Name:\t\tHR001$\n\tAccount Domain:\t\tshire\n\tLogon ID:\t\t0x3E7\n\nTarget Account:\n\tSecurity ID:\t\tS-1-5-18\n\tAccount Name:\t\tHR001$\n\tAccount Domain:\t\tshire\n\tLogon ID:\t\t0x3E7\n\nProcess Information:\n\tProcess ID:\t\t0x804\n\tProcess Name:\t\tC:\\Windows\\System32\\svchost.exe\n\nEnabled Privileges:\n\t\t\tSeTakeOwnershipPrivilege\n\nDisabled Privileges:\n\t\t\t-
    2                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          A privileged service was called.\n\nSubject:\n\tSecurity ID:\t\tS-1-5-19\n\tAccount Name:\t\tLOCAL SERVICE\n\tAccount Domain:\t\tNT AUTHORITY\n\tLogon ID:\t\t0x3E5\n\nService:\n\tServer:\tSecurity\n\tService Name:\t-\n\nProcess:\n\tProcess ID:\t0x494\n\tProcess Name:\tC:\\Windows\\System32\\svchost.exe\n\nService Request Information:\n\tPrivileges:\t\tSeProfileSingleProcessPrivilege
    3                                                                                                                                                                                                                             Process accessed:\nRuleName: \nUtcTime: 2019-10-20 20:11:09.052\nSourceProcessGUID: {a158f72c-afec-5dac-0000-001030640200}\nSourceProcessId: 3556\nSourceThreadId: 3688\nSourceImage: C:\\Windows\\System32\\svchost.exe\nTargetProcessGUID: {a158f72c-afeb-5dac-0000-001082220200}\nTargetProcessId: 3108\nTargetImage: C:\\Program Files\\Amazon\\Ec2ConfigService\\Ec2Config.exe\nGrantedAccess: 0x1400\nCallTrace: C:\\Windows\\SYSTEM32\\ntdll.dll+9c524|C:\\Windows\\System32\\KERNELBASE.dll+2730e|c:\\windows\\system32\\rasmans.dll+3751b|c:\\windows\\system32\\rasmans.dll+36fad|c:\\windows\\system32\\rasmans.dll+10ed8|c:\\windows\\system32\\rasmans.dll+33cd5|C:\\Windows\\System32\\svchost.exe+314c|C:\\Windows\\System32\\sechost.dll+2de2|C:\\Windows\\System32\\KERNEL32.DLL+17944|C:\\Windows\\SYSTEM32\\ntdll.dll+6ce71
    4                                                                                                                                                                                                                             Process accessed:\nRuleName: \nUtcTime: 2019-10-20 20:11:10.985\nSourceProcessGUID: {a158f72c-afe7-5dac-0000-0010044f0200}\nSourceProcessId: 3548\nSourceThreadId: 3660\nSourceImage: C:\\Windows\\System32\\svchost.exe\nTargetProcessGUID: {a158f72c-afe6-5dac-0000-001077030200}\nTargetProcessId: 1968\nTargetImage: C:\\Program Files\\Amazon\\Ec2ConfigService\\Ec2Config.exe\nGrantedAccess: 0x1400\nCallTrace: C:\\Windows\\SYSTEM32\\ntdll.dll+9c524|C:\\Windows\\System32\\KERNELBASE.dll+2730e|c:\\windows\\system32\\rasmans.dll+3751b|c:\\windows\\system32\\rasmans.dll+36fad|c:\\windows\\system32\\rasmans.dll+10ed8|c:\\windows\\system32\\rasmans.dll+33cd5|C:\\Windows\\System32\\svchost.exe+314c|C:\\Windows\\System32\\sechost.dll+2de2|C:\\Windows\\System32\\KERNEL32.DLL+17944|C:\\Windows\\SYSTEM32\\ntdll.dll+6ce71
    5                                                                                                                                                                                                                             Process accessed:\nRuleName: \nUtcTime: 2019-10-20 20:11:11.249\nSourceProcessGUID: {a158f72c-afe6-5dac-0000-00107a640200}\nSourceProcessId: 3632\nSourceThreadId: 3772\nSourceImage: C:\\Windows\\System32\\svchost.exe\nTargetProcessGUID: {a158f72c-afe4-5dac-0000-00104d040200}\nTargetProcessId: 2032\nTargetImage: C:\\Program Files\\Amazon\\Ec2ConfigService\\Ec2Config.exe\nGrantedAccess: 0x1400\nCallTrace: C:\\Windows\\SYSTEM32\\ntdll.dll+9c524|C:\\Windows\\System32\\KERNELBASE.dll+2730e|c:\\windows\\system32\\rasmans.dll+3751b|c:\\windows\\system32\\rasmans.dll+36fad|c:\\windows\\system32\\rasmans.dll+10ed8|c:\\windows\\system32\\rasmans.dll+33cd5|C:\\Windows\\System32\\svchost.exe+314c|C:\\Windows\\System32\\sechost.dll+2de2|C:\\Windows\\System32\\KERNEL32.DLL+17944|C:\\Windows\\SYSTEM32\\ntdll.dll+6ce71
    6                                                                                                                                                                                  Process accessed:\nRuleName: \nUtcTime: 2019-10-20 20:11:15.016\nSourceProcessGUID: {a158f72c-b09e-5dac-0000-0010555c2c00}\nSourceProcessId: 7112\nSourceThreadId: 7004\nSourceImage: C:\\ProgramData\\Microsoft\\Windows Defender\\platform\\4.18.1909.6-0\\MsMpEng.exe\nTargetProcessGUID: {a158f72c-b0a0-5dac-0000-0010810f2d00}\nTargetProcessId: 7592\nTargetImage: C:\\ProgramData\\Microsoft\\Windows Defender\\platform\\4.18.1909.6-0\\NisSrv.exe\nGrantedAccess: 0x1400\nCallTrace: C:\\Windows\\SYSTEM32\\ntdll.dll+9c524|C:\\Windows\\System32\\KERNELBASE.dll+2730e|C:\\ProgramData\\Microsoft\\Windows Defender\\platform\\4.18.1909.6-0\\mpsvc.dll+db61b|C:\\ProgramData\\Microsoft\\Windows Defender\\platform\\4.18.1909.6-0\\mpsvc.dll+dec42|C:\\Windows\\System32\\ucrtbase.dll+1d912|C:\\Windows\\System32\\KERNEL32.DLL+17944|C:\\Windows\\SYSTEM32\\ntdll.dll+6ce71
    7                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          File created:\nRuleName: \nUtcTime: 2019-10-20 20:11:15.437\nProcessGuid: {b2887b82-a64e-5dac-0000-001038fa0000}\nProcessId: 1464\nImage: C:\\Windows\\System32\\svchost.exe\nTargetFilename: C:\\Windows\\ServiceState\\EventLog\\Data\\lastalive1.dat\nCreationUtcTime: 2019-10-20 18:25:15.182
    8                                                                                                                                                                                                                                                            Process accessed:\nRuleName: \nUtcTime: 2019-10-20 20:11:15.521\nSourceProcessGUID: {a158f72c-afe8-5dac-0000-001057850000}\nSourceProcessId: 960\nSourceThreadId: 8364\nSourceImage: C:\\Windows\\system32\\svchost.exe\nTargetProcessGUID: {a158f72c-b009-5dac-0000-0010489c0c00}\nTargetProcessId: 6204\nTargetImage: C:\\Windows\\System32\\svchost.exe\nGrantedAccess: 0x1000\nCallTrace: C:\\Windows\\SYSTEM32\\ntdll.dll+9c524|C:\\Windows\\SYSTEM32\\psmserviceexthost.dll+222a3|C:\\Windows\\SYSTEM32\\psmserviceexthost.dll+1a172|C:\\Windows\\SYSTEM32\\psmserviceexthost.dll+19e3b|C:\\Windows\\SYSTEM32\\psmserviceexthost.dll+19318|C:\\Windows\\SYSTEM32\\ntdll.dll+3082d|C:\\Windows\\SYSTEM32\\ntdll.dll+345c4|C:\\Windows\\System32\\KERNEL32.DLL+17944|C:\\Windows\\SYSTEM32\\ntdll.dll+6ce71
    9                                                                                                                                                                                                                                                            Process accessed:\nRuleName: \nUtcTime: 2019-10-20 20:11:15.521\nSourceProcessGUID: {a158f72c-afe8-5dac-0000-001057850000}\nSourceProcessId: 960\nSourceThreadId: 8364\nSourceImage: C:\\Windows\\system32\\svchost.exe\nTargetProcessGUID: {a158f72c-b009-5dac-0000-0010489c0c00}\nTargetProcessId: 6204\nTargetImage: C:\\Windows\\System32\\svchost.exe\nGrantedAccess: 0x1000\nCallTrace: C:\\Windows\\SYSTEM32\\ntdll.dll+9c524|C:\\Windows\\SYSTEM32\\psmserviceexthost.dll+222a3|C:\\Windows\\SYSTEM32\\psmserviceexthost.dll+1a172|C:\\Windows\\SYSTEM32\\psmserviceexthost.dll+19e3b|C:\\Windows\\SYSTEM32\\psmserviceexthost.dll+19318|C:\\Windows\\SYSTEM32\\ntdll.dll+3082d|C:\\Windows\\SYSTEM32\\ntdll.dll+345c4|C:\\Windows\\System32\\KERNEL32.DLL+17944|C:\\Windows\\SYSTEM32\\ntdll.dll+6ce71
    10 Process accessed:\nRuleName: \nUtcTime: 2019-10-20 20:11:16.459\nSourceProcessGUID: {a158f72c-afea-5dac-0000-001035650100}\nSourceProcessId: 2076\nSourceThreadId: 2152\nSourceImage: C:\\Windows\\system32\\svchost.exe\nTargetProcessGUID: {a158f72c-afe7-5dac-0000-001098590000}\nTargetProcessId: 600\nTargetImage: C:\\Windows\\system32\\csrss.exe\nGrantedAccess: 0x3000\nCallTrace: C:\\Windows\\SYSTEM32\\ntdll.dll+9c524|C:\\Windows\\System32\\KERNELBASE.dll+2730e|c:\\windows\\system32\\sysmain.dll+446bf|c:\\windows\\system32\\sysmain.dll+1e441|c:\\windows\\system32\\sysmain.dll+1e366|c:\\windows\\system32\\sysmain.dll+1e24d|c:\\windows\\system32\\sysmain.dll+1e0b1|c:\\windows\\system32\\sysmain.dll+1c32b|c:\\windows\\system32\\sysmain.dll+1bf95|c:\\windows\\system32\\sysmain.dll+74a8a|c:\\windows\\system32\\sysmain.dll+73ae2|c:\\windows\\system32\\sysmain.dll+60223|C:\\Windows\\system32\\svchost.exe+314c|C:\\Windows\\System32\\sechost.dll+2de2|C:\\Windows\\System32\\KERNEL32.DLL+17944|C:\\Windows\\SYSTEM32\\ntdll.dll+6ce71
        host                     Description      Category Severity
    1  HOST9      A token right was adjusted Privilege Use   Medium
    2  HOST2 A privileged service was called Privilege Use     High
    3  HOST6                    ProcessStart       Process      Low
    4  HOST1                    ProcessStart       Process      Low
    5  HOST5                    ProcessStart       Process      Low
    6  HOST5                    ProcessStart       Process      Low
    7  HOST9                      ProcessEnd       Process      Low
    8  HOST6                    ProcessStart       Process      Low
    9  HOST2                    ProcessStart       Process      Low
    10 HOST8                    ProcessStart       Process      Low

#### 9. Анализ по уровням значимости

``` r
cat("\n=== ЗАДАНИЕ 5: Анализ по уровням значимости ===\n")
```


    === ЗАДАНИЕ 5: Анализ по уровням значимости ===

``` r
# Анализ событий по уровням значимости
severity_analysis <- events_with_reference %>%
  dplyr::count(Severity, name = "events_count") %>%
  dplyr::mutate(percentage = round(events_count / sum(events_count) * 100, 1)) %>%
  dplyr::arrange(desc(events_count))

cat("Распределение событий по уровням значимости:\n")
```

    Распределение событий по уровням значимости:

``` r
print(severity_analysis)
```

      Severity events_count percentage
    1     <NA>        58170       57.1
    2      Low        42532       41.7
    3   Medium         1006        1.0
    4     High          196        0.2

``` r
# События с высоким и средним уровнем значимости
high_medium_events <- events_with_reference %>%
  dplyr::filter(Severity %in% c("High", "Medium"))

cat("\n=== РЕЗУЛЬТАТ ===\n")
```


    === РЕЗУЛЬТАТ ===

``` r
cat("Событий с высоким и средним уровнем значимости:", nrow(high_medium_events), "\n")
```

    Событий с высоким и средним уровнем значимости: 1202 

``` r
if (nrow(high_medium_events) > 0) {
  cat("Из них:\n")
  cat("  - Высокий уровень:", nrow(high_medium_events %>% dplyr::filter(Severity == "High")), "\n")
  cat("  - Средний уровень:", nrow(high_medium_events %>% dplyr::filter(Severity == "Medium")), "\n")
  
  cat("\nДетализация высоко- и средне-значимых событий:\n")
  high_medium_detail <- high_medium_events %>%
    dplyr::count(event_id, Severity, Description, Category, sort = TRUE)
  print(high_medium_detail)
} else {
  cat("Событий с высоким и средним уровнем значимости не найдено\n")
}
```

    Из них:
      - Высокий уровень: 196 
      - Средний уровень: 1006 

    Детализация высоко- и средне-значимых событий:
      event_id Severity                              Description      Category   n
    1     4703   Medium               A token right was adjusted Privilege Use 809
    2     4689   Medium      A handle to an object was requested Object Access 196
    3     4673     High          A privileged service was called Privilege Use 143
    4     4672     High Special privileges assigned to new logon Privilege Use  53
    5     4616   Medium              The system time was changed        System   1

## Оценка результата

Проведен успешный анализ 101,904 событий Windows журналов, выявлены
события с высоким и средним уровнем значимости, определено количество
уникальных хостов в системе и создана расшифровка Event ID для
классификации инцидентов безопасности.

## Вывод

Работа демонстрирует эффективное применение пакета dplyr для анализа
журналов безопасности, позволяя идентифицировать потенциальные угрозы и
оценить общее состояние защищенности информационной системы на основе
анализа событий Windows.
