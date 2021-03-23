**Цель**: организовать логгирование docker контейнеров. Также необходимо структурировать записи логов и обеспечить поиск и фильтрацию по их полям.

## Обзор используемого стека
Тестовое приложение будем запускать с помощью [docker-compose](https://docs.docker.com/compose/).

Для организации логгирования воспользуемся следующими технологиями:
- [fluent-bit](https://docs.fluentbit.io/manual/) - осуществляет сбор, обработку и пересылку в хранилище лог-сообщений.
- [elasticsearch](https://www.elastic.co/elasticsearch/) - централизованно хранит лог-сообщения, обеспечивает их быстрый поиск и фильтрацию.
- [kibana](https://www.elastic.co/kibana) - предоставляет интерфейс пользователю, для визуализации данных хранимых в elasticsearch.

На хабре есть [обзор стеков](https://habr.com/ru/company/southbridge/blog/517636/) технологий, используемых для логгирования контейнеров. Прежде чем идти дальше предварительно можно с ней ознакомиться.

## Подготовка тестового приложения

Для примера организуем логгирование веб-сервера Nginx.

### Подготовка Nginx

1. Создадим директорию с проектом и добавим в нее docker-compose.yml, в котором будем задавать конфигурацию запуска контейнеров приложения.
2. Определим формат логов Nginx. Для этого создадим директорию nginx c файлом nginx.conf. В нем переопределим стандартный формат логов:
    ```
    user  nginx;
    worker_processes  1;

    error_log  /var/log/nginx/error.log warn;
    pid        /var/run/nginx.pid;

    events {
        worker_connections  1024;
    }

    http {
        include       /etc/nginx/mime.types;
        default_type  application/octet-stream;

        log_format  main  'access_log $remote_addr "$request" '
                          '$status "$http_user_agent"';

        access_log  /var/log/nginx/access.log  main;

        sendfile        on;

        keepalive_timeout  65;

        include /etc/nginx/conf.d/*.conf;
    }
    ```
3. Добавим сервис **web** в docker-compose.yml:
    ```
    version: "3.8"
    services:
      web:
        container_name: nginx
        image: nginx
        ports:
          - 80:80
        volumes:
          # добавляем конфигурацию в контейнер
          - ./nginx/nginx.conf:/etc/nginx/nginx.conf
    ```

### Подготовка fluent-bit

Для начала организуем самый простой вариант логгирования.
Создадим директорию fluent-bit c конфигурационным файлом fluent-bit.conf.
Про формат и схему конфигурационного файла можно прочитать [здесь](https://docs.fluentbit.io/manual/administration/configuring-fluent-bit).
1. Fluent-bit предоставляет большое количество плагинов для сбора лог-сообщений из различных источников. Полный список можно найти [здесь](https://docs.fluentbit.io/manual/pipeline/inputs). В нашем примере мы будем использовать плагин [forward](https://docs.fluentbit.io/manual/pipeline/inputs/forward).

    Плагин вывода [stdout](https://docs.fluentbit.io/manual/pipeline/outputs/standard-output) позволяет перенаправить лог-сообщения в стандартный вывод (standard output).
    ```
    [INPUT]
        Name              forward

    [OUTPUT]
        Name stdout
        Match *
    ```
2. Добавим в docker-compose.yml сервис **fluent-bit**:
    ```
    version: "3.8"
    services:
      web:
        ...

      fluent-bit:
        container_name: fluent-bit
        image: fluent/fluent-bit
        ports:
          # необходимо открыть порты, которые используются плагином forward
          - 24224:24224
          - 24224:24224/udp
        volumes:
          # добавляем конфигурацию в контейнер
          - ./fluent-bit/fluent-bit.conf:/fluent-bit/etc/fluent-bit.conf
    ```
3. Добавим настройки логгирования для сервиса **web**:
    ```
    version: "3.8"
    services:
      web:
        ...
        depends_on:
          - fluent-bit
        logging:
          # используемый драйвер логгирования
          driver: "fluentd"
          options:
            # куда посылать лог-сообщения, необходимо что бы адрес совпадал с настройками плагина forward
            fluentd-address: localhost:24224
            # теги используются для маршрутизации лог-сообщений, тема маршрутизации будет рассмотрена ниже
            tag: nginx.logs

      fluent-bit:
        ...
      ```
4. Запустим тестовое приложение:
    ```
    docker-compose up
    ```
    Сгенерируем лог-сообщение, откроем еще одну вкладку терминала и выполним команду:
    ```
    curl localhost
    ```
    Получим лог-сообщение в следующем формате:
    ```
    [
        1616473204.000000000,
        {"source"=>"stdout",
        "log"=>"172.29.0.1 "GET / HTTP/1.1" 200 "curl/7.64.1"",
        "container_id"=>"efb81a754706b1ece694807294e9368958c5f82534df85ea44466305b326cd45",
        "container_name"=>"/nginx"}
    ]
    ```
    Сообщение состоит из:
    - временной метки, добавляемой fluent-bit;
    - лог-сообщения;
    - мета данных, добавляемых драйвером fluentd.

  На этом подготовительный этап можно считать завершенным.
  На текущем этапе структура проекта выглядит следующим образом:
  ```
  ├── docker-compose.yml
  ├── fluent-bit
  │   └── fluent-bit.conf
  └── nginx
      └── nginx.conf
  ```

## Кратко о маршрутизации лог-сообщиний в fluent-bit

Маршрутизация в fluent-bit позволяет направлять лог-сообщения через различные фильтры, для их преобразования, и в конечном итоге в один или несколько выходных интерфейсов. Для организации маршрутизации используется две основные концепции:
- тег (tag) - человеко читаемый индикатор, позволяющий однозначно определить источник лог-сообщения;
- правило сопоставления (match) - правило, определяющее куда лог-сообщение должно быть перенаправлено.

Выглядит все следующим образом:
1. Входной интерфейс присваивает лог-сообщению заданные тег.
2. В настройках фильтра или выходного интерфейса обязательно необходимо указать правило сопостовления, которое определяет выполнять обработку данного лог-сообщения или нет.

Подробнее можно прочитать в [официальной документации](https://docs.fluentbit.io/manual/concepts/data-pipeline/router).

## Очистка лог-сообщений от мета данных.

Мета данные для нас не представляют интерес, и только загромождают лог сообщение. Давайте удалим их. Для этого воспользуемся фильтром [record_modifier](https://docs.fluentbit.io/manual/pipeline/filters/record-modifier). Зададим его настройки в файле fluent-bit.conf:
  ```
  [FILTER]
      Name record_modifier
      # для всех лог-сообщений
      Match *
      # оставить только поле log
      Whitelist_key log
  ```
Теперь лог-сообщение имеет вид:
  ```
  [
      1616474511.000000000,
      {"log"=>"172.29.0.1 "GET / HTTP/1.1" 200 "curl/7.64.1""}
  ]
  ```

## Отделение логов запросов от логов ошибок

На текущий момент логи посылаемые Nginx можно разделить на две категории:
- логи с предупреждениями, ошибками;
- логи запросов.

Давайте разделим логи на две группы и будем структурировать только логи запросов.
Все логи-сообщения от Nginx помечаются тегом nginx.logs.
Поменяем тег для лог-сообщений запросов на nginx.access.
Для их идентификации мы заблаговременно добавили в начало сообщения префикс access_log.

Добавим новый фильтр [rewrite_tag](https://docs.fluentbit.io/manual/pipeline/filters/rewrite-tag). Ниже приведена его конфигурация.
  ```
  [FILTER]
      Name rewrite_tag
      # для сообщений с тегом nginx.logs
      Match nginx.logs
      # применить правило: для лог-сообщений поле log которых содержит строку
      # access_log, поменять тег на nginx.access, исходное лог-сообщение отбросить.
      Rule $log access_log nginx.access false
  ```
Теперь все лог-сообщения запросов будут помечены тегом nginx.access, что в будущем позволит нам выполнять фильтрацию логов описанным выше категориям.

## Парсинг лог-сообщения

Давайте структурируем наше лог-сообщение.
Для придания структуры лог-сообщению его необходимо распарсить.
Это делается с помощью фильтра [parser](https://docs.fluentbit.io/manual/pipeline/filters/parser).

1. Лог-сообщение представляет собой строку. Воспользуемся парсером [regex](https://docs.fluentbit.io/manual/pipeline/parsers/regular-expression), который позволяет с помощью регулярных выражений определить пары ключ-значение для информации содержащейся в лог-сообщении. Зададим настройки парсера. Для этого в директории fluent-bit создадим файл parsers.conf и добавим в него следующее:
    ```
    [PARSER]
        Name   nginx_parser
        Format regex
        Regex  ^access_log (?<remote_address>[^ ]*) "(?<method>\S+)(?: +(?<path>[^\"]*?)(?: +\S*)?)?" (?<status>[^ ]*) "(?<http_user_agent>[^\"]*)"$
        Types  status:integer
    ```
2. Обновим конфигурационный файл fluent-bit.conf. Подключим к нему файл с конфигурацией парсера и добавим фильтр parser.
    ```
    [SERVICE]
        Parsers_File /fluent-bit/parsers/parsers.conf

    [FILTER]
        Name parser
        # для сообщений с тегом nginx.access
        Match nginx.access
        # парсить поле log
        Key_Name log
        # при помощи nginx_parser
        Parser nginx_parser
    ```
3. Теперь необходимо добавить файл parsers.conf в контейнер, сделаем это путем добавления еще одного volume к сервису fluent-bit:
    ```
    version: "3.8"
    services:
      web:
        ...

      fluent-bit:
        ...
        volumes:
          - ./fluent-bit/fluent-bit.conf:/fluent-bit/etc/fluent-bit.conf
    ```
4. Перезапустим приложение, сгенерируем лог-сообщение запроса. Теперь оно имеет следующую структуру:
    ```
    [
      1616493566.000000000,
      {
        "remote_address"=>"172.29.0.1",
        "method"=>"GET",
        "path"=>"/",
        "status"=>200,
        "http_user_agent"=>"curl/7.64.1"
      }
    ]
    ```

## Сохранение лог-сообщений в elasticsearch

Теперь организуем отправку лог-сообщений на хранения в elasticsearch.

1. Добавим два выходных интерфейса в конфигурацию fluent-bit, один для лог-сообщений запросов, другой для лог-сообщений ошибок. Для этого воспользуемся плагином [es](https://docs.fluentbit.io/manual/pipeline/outputs/elasticsearch).
    ```
    [OUTPUT]
        Name  es
        Match nginx.logs
        Host  elasticsearch
        Port  9200
        Logstash_Format On
        # Использовать префикс nginx-logs для логов ошибок
        Logstash_Prefix nginx-logs

    [OUTPUT]
        Name  es
        Match nginx.access
        Host  elasticsearch
        Port  9200
        Logstash_Format On
        # Использовать префикс nginx-access для логов запросов
        Logstash_Prefix nginx-access
    ```
2. Добавим в docker-compose.yml сервисы elasticsearch и kibana.
    ```
    version: "3.8"
    services:
      web:
        ...

      fluent-bit:
        ...
        depends_on:
          - elasticsearch

      elasticsearch:
        container_name: elasticsearch
        image: docker.elastic.co/elasticsearch/elasticsearch:7.10.2
        environment:
          - "discovery.type=single-node"

      kibana:
        container_name: kibana
        image: docker.elastic.co/kibana/kibana:7.10.1
        depends_on:
          - "elasticsearch"
        ports:
          - "5601:5601"
    ```

На текущем этапе структура проекта выглядит следующим образом:
  ```
  ├── docker-compose.yml
  ├── fluent-bit
  │   ├── fluent-bit.conf
  │   └── parsers.conf
  └── nginx
      └── nginx.conf
  ```
Финальную версию проекта можно найти в репозитории.

## Результаты

Рассмотрим несколько примеров фильтрации логов.

1. Запусти тестовое приложение (должно быть запущено 4 контейнера).
2. В браузере перейдем на адрес http://localhost:5601.
3. Сгенерируем логи любым доступным способом.
4. Добавим новый индекс с паттерном nginx-*.
5. Перейдем в Kibana Discover.

Благодаря структурированию лог-сообщений мы можем фильтровать их по различным полям, к примеру:
   * показать только лог-сообщения запросов;
   * показать лог-сообщения запросов с http статусом 404;
   * отображать не все поля лог-сообщения.