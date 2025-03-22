# Домашнее задание

# Установка и настройка отправки данных с помощью Beats

### Цель:

Научиться отправлять логи, метрики с помощью beats в elasticsearch.

### Описание/Пошаговая инструкция выполнения домашнего задания:

Для успешного выполнения дз вам нужно сконфигурировать hearthbeat, filebeat и metricbeat на отправку данных в elasticsearch:

1.  На виртуальной машине установите любую open source CMS, которая включает в себя следующие компоненты: nginx, php-fpm, database (MySQL or Postgresql). Можно взять из предыдущих заданий;
2.  На этой же VM установите filebeat и metricbeat. Filebeat должен собирать логи nginx, php-fpm и базы данных. Metricbeat должен собирать метрики VM, nginx, базы данных;
3.  Установите на второй VM Elasticsearch и kibana, а также heartbeat;  
    Heartbeat должен проверять доступность следующих ресурсов: веб адрес вашей CMS и порта БД

### Задания со звездочкой

4.  Настройте политики ILM так, чтобы логи nginx и базы данных хранились 30 дней, а php-fpm 14 дней;
    
5.  Настройте в filebeat dissect для логов nginx, так чтобы он переводил access логи в json;
    

В качестве результата создайте репозиторий приложите конфиги hearthbeat, filebeat и metricbeat.

Приложите скриншот полученных данных отображенных в Kibana.

Критерии оценки:  
0 баллов - задание не выполнено согласно инструкции  
1 балл - задание выполнено успешно согласно инструкции

Компетенции:  
Системы логирования

- Отправка данных с помощью filebeat, metricbeat и heartbeat

* * *

Тестовый стенд

Для домашнего задания используется два сервера (VirtualBox):

Имя: cms.local (CNAME joomla.local)  
ОС: Ubuntu 22.04.5 LTS (Jammy Jellyfish)  
IP: 192.168.5.136  
CPU: 2 vCPU  
RAM: 2 GB  
Disk: 25 GB  
Назначение: CMS Joomla (nginx, phpfpm, postgresql)

Имя: elasticsearch.local  
ОС: Ubuntu 22.04.5 LTS (Jammy Jellyfish)  
IP: 192.168.5.  
CPU: 2 vCPU  
RAM: 4 GB  
Disk: 25 GB  
Назначение: Система централизованного лоигрования Elasticsearch + Kibana

/etc/hosts

192.168.5.136 joomla.local  
192.168.5.137 prometheus.local  
192.168.5.138 elasticsearch.local

* * *

# Установка Elasticsearch + Kibana

Официальная документация: https://www.elastic.co/guide/en/elastic-stack/8.17/installing-elastic-stack.html

Добавляем репозиторий Elasticsearch (зеркало яндекс) и обновлем инфу по пакетам:

`echo "deb [trusted=yes] https://mirror.yandex.ru/mirrors/elastic/8/ stable main" | sudo tee /etc/apt/sources.list.d/elastic-8.list`

`apt update`

Запускаем установку:

`apt install elasticsearch`

Пароль супер-пользователя будет сгенерирован при установке:

```
--------------------------- Security autoconfiguration information ------------------------------

Authentication and authorization are enabled.
TLS for the transport and HTTP layers is enabled and configured.

The generated password for the elastic built-in superuser is : lkSC57vGwQAYpdSLGoos

If this node should join an existing cluster, you can reconfigure this with
'/usr/share/elasticsearch/bin/elasticsearch-reconfigure-node --enrollment-token <token-here>'
after creating an enrollment token on your existing cluster.

You can complete the following actions at any time:

Reset the password of the elastic built-in superuser with 
'/usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic'.

Generate an enrollment token for Kibana instances with 
 '/usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s kibana'.

Generate an enrollment token for Elasticsearch nodes with 
'/usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s node'.

-------------------------------------------------------------------------------------------------

```

Запускаем Elasticsearch и добавляем в автозагрузку:

`systemctl start elasticsearch`

`systemctl enable elasticsearch`

Проверяем подключение к Elasticsearch:

`curl -XGET https://127.0.0.1:9200 --insecure -u "elastic:lkSC57vGwQAYpdSLGoos"`

```
{
  "name" : "elasticsearch",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "NTBncCo9SxO_J6KFWcuoNA",
  "version" : {
    "number" : "8.17.3",
    "build_flavor" : "default",
    "build_type" : "deb",
    "build_hash" : "a091390de485bd4b127884f7e565c0cad59b10d2",
    "build_date" : "2025-02-28T10:07:26.089129809Z",
    "build_snapshot" : false,
    "lucene_version" : "9.12.0",
    "minimum_wire_compatibility_version" : "7.17.0",
    "minimum_index_compatibility_version" : "7.0.0"
  },
  "tagline" : "You Know, for Search"
}
```

# Установка Kibana

Запускаем устанвоку:

`apt install kibana`

Запускаем и добавляем в автозагрузку:

`systemctl start kibana`

`systemctl enable kibana`

## Настройка Kibana

Для Kibana надо сгенерировать токен доступа:

`/usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s kibana`

```
eyJ2ZXIiOiI4LjE0LjAiLCJhZHIiOlsiMTkyLjE2OC41LjEzODo5MjAwIl0sImZnciI6Ijg2NzFhMDNjNGI3MTBhNzRkMzNlZGExNTJkZGM1YjFjYmI5MDk1YjlmZGIzZDA2NjA2MzYyYmRjZDk1YjhjYjIiLCJrZXkiOiJZN0l2ZTVVQlF3OXZ4Q244elF5RzpMMDNaT1BWWlQ3bVZrTW9aVU16ZlNBIn0=
```

По умолчанию kibana принимает соединения на localhost:5601, меняем на 0.0.0.0 чтобы принимала соединения на всех интерфейсах:

`nano /etc/kibana/kibana.yml`

```
# Specifies the address to which the Kibana server will bind. IP addresses and host names are both valid val>
# The default is 'localhost', which usually means remote machines will not be able to connect.
# To allow connections from remote users, set this parameter to a non-loopback address.
#server.host: "localhost"

server.host: "0.0.0.0"
```

Перезапускаем службу:

`systemctl restart kibana`

Переходим на страницу http://elasticsearch.local:5601/ и вводим сгенерированный код

Далее kibana попросит ввести код подтверждения, его нужно сгенерировать так:

`/usr/share/kibana/bin/kibana-verification-code`

```
Your verification code is:  214 980 
```

![85cb8c1aebf80b2463969f2a510f9934.png](:/435da4677f2342958310cfbc5eabf10a)

Входим в kibana с учеткой elastic и паролем lkSC57vGwQAYpdSLGoos

# Установка Elastic Heartbeat

`apt-get install heartbeat-elastic`

`systemctl enable heartbeat-elastic`

Подключаем к Elastic Stack

`nano /etc/heartbeat/heartbeat.yml`

```
# =================================== Kibana ===================================
setup.kibana:
  host: "localhost:5601"
  ssl.enabled: true
  ssl.certificate_authorities: /etc/elasticsearch/certs/http_ca.crt
# ---------------------------- Elasticsearch Output ----------------------------
output.elasticsearch:
  hosts: ["localhost:9200"]
  preset: balanced
  protocol: "https"
  username: "elastic"
  password: "lkSC57vGwQAYpdSLGoos"
  ssl.enabled: true
  ssl.certificate_authorities: /etc/elasticsearch/certs/http_ca.crt
```

Загружаем рекомендуемый шаблон индекса для записи в Elasticsearch:

`heartbeat setup`

```
Overwriting lifecycle policy is disabled. Set `setup.ilm.overwrite: true` to overwrite.
Index setup finished.
````

Добавляем мониторы:

```
- type: tcp
  schedule: '@every 5s' 
  hosts: ["joomla.local:5432"]
  mode: any 
  id: joomla-postgresql-service
  name: Joomla Postgresql Service
- type: http
  schedule: '@every 5s'
  urls: ["http://joomla.local"]
  service.name: apm-service-name 
  id: joomla-http-service
  name: Joomla HTTP Service
```

Перезапускаем heartbeat-elastic:

`systemctl restart heartbeat-elastic`

Заходим в Kibana и проверяем что данные из heartbeat пошли:

![334c1a705ef7c15d2bdd4a30b060238d.png](:/8c3b110778824af7b626f60d4dab543f)

![f12d1e15c4dbe4f8692acb8efd6b1588.png](:/28382d48836a4df0a3810dbdc20d8436)

## Разрешаем подключение к кластеру PostgreSQL снаружи

### Разрешаем прослушку порта 5432 на всех интерфейсах

Расскомментируем строку "listen_addresses" и меняем на "\*" в файле "/etc/postgresql/14/main/postgresql.conf":

```
...
#listen_addresses = 'localhost'         # what IP address(es) to listen on;
...
```

Меняем на

```
...
listen_addresses = '*' 	        # what IP address(es) to listen on;
...
```

Перезапускаем службу PostgreSQL:

`systemctl restart postgresql@14-main.service`

Проверяем что PostgreSQL слушает порт 5432 на всех интерфейсах:

`ss -tulpan | grep 5432`

```
tcp   LISTEN   0      244          0.0.0.0:5432         0.0.0.0:*     users:(("postgres",pid=4692,fd=5))                                                                                                                                                                  
tcp   LISTEN   0      244             [::]:5432            [::]:*     users:(("postgres",pid=4692,fd=6))
```

# Установка и настройка Filebeat

Официальная документация: https://www.elastic.co/guide/en/beats/filebeat/current/filebeat-installation-configuration.html

Добавляем репозиторий Elasticsearch (зеркало яндекс) и обновлем инфу по пакетам:

`echo "deb [trusted=yes] https://mirror.yandex.ru/mirrors/elastic/8/ stable main" | sudo tee /etc/apt/sources.list.d/elastic-8.list`

`apt update`

Выполняем установку:

`apt install filebeat`

Добавляем в автозагрузку:

`systemctl enable filebeat`

Создаем пользователя для подключения Filebeat к Elastic Stack:

*Для прода нужно создать отдельного пользователя, для учебного стенда будем использовать уже созданную учтеку elastic *

Подключаем к Elastic Stack:

`nano /etc/filebeat/filebeat.yml`

```
# =================================== Kibana ===================================
setup.kibana:
  host: "elasticsearch.local:5601"
...
output.elasticsearch:
  hosts: ["https://elasticsearch.local:9200"]
  protocol: "https"
  username: "elastic"
  password: "lkSC57vGwQAYpdSLGoos"
  ssl.enabled: true
  ssl.verification_mode: "none" # Отключаем проверку SSL сертификата
```

Добавляем модули и настраиваем:

### nginx

`filebeat modules enable nginx`

`nano /etc/filebeat/modules.d/nginx.yml`

```
- module: nginx
  access:
    enabled: true
    var.paths: ["/var/log/nginx/access.log*"]
```

### php-fpm

По логированию PHP не разобрался, нужна помощь!

Готового модуля php8.3-fpm нет для filebeat (или я не нашел), поэтому будем использовать тип  filestream:

`nano /etc/filebeat/modules.d/php8.3-fpm.yml`

```
filebeat.inputs:
- type: filestream
  id: my-php8.3-fpm-id
  paths:
    - "/var/log/php8.3-fpm.log*"
```

### postgresql 

`filebeat modules enable postgresql`

`nano /etc/filebeat/modules.d/postgresql.yml`

```
- module: postgresql
  log:
    enabled: true
    var.paths: ["/var/log/postgresql/*.log*"]
```

Запускаем настройки, загружаем дашборды в Kibana:

`filebeat setup -e`

Перезапускаем службу:

`systemctl restart filebeat`

Проверяем логи в Kibana в разделе Discover Filebeat data

# Установка и настройка Metricbeat

Официальная документация: https://www.elastic.co/guide/en/beats/metricbeat/8.17/metricbeat-installation-configuration.html

Выполняем установку:

`apt install metricbeat`

Добавляем в автозагрузку:

`systemctl enable metricbeat`

Подключаем к Elastic Stack:

`/etc/metricbeat/metricbeat.yml`

```
# =================================== Kibana ===================================
setup.kibana:
  host: "elasticsearch.local:5601"
...
output.elasticsearch:
  hosts: ["https://elasticsearch.local:9200"]
  protocol: "https"
  username: "elastic"
  password: "lkSC57vGwQAYpdSLGoos"
  ssl.enabled: true
  ssl.verification_mode: "none" # Отключаем проверку SSL сертификата
```

Добавляем модули и настраиваем:

### Метрики VM

Отдельно модуль для сбора системных метрик активировать не требуется, он по умолчанию.

### Метрики NGINX

`metricbeat modules enable nginx`

### Метрики PostgreSQL

`metricbeat modules enable postgresql`

### Метрики PHP-FPM

`metricbeat modules enable php_fpm`

Загружаем настройки:

`metricbeat setup -e`

Убеждаемся, что дашборды загружены в Kibana:

```
{"log.level":"info","@timestamp":"2025-03-22T16:06:41.789Z","log.origin":{"function":"github.com/elastic/beats/v7/libbeat/cmd/instance.(*Beat).loadDashboards","file.name":"instance/beat.go","file.line":1306},"message":"Kibana dashboards successfully loaded.","service.name":"metricbeat","ecs.version":"1.6.0"}
Loaded dashboards
````

Перезапускаем службу:

`systemctl restart metricbeat`

Проверяем метрики в Kibana в разделе Discover
