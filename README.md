# 1. Тема, MVP, анализ аудитории.
**Облако Mail.ru** - облачный сервис для хранения и обмена файлами.

### MVP
1. Загрузка файлов на облако;
2. Скачивание файлов с облака;
3. Возможность создания директорий для распределения по ним файлов;
4. Отображение файлов и директорий списокм (листинг);
5. Отображение метаинформации о файле (название, формат, вес, дата добавления);
6. Возможность предоставить доступ к хранилищу или к папке другому человеку;

### Анализ трафика
- MAU - 23 млн.[^1]
- DAU - 2.5 млн.[^1]. Примем во внимание, что речь в заявлении идет о пользователях ресурса без учета пользователей мобильного приложения.
- За год пользователи загружают 30 млрд. файлов[^2]
- Общий объем хранилища составляет 600 петабайт[^2]

### Анализ аудитории
По данным сайта Similarweb[^3] аудитория распределяется следующим образом:
```mermaid
pie title Страна
    "Россия": 86.1
    "Беларусь": 3.77
    "Казахстан": 2.64
    "Другие": 7.49
```
Таким образом, можно сказать, что вся ключевая аудитория располагается на территории СНГ.

## Отличия от конкурентов
От конкурентов в лице Google Drive, Dropbox и других облачных хранилищ, "облако" отличает интеграция с почтовыми сервисами. 

# 2. Расчет нагрузки
## Продуктовые метрики
- **Средний размер хранилища мользователя** - так как компания не разглашает информацию о среднем хранилище пользователя, я провел опрос с участием 100 человек и установил, что средний размер облачного хранилища пользователя составляет 37 Гб.
- **DAU** - 2.5 млн. Как уже было сказано ранее, это пользователи ресурса без учета мобильного приложения. Основываясь на том, что основная доля пользователей Облака Mail.ru из России и того, что доля мобильного трафика в России составляет 45,3%[^6], то примем DAU равным 4.6 млн, полагая, что все пользователи мобильных устройств используют мобильное приложения.
- **Среднее количество действий пользователя** - т.к. за год пользователи загружают 29.3 млрд. файлов[^5], то это в среднем 80 млн. файлов в день. Исходя из дневной аудитории в 4.6 млн. пользователей можно предположить, что в день средний пользователь загружает 3 файла. Такой большое значение связано, скорее всего, с тем, что у мобильного и десктопного приложение есть возможность автоматически загружать на облако данные из галереи на телефоне, либо же из выбранной папки на компьютере. Ввиду того, что мы разрабатываем MVP - опустим функцию синхронизации облачного хранилища с хранилищем на телефоне или ПК. Тогда, примем число ручных загрузок файлов равным 3, а число скачиваний равным 6.

### Логическая схема БД

Нам также понадобится хранить различные метаданные пользователей и файлов: имя пользователя, ID пользователя, имя файла, путь к нему в файловом хранилище, является ли файл директорией и т.д. Рассмотрим примерную схему БД метаданных:
```mermaid
erDiagram
USER{
    uuid ID PK
    text user_name
    text password
    timestamp created_at 
}

FILE_META{
    uuid ID PK
    text file_name
    text description
    bigint size
    text type
    uuid directory_id FK
    text checksum
    uuid owner_id
    timestamp created_at  
}

FILE_DATA{
    uuid ID PK, FK
    text data
}

FILE_META ||--|| FILE_DATA : has

DIRECTORY{
    uuid ID PK
    uuid owner_id FK
    text name
    ltree path
    timestamp created_at  
}

FILE_ACCESS {
    uuid user_id FK
    uuid file_id FK
    timestamp created_at
}

DIRECTORY_ACCESS {
    uuid user_id FK
    uuid directory_id FK
    enum mode 
    timestamp created_at
}

USER }|--o{ FILE_META : owns
USER }|--o{ DIRECTORY : owns
USER ||--o{ FILE_ACCESS : has
FILE_META ||--|{ FILE_ACCESS : allows
FILE_META }o--|| DIRECTORY : in
USER ||--o{ DIRECTORY_ACCESS : has
DIRECTORY ||--|{ DIRECTORY_ACCESS : allows
```
Посчитаем теперь сколько метаинформации придется дополнительно хранить, ограничив длину текста 30 символами:
Тип данных | Размер
-----------| ------
Uuid       | 16 байт
Bigint     | 8 байт
Text       | 31 байт
Boolean    | 1 байт
Timestamp  | 8 байт

### Описание схемы БД
Таблица          | Описание
-----------------| ------
USER             | Таблица с информацией о пользователе
FILE_META        | Таблица, содержащая основную метаинформацию о файле
FILE_DATA        | Сам файл
DIRECTORY        | Таблица со списком директорий. У каждой директории есть родительская директория, так что можно создавать директории внутри директорий
FILE_ACCESS      | Таблица содержит информацию о доступе пользователей к файлам, который они могут скачать и метаинформацию о которых они могут просмотреть
DIRECTORY_ACCESS | Таблица содержит информацию о доступе пользователей к директориям и о режиме доступа: запись (w), чтение (r), чтение и запись (rw). При доступе только на запись у пользователя есть возможность добавлять свои файлы в директорию, но нет возможности видеть чужие файлы. При записи на чтение есть возможность видеть и скачивать чужие файлы.

### RPS для каждой таблицы

Тогда для каждого пользователя мы дополнительно храним 78 байт. Для каждого файла мы дополнительно храним 110 байт и также мы храним директории для организации файлов весом 23 байта.
Найти информацию о среднем весе одного файла не удалось, поэтому, основываясь на том, что в основном пользователи хранят на облаке изображения со средним весом 1.75 Мб, чуть реже PDF со средним весом 1.5 Мб и еще реже mp4 со средним весом 300 Мб[^5] проведем расчеты. В пятерке самых распространенных файлов первые 3 места занимают изображения в разных форматах, 4 место PDF и 5 место MP4. На основе этих данных предположим, что 50% файлов, хранимых пользователями - фото, 30% - документы, 15% - видео, и 5% - прочие файлы весом 0-20 МБ. Тогда средний размер одного файла 0.5 * 1.75 Мб + 0.3 * 1.5 Мб + 0.15 * 300Мб + 0.05 * 10Мб = 47 Мб.

Предположим, что средний пользователь просматривает информацию о каждом файле до скачивания. Также пусть пользователь 3 раза в день заправшивает список файлов и этот список будет содержать максимум 20 файлов.

Итого:
Параметр | Значение
------ | ------
Месячная аудитория | 23 млн. человек
Дневная аудитория | 4.6 млн. человек
Средний размер хранилища пользователя | 37 Гб
Средний размер одного файла | 47 Мб
Загрузка файлов | 3 файлов/день
Скачивание файлов | 6 файлов/день
Листинг | 3 запроса по 20 файлов в день
Метаинформация о файле | 6 запросов/день
Метаданные пользователя | 78 байт
Метаданные файла | 119 байт

## Технические метрики
### Размер хранения
Компания утверждает, что размер хранилища составляет 600 Пб[^2].
### Сетевой трафик
Посчитаем сетевой трафик при загрузке файла на облако и скачивании файла с облака. Также нам нужно
- **Загрузка файлов на облако** - 3 запроса/день * (47 Мб + 119 байт) = 141 Мб/день;
- **Скачивание файлов с облака** - 282 Мб/день;

Тогда средний суточный трафик загрузки будет равен: 4.6 млн. * (3 запроса/день / 86400) * (47 Мб + 119 байт) * 8 / 1024 = 58 Гбит/с. Для скачивания трафик будет, соответственно, равен 116 Гбит/с. Всего за сутки трафик загрузки будет составлять: 4.6 млн * 3 запроса/день * (47 Мб + 119 байт) / 1024 = 633400 Гб/сутки. Суточный трафик скачивания - 1266800 Гб/сутки.

При запросе списка из 20 файлов мы запрашиваем только название (text весом 31 байт) и формат (text весом 7 байт). Получаем, что за день 4.6млн * (3 запроса/день /86400) * (38 байт) * 8 = 47,42 Кбит/с. За сутки 4.6 млн * 3 запроса/день * (38 байт) = 500 Мб/сутки.
При запросе метаинформации о файле мы запрашиваем его название (31 байт), его формат (7 байт), его размер (8 байт) и дату создания (8 байт). Итого 54 байта. Пиковый показатель 4.6млн * (6 запросов/день / 86400) * (54 байта) * 8 = 135 Кбит/с. За сутки 4.6млн * 6 * 54 байта = 1,4 ГБ/сутки.

Для дальнейших расчетов примем, что в пиковые часы мы будем ограничивать трафик, так как скорость передачи информации не играет существенной роли в контексте облачных сервисов. 
Так как каналы дуплексные, то итоговые значения не нужно скалдывать, так как загрузка и скачивание происходят параллельно.

Тип трафика | Пиковое в Гбит/c | Суммарный суточной Гб/cутки
------ | ----- | -----
Загрузка файла | 58 | 633400
Скачивание файла | 116 | 1266800
Листинг файлов | 47,42 * 2 ^ (-20)| 0,5
Отображение информации о файле | 135 * 2 ^ (-20) | 1,4
Итого | 116 | 1266801

### RPS
Посчитаем RPS в предположении, что пиковый RPS будет в 2 раза выше среднего.

Запрос | средний RPS | Пиковый RPS  | 
------ | ------ | -----
Загрузка файла | 4.6 млн * 3 / 86400 = 160 | 320
Скачивание файла | 4.6 млн * 6 / 86400 = 320 | 640
Листинг файлов | 4.6млн * 3 / 86400 = 160 | 320
Отображение информации о файле | 4.6 млн * 6 / 86400 = 320 | 640
Итого | 960 | 1920

## Глобальная балансировка нагрузки
### Функциональное разбиение по доменам
- Основной домен (точка входа): cloud.mail.ru;
- Отдача статики: static-XX.cloud.mail.ru (XX - номер сервера);
- Отдача динамического контента (листинга файлов, метаинформации): api.cloud.mail.ru;
- Загрузка файлов в хранилище: s3-XX.cloud.mail.ru (XX - номер сервера);
### Расположение датацентров
Исходя из данных similarweb основная часть пользователей находится в России[^3]. Значит, для глобальной балансировки нагрузки можно обойтись без DNS-балансировки, так как нам нужна балансировка внутри страны.

Теперь нужно разобраться с расположением датацентров внутри страны. Наибольшее количество пользователей проживает в Москве, Санкт-Петербурге, Уфе, Краснодаре и Новосибирске. Значит, нужно расположить датацентры в этих городах. Стоит принять во внимание, что в этом списке нет ни одного города на Дальнем Востоке. Основываясь на численности населения, установим датацентр в Хабаровске, чтобы у пользователей из Дальнего Востока был датацентр поблизости. Посмотреть насположение серверов можно на карте:
![Карта датацентров](imgs/DCmap.png)

### Функционал датацентров
Основной домен (cloud.mail.ru) и домен для отдачи динамического контента (api.cloud.mail.ru) расположим в датацентре Москве.

CDN для отдачи статики (static-XX.cloud.mail.ru) и S3 хранилища расположим в ДЦ в Москве, Санкт-Петербурге, Уфе, Краснодаре, Новосибирске и Хабаровске.

### Расчет плотности запросов
Сделаем выводы о распределении плотности нагрузки на основании плотности населения:
Учтем не только население городов, но и население субъектов федерации, в которых эти города находятся:
- Москва: 12,6 млн человек.
- Санкт-Петербург: ~5,6 млн человек.
- Республика Башкортостан (субъект федерации, где находится Уфа): ~4,1 млн человек.
- Краснодарский край (субъект федерации, где находится Краснодар): ~5,8 млн человек.
- Новосибирская область (субъект федерации, где находится Новосибирск): ~2,8 млн человек.
- Дальневосточный федеральный округ (ДФО): ~8,1 млн человек.

Тогда получаем следующее распределение трафика:
- Москва: весь трафик API + 32% трафика CDN
- Санкт-Петербург: 14% трафика CDN
- Уфа: 11% трафика CDN
- Краснодар: 15% трафика CDN 
- Новосибирск: 7% трафика CDN
- Хабаровск: 21% трафика CDN

| Тип запроса                    | Москва, RPS | Санкт-Петербург CDN, RPS | Уфа CDN, RPS            | Краснодар CDN, RPS    | Новосибирск CDN, RPS | Хабаровск CDN, RPS |
|--------------------------------|-------------|--------------------------|-------------------------|-----------------------|----------------------|--------------------|
| Загрузка файла                 | 103         | 45                       | 35                      | 48                    | 23                   | 67                 |
| Скачивание файла               | 206         | 90                       | 70                      | 96                    | 46                   | 134                |
| Листинг файлов                 | 320         | 0                        | 0                       | 0                     | 0                    | 0                  |
| Отображение информации о файле | 640         | 0                        | 0                       | 0                     | 0                    | 0                  |
| Итого                          | 1269        | 135                      | 105                     | 144                   | 69                   | 201                |

### Схема балансировки
Для балансировки нагрузки будем использовать технологию BGP Anycast:
- Все датацентры объединяются в одну автономную систему и анонсируют один IP адрес, благодаря чему все запросы с помощью технологии BGP Anycast приходят на ближайший датацентр. 
- Во всех датацентрах установлены CDN сервера и S3 хранилища. Если файл, запрашиваемый пользователем, отсутсвтует на CDN сервере, то совершается запрос к S3 хранилищу, после чего полученный файл кешируется CDN сервером.
- Если один из ДЦ выходит из строя, то весь трафик автоматически перенаправляется в другой ближайший ДЦ. 

## Локальная балансировка нагрузки
Итак, трафик пришел в датацентр после глобальной балансировки нагрузки. Теперь нужно придумать способ добавить балансировку внутри самого датацентра. Разумно будет установить L7-балансировщик, но необходимо сделать систему отказоустойчивой, поэтому добавим к нему L4-балансировку.

## L4-балансировщик
Для L4-балансировки будем использовать Virtual Server via Direct Routing. С его помощью мы будем распределять нагрузку между L7-балансировщиками. Мы можем применить эту технологию, так как работаем внутри одного датацентра и L4 и L7 балансировщики находятся в одной подсети.

Для обеспечения отказоустойчивости необходимо следить за состоянием L7-балансировщиков. Используем сервис Keepalived, который будет делать запросы на L7-балансировщики и проверять их состояние.

## L7-балансировка
Установим внутри каждого датацентра веб-сервер с установленным nginx, который будет выоплнять следующий функции:
- Проксирование запросов к файловому хранилищу;
- Обратное проксирование;
- Терминация SSL;
- Распределение запросов по серверам API;
- Ограничение частоты запросов;
- Поддержание keepalive-соединений;
- Сжатие данных;
- Установка таймаутов;
- Перенаправление идемпотентных запросов с нерабочего сервера на рабочий;
- API Gateway;

### API Gateway
Важно отметить, что внутри каждого датацентра L7-балансировшик будет играть роль API Gateway, то есть шлюза, который вначале сделает запрос для авторизации пользователя, после чего перенаправит запрос пользователя либо на API сервер, либо на CDN сервер, либо на S3 сервер. Это позволяет нам сокрыть от пользователя детали реализции нашей архитектуры, так как внешний пользователь будет взаимодействовать только с доменом cloud.mail.ru.

### Терминация SSL
Терминация SSL создаст дополнительную нагрузку на процессор. Примем во внимание, что средний размер SSL-сессии - 1 КБ, а средняя нагрузка современных алгоритмов шифрования на процессор составляет 0,75 мс на одно соединение. Тогда **объем данных = RPS * 1 Кб**, **Необходимые ядра = RPS * 0,75 мс**.

| Датацентр       | RPS    | Объем данных КБ/с | Требуемые ядра CPU |
|-----------------|--------|-------------------|--------------------|
| Москва          | 1269   | 1269              | 1                  |
| Санкт-Петербург | 135    | 135               | 1                  |
| Уфа             | 105    | 105               | 1                  |
| Краснодар       | 144    | 144               | 1                  |
| Новосибирск     | 69     | 69                | 1                  |
| Хабаровск       | 201    | 201               | 1                  |


## Физическая схема БД
### Индексы
Для начала определим, какие операции будут происходить наиболее часто. Во-первых, это получение файлов и директорий конкретного пользователя при его переходе на свою страницу в облаке. Во-вторых, это поиск и получение файлов внутри директории. В-третьих, проверка прав пользователя на доступ к файлу/директории. Тогда нам нужны следующие индексы:
1. CREATE INDEX idx_file_in_directory ON FILE_META(directory_id); - для получения файлов в директории
2. CREATE INDEX idx_owner_files ON FILE_META(owner_id); - для получения файлов пользователя
3. CREATE INDEX idx_search_file ON FILE_META(file_name, directory_id); - для поиска по имени файла внутри директории
4. CREATE INDEX idx_owner_directory ON DIRECTORY(owner_id); - для получения директорий пользователя
5. CREATE INDEX idx_directory_path ON DIRECTORY USING GIST (path); - для работы со вложенными директориями
6. CREATE INDEX idx_file_access ON FILE_ACCESS(user_id, file_id); - для проверки прав на доступ к файлу
7. CREATE INDEX idx_directory_access ON DIRECTORY_ACCESS(user_id, directory_id) INCLUDE (mode); - для проверки прав на доступ к директории




## Список источников
[^1]: [Заявления компании об активных пользователях](https://habr.com/ru/news/711772/)
[^2]: [Объем пользовательских данных в Облаке Mail.ru](https://hi-tech.mail.ru/news/102223-raskryit-obem-polzovatelskih-dannyih-v-oblake-mailru/)
[^3]: [Аналитика трафика cloud.mail.ru](https://www.similarweb.com/website/cloud.mail.ru/#ranking)
[^4]: [Дневная нагрузка почты mail.ru](https://www.cnews.ru/news/line/2023-10-18_pochta_mailru_obrabatyvaet)
[^5]: [Загружаемые файлы облака mail.ru](https://searchengines.guru/ru/news/2058384)
[^6]: [Анализ использования Интернета за 2024 год](https://www.meltwater.com/en/2024-global-digital-trends)
[^7]: [Средний размер файла в облачном хранилище](https://www.globaldots.com/resources/blog/how-much-is-stored-in-the-cloud/)
[^8]: [Интерес к облачных хранилищам в России](https://www.yota.ru/corporate/press/1124222)
