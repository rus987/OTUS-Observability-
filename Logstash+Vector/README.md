# Домашнее задание

# Преобразование входящих сообщений с помощью Logstash и Vector

### Цель:
Установить и настроить Logstash и Vector таким образом, чтобы происходил парсинг входящих сообщений.


### Описание/Пошаговая инструкция выполнения домашнего задания:
Используйте тот же стенд из 2х VM как в предыдущем задании:

1. Установите Logstash на виртуальную машину с Elasticsearch. Перенастройте отправку логов в него. В Logstash добавьте парсинг логов при помощи grok фильтр (Filebeat пусть при этом шлет сырые данные).
2. Вместо Logstash установите и настройке Vector с аналогичным парсингом логов при помощи VRL.

### Задание со звездочкой

3. Настройте Dead letter queue (DLQ) в Logstash.

4. Настройте политики ILM в Logstash при отправке Elasticsearch.


В качестве результата создайте Git-репозиторий, приложите конфиги Logstash и Vector.

Приложите скриншот полученных в Kibana данных.

### Критерии оценки:
0 баллов - задание не выполнено согласно инструкции
1 балл - задание выполнено успешно согласно инструкции

### Компетенции:
Системы логирования
- Преобразование данных при помощи vector и logstash

* * *

## Установка Logstash

Устанавливаем Logstash:

`apt install logstash`

Запускаем Logstash в автозагрузку:

`systemctl enable logstash`

## Настройка Logstash

Создаем конфиг input:

`nano /etc/logstash/conf.d/input.conf`

```
input {
  beats {
    port => 5044
  }
}
````

Создаем конфиг output:

`nano /etc/logstash/conf.d/output.conf`

```
output {
  elasticsearch {
    hosts => "https://localhost:9200"
    index => "%{[@metadata][beat]}-%{[@metadata][version]}-%{+YYYY.MM.dd}"
	user => "elastic"
	password => "lkSC57vGwQAYpdSLGoos"
	ssl => true
	ssl_certificate_verification => false
	cacert => "/etc/elasticsearch/certs/http_ca.crt"
	}
}
````

Создаем конфиг filter:

`nano /etc/logstash/conf.d/filter.conf`

```
filter {
  if [type] == "nginx_access" {
    grok {
      match => { "message" => "%{IPORHOST:remote_ip} - %{DATA:user} \[%{HTTPDATE:access_time}\] \"%{WORD:http_method} %{DATA:url} HTTP/%{NUMBER:http_version}\" %{NUMBER:response_code} %{NUMBER:body_sent_bytes} \"%{DATA:referrer}\" \"%{DATA:agent}\"" }
    }
  }
  date {
    match => [ "timestamp" , "dd/MMM/YYYY:HH:mm:ss Z" ]
  }
}
````

Проверяем конфигурацию:

`/usr/share/logstash/bin/logstash --path.settings /etc/logstash -t`

Убеждаемся что все OK:

````
[2025-03-23T11:37:14,762][INFO ][logstash.runner          ] Using config.test_and_exit mode. Config Validation Result: OK. Exiting Logstash
````

Запускаем сервис logstash:

`systemctl start logstash`

Проверяем что порт поднялся:

`ss -tulpan | grep 5044`

````
tcp   LISTEN    0      4096                        *:5044                       *:*     users:(("java",pid=18106,fd=105))
````

Запустить Logstash в интерактивном режиме, чтобы посомтреть что происходит:

`logstash -f /etc/logstash/conf.d/<имя конфигурации>`

## Настрока filebeat

Диактивируем модули filebeat из предыдущего ДЗ:

`filebeat modules list`

```
Enabled:
nginx
phpfpm
postgresql
```

`filebeat modules disable nginx`

`filebeat modules disable phpfpm`

`filebeat modules disable postgresql`

Комментируем настройки Kibana и Elasticsearch Output :

`nano /etc/filebeat/filebeat.yml`

```
# =================================== Kibana ===================================

# Starting with Beats version 6.0.0, the dashboards are loaded via the Kibana API.
# This requires a Kibana endpoint configuration.
setup.kibana:

  # Kibana Host
  # Scheme and port can be left out and will be set to the default (http and 5601)
  # In case you specify and additional path, the scheme is required: http://localhost:5601/path
  # IPv6 addresses should always be defined as: https://[2001:db8::1]:5601
  #host: "elasticsearch.local:5601"

...

# ---------------------------- Elasticsearch Output ----------------------------
#output.elasticsearch:
  # Array of hosts to connect to.
  #hosts: ["https://elasticsearch.local:9200"]

  # Performance preset - one of "balanced", "throughput", "scale",
  # "latency", or "custom".
  #preset: balanced

  # Protocol - either `http` (default) or `https`.
  #protocol: "https"

  # Authentication credentials - either API key or username/password.
  #api_key: "id:api_key"
  #username: "elastic"
  #password: "lkSC57vGwQAYpdSLGoos"
  #ssl.enabled: true
  #ssl.verification_mode: "none"
```

Настраиваем отправку логов в Logsrash:

```
# ------------------------------ Logstash Output -------------------------------
output.logstash:
  # The Logstash hosts
  hosts: ["elasticsearch.local:5044"]
```

Добавляем конфигурацию сбора логов:

`nano /etc/filebeat/filebeat.yml`

```
# ============================== Filebeat inputs ===============================

filebeat.inputs:
- type: log
  enabled: true
  paths:
      - /var/log/nginx/*access.log
  fields:
    type: nginx_access
  fields_under_root: true
  scan_frequency: 5s
- type: log
  enabled: true
  paths:
      - /var/log/postgresql/*.log
  fields:
    type: postgresql
  fields_under_root: true
  scan_frequency: 5s
- type: log
  enabled: true
  paths:
      - /var/log/php8.3-fpm.log
  fields:
    type: php_fpm
  fields_under_root: true
  scan_frequency: 5s
```

Перезапускаем filebeat:

`systemctl restart filebeat`

## Установка Vector

Официальная документация: https://vector.dev/docs/setup/installation/

Vector работает как агент и как агрегатор, поэтому ставим на оба сервера.

Добавляем репозиторий:

`bash -c "$(curl -L https://setup.vector.dev)"`

Устанавливаем Vector:

`apt install vector`

Добавляем в автозагрузку:

`systemctl enable vector.service`

### Настройка Vector на cms.local

Vector по умолчанию пытается запуститься с конфигом /etc/vector/vector.yaml, чтобы использовать формат toml, то добавляем опцию в файл юнита:

`nano /lib/systemd/system/vector.service`

````
ExecStart=/usr/bin/vector --config /etc/vector/vector.toml
````

Перечитываем конфигурацию юнитов:

`systemctl daemon-reload`

Не удаляем текущий конфигурационный файл /etc/vector/vector.yaml

Создаем новый:

`nano /etc/vector/vector.toml`

```
data_dir = "/var/lib/vector"

[sources.nginx_access_logs]
type = "file"
include = [ "/var/log/nginx/access.log" ]

[sources.postgresql_logs]
type = "file"
include = [ "/var/log/postgresql/postgresql*.log" ]

[sources.php_fpm_logs]
type = "file"
include = [ "/var/log/php*-fpm.log" ]

[sinks.vector_output]
type = "vector"
inputs = [
	"nginx_access_logs",
	"postgresql_logs",
	"php_fpm_logs"
]
address = "elasticsearch.local:6000"
```

Перезапускаем службу vector:

`systemctl restart vector.service`

Смотрим логи vector:

`journalctl -f -u vector`

### Настройка Vector на elasticsearch.local

Vector по умолчанию пытается запуститься с конфигом /etc/vector/vector.yaml, чтобы использовать формат toml, то добавляем опцию в файл юнита:

`nano /lib/systemd/system/vector.service`

````
ExecStart=/usr/bin/vector --config /etc/vector/vector.toml
````

Перечитываем конфигурацию юнитов:

`systemctl daemon-reload`

Не удаляем текущий конфигурационный файл /etc/vector/vector.yaml

Создаем новый:

`nano /etc/vector/vector.toml`

```
[sources.vector_input]
type = "vector"
address = "0.0.0.0:6000"

[transforms.nginx_access_logs_parse]
type = "remap"
inputs = [ "vector_input" ]
drop_on_abort = true
source = '''
parsed = parse_nginx_log!(.message, "combined")
. = merge(., parsed)
'''

[transforms.nginx_access_logs_to_json]
type = "remap"
inputs = [ "nginx_access_logs_parse" ]
drop_on_abort = true
source = '''
. = parse_json!(.message)
'''

[sinks.elasticsearch]
type = "elasticsearch"
inputs = [ "nginx_access_logs_to_json" ]
compression = "none"
healthcheck = true
endpoints = [ "https://localhost:9200" ]
auth.strategy = "basic"
auth.user = "elastic"
auth.password = "lkSC57vGwQAYpdSLGoos"
tls.verify_certificate = false

````

```
sources:
  vector_input:
    type: vector
    address:
      - 0.0.0.0:6000
sinks:
  elasticsearch:
    type: elasticsearch
    inputs:
      - vector_input
    api_version: auto
    compression: none
    healthcheck: true
    doc_type: _doc
    endpoints:
      - https://localhost:9200
    auth.strategy: basic
    auth.user: "elastic"
    auth.password: "lkSC57vGwQAYpdSLGoos"
    tls.verify_certificate: false
```

Смотрим логи vector:

`journalctl -f -u vector`


## Меняем формат лога postgresql на json

Официальная документация: https://dev.to/uptrace/monitoring-postgresql-15-logs-with-vector-and-uptrace-4165

Формат json лога поддерживается с 15 версии PostgreSQL, для этого обновляем PostgreSQL с 14 на 15.

После обновления меняем формат в конфиге /etc/postgresql/15/main/postgresql.conf:

```
427 #------------------------------------------------------------------------------
428 # REPORTING AND LOGGING
429 #------------------------------------------------------------------------------
430
431 # - Where to Log -
432
433 log_destination = 'jsonlog'             # Valid values are combinations of
434                                         # stderr, csvlog, syslog, and eventlog,
435                                         # depending on platform.  csvlog
436                                         # requires logging_collector to be on.
437
438 # This is used when logging to stderr:
439 logging_collector = on          # Enable capturing of stderr and csvlog
440                                         # into log files. Required to be on for
441                                         # csvlogs.
442                                         # (change requires restart)

```

Перезапускаем службу postgesql:

`systemctl restart postgresql@15-main.service`

Проверяем логи в формате json:

`ls -la /var/lib/postgresql/15/main/log/*.json`

Меняем сбор логов в vector

На cms:

```
data_dir = "/var/lib/vector"

[sources.nginx_access_logs]
type = "file"
include = [ "/var/log/nginx/access.log" ]

[sources.postgresql_logs]
type = "file"
read_from = "beginning"
include = [ "/var/lib/postgresql/15/main/log/*.json" ]

[sources.php_fpm_logs]
type = "file"
include = [ "/var/log/php*-fpm.log" ]

[sinks.vector_nginx_output]
type = "vector"
inputs = [
        "nginx_access_logs"
]
address = "elasticsearch.local:6000"


#[sinks.output]
#type = "console"
#inputs = ["pg_json"]
#target = "stdout"
#encoding.codec = "json"

[sinks.vector_postgresqloutput]
type = "vector"
inputs = ["postgresql_logs"]
address = "elasticsearch.local:6001"
```

На elasticsearch.local:

```
[sources.vector_nginx_input]
type = "vector"
address = "0.0.0.0:6000"

[sources.vector_postgresql_input]
type = "vector"
address = "0.0.0.0:6001"

[transforms.nginx_access_logs_parser]
type = "remap"
inputs = [ "vector_nginx_input" ]
drop_on_abort = true
source = '''
parsed = parse_nginx_log!(.message, "combined")
. = merge(., parsed)
'''

[transforms.nginx_access_logs_to_json]
type = "remap"
inputs = [ "nginx_access_logs_parser" ]
drop_on_abort = true
source = '''
. = parse_json!(.message)
'''
[transforms.postgresql_parse_json]
type = "remap"
inputs = ["vector_postgresql_input"]
source = '''
kvs = parse_json!(.message)
if kvs == null {
  abort
}

. = merge!(., kvs)
'''
[sinks.nginx_to_elasticsearch]
type = "elasticsearch"
inputs = [ "nginx_access_logs_to_json" ]
bulk = { index = "nginx-access-logs-%Y-%m-%d" }
compression = "none"
healthcheck = true
endpoints = [ "https://localhost:9200" ]
auth.strategy = "basic"
auth.user = "elastic"
auth.password = "lkSC57vGwQAYpdSLGoos"
tls.verify_certificate = false

[sinks.postgresql_to_elasticsearch]
type = "elasticsearch"
inputs = ["postgresql_parse_json"]
bulk = { index = "postgresql-logs-%Y-%m-%d" }
compression = "none"
healthcheck = true
endpoints = [ "https://localhost:9200" ]
auth.strategy = "basic"
auth.user = "elastic"
auth.password = "lkSC57vGwQAYpdSLGoos"
tls.verify_certificate = false

```
