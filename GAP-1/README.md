#------ Создание и подготовка тестового стенда

Для домашнего задания используется два сервера (на ноутбуке в VirtualBox):

Имя: cms.local (CNAME joomla.local)
ОС: Ubuntu 22.04.5 LTS (Jammy Jellyfish)
IP: 192.168.5.136
CPU: 2 vCPU
RAM: 2 GB
Disk: 25 GB
Назначение: CMS Joomla (nginx, phpfpm, postgresql)

Имя: prometheus.local
ОС: Ubuntu 22.04.5 LTS (Jammy Jellyfish)
IP: 192.168.5.137
CPU: 2 vCPU
RAM: 2 GB
Disk: 25 GB
Назначение: Prometheus Server

#------ Установка CMS

Устанавливаем компоненты:

- Веб сервер nginx:
# apt install nginx

- Интерпритатор php:
# apt install php8.1 php8.1-fpm php8.1-pgsql php-json php8.1-pgsql php8.1-gd php8.1-xml php8.1-mbstring

- СУБД postgresql:
# apt install postgresql-14

Связываем nginx с php:

/etc/php/8.1/fpm/pool.d/www.conf
...
listen = /run/php/php8.1-fpm.sock
...
listen.owner = www-data
listen.group = www-data
listen.mode = 0660
...

Создаем папку для CMS Joomla:

# mkdir /var/www/html/joomla

Скачиваем дистрибутив CMS Joomla и распаковываем в каталог, назначаем владельца каталога:

# wget https://downloads.joomla.org/cms/joomla5/5-2-2/Joomla_5-2-2-Stable-Full_Package.tar.gz?format=gz

# cd /var/www/html/joomla/

# tar -xvzf /root/Joomla_5-2-2-Stable-Full_Package.tar.gz

# chown -R www-data:www-data /var/www/html/

Создаем конфиг сайта Joomla:

# nano /etc/nginx/sites-available/joomla
...
server {
        root /var/www/html/joomla;
        index index.php;
        server_name joomla.local;
        location ~ \.php$ {
                fastcgi_pass unix:/run/php/php8.3-fpm.sock;
                fastcgi_index index.php;
                fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
                include /etc/nginx/fastcgi_params;
        }
}
...

Активируем конфиг сайта Joomla:

# ln -s /etc/nginx/sites-available/joomla /etc/nginx/sites-enabled/joomla

# nginx -t

# systemctl restart nginx.service

Создаем пользователя в СУБД postgresql:

# su - postgres

$ psql

postgres=# CREATE USER joomla WITH PASSWORD 'new_password';

Создаем базу данных:

postgres=# CREATE DATABASE joomla_db WITH OWNER joomla;

Запускаем установку в веб интерфейсе http://joomla.local

#------ Установка Prometheus Server

Устанавливаем вспомогательные пакеты и скачиваем Prometheus:

$ wget https://github.com/prometheus/prometheus/releases/download/v2.44.0/prometheus-2.44.0.linux-amd64.tar.gz

Создаем пользователя и нужные каталоги, настраиваем для них владельцев:

# useradd --no-create-home --shell /bin/false prometheus

# mkdir /etc/prometheus

# mkdir /var/lib/prometheus

# chown prometheus:prometheus /etc/prometheus

# chown prometheus:prometheus /var/lib/prometheus

Распаковываем архив, для удобства переименовываем директорию и копируем бинарники в /usr/local/bin:

$ tar -xvzf prometheus-2.44.0.linux-amd64.tar.gz

$ mv prometheus-2.44.0.linux-amd64 prometheuspackage

# cp prometheuspackage/prometheus /usr/local/bin/

# cp prometheuspackage/promtool /usr/local/bin/

Меняем владельцев у бинарников:

# chown prometheus:prometheus /usr/local/bin/prometheus

# chown prometheus:prometheus /usr/local/bin/promtool

По аналогии копируем библиотеки:

# cp -r prometheuspackage/consoles /etc/prometheus

# cp -r prometheuspackage/console_libraries /etc/prometheus

# chown -R prometheus:prometheus /etc/prometheus/consoles

# chown -R prometheus:prometheus /etc/prometheus/console_libraries

Создаем файл конфигурации:

# vim /etc/prometheus/prometheus.yml
...
global:
  scrape_interval: 10s
scrape_configs:
  - job_name: 'prometheus.local'
    scrape_interval: 5s
    static_configs:
      - targets: ['localhost:9100']
...

# chown prometheus:prometheus /etc/prometheus/prometheus.yml

Настраиваем сервис:

# vim /etc/systemd/system/prometheus.service
...
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target
[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
--config.file /etc/prometheus/prometheus.yml \
--storage.tsdb.path /var/lib/prometheus/ \
--web.console.templates=/etc/prometheus/consoles \
--web.console.libraries=/etc/prometheus/console_libraries
[Install]
WantedBy=multi-user.target
...

# systemctl daemon-reload

# systemctl start prometheus

# systemctl status prometheus

Проверка:

http://prometheus.local:9090/graph

#------ Установка Prometheus exporters

Ставим Prometheus exporter на каждую виртуальную машину

Скачиваем и распаковываем Node Exporter:

$ wget https://github.com/prometheus/node_exporter/releases/download/v1.5.0/node_exporter-1.5.0.linux-amd64.tar.gz

$ tar xzfv node_exporter-1.5.0.linux-amd64.tar.gz

Создаем пользователя, перемещаем бинарник в /usr/local/bin:

$ useradd -rs /bin/false nodeusr

$ mv node_exporter-1.5.0.linux-amd64/node_exporter /usr/local/bin/

Создаем сервис:

# vim /etc/systemd/system/node_exporter.service
...
[Unit]
Description=Node Exporter
After=network.target
[Service]
User=nodeusr
Group=nodeusr
Type=simple
ExecStart=/usr/local/bin/node_exporter
[Install]
WantedBy=multi-user.target
...

Запускаем сервис:

# systemctl daemon-reload
# systemctl start node_exporter
# systemctl enable node_exporter

Добавляем node_exporter в конфигурацию Prometheus Server:

# vim /etc/prometheus/prometheus.yml
...
global:
  scrape_interval: 10s
scrape_configs:
  - job_name: 'prometheus.local' # Имя задания, которое будет отображаться в консоли Prometheus Server
    scrape_interval: 5s # Интервал опроса node_exporter
    static_configs:
      - targets: ['localhost:9090'] # Точка подключения к node_exporter для сбора данных
  - job_name: 'cms.local' # Имя задания, которое будет отображаться в консоли Prometheus Server
    scrape_interval: 5s # Интервал опроса node_exporter
    static_configs:
      - targets: ['cms.local:9100'] # Точка подключения к node_exporter для сбора данных
...

Перезапускаем сервис Prometheus Server:

# systemctl restart prometheus

#------ Установка и настройка NGINX Prometheus Exporter (https://github.com/nginxinc/nginx-prometheus-exporter)

Добавляем страницу статитстики в nginx.conf:

...
location = /basic_status {
    stub_status;
    allow 127.0.0.1;
    allow ::1;
    deny all;
}
...

# nginx -t

# systemctl restart nginx.service

$ curl http://localhost/basic_status
Active connections: 1
server accepts handled requests
 1 1 1
Reading: 0 Writing: 1 Waiting: 0

Скачиваем архив с исполняемым файлом и распаковываем:

$ wget https://github.com/nginxinc/nginx-prometheus-exporter/releases/download/v1.4.0/nginx-prometheus-exporter_1.4.0_linux_amd64.tar.gz

tar xzfv nginx-prometheus-exporter_1.4.0_linux_amd64.tar.gz

Проверяем запуск экспортера и сбора данных:

./nginx-prometheus-exporter --nginx.scrape-uri=http://localhost/stub_status

http://<имя или IP хоста nginx>:9113/metrics

Создаем пользователя, перемещаем бинарник в /usr/local/bin, назначаем владельца:

# useradd -rs /bin/false nginx-prom-exp

# mv nginx-prometheus-exporter /usr/local/bin/

# chown nginx-prom-exp:nginx-prom-exp /usr/local/bin/nginx-prometheus-exporter

Создаем сервис:

# vim /etc/systemd/system/nginx-prometheus-exporter.service
...
[Unit]
Description=Nginx Prometheus Exporter
After=network.target
[Service]
User=nginx-prom-exp
Group=nginx-prom-exp
Type=simple
ExecStart=/usr/local/bin/nginx-prometheus-exporter --nginx.scrape-uri=http://localhost/stub_status
[Install]
WantedBy=multi-user.target
...

Запускаем сервис:

# systemctl daemon-reload
# systemctl start nginx-prometheus-exporter
# systemctl enable nginx-prometheus-exporter

Добавляем nginx-prometheus-exporter в конфигурацию Prometheus Server:

# vim /etc/prometheus/prometheus.yml
...
global:
  scrape_interval: 10s
scrape_configs:
  - job_name: 'prometheus.local' # Имя задания, которое будет отображаться в консоли Prometheus Server
    scrape_interval: 5s # Интервал опроса node_exporter
    static_configs:
      - targets: ['localhost:9090'] # Точка подключения к node_exporter для сбора данных
  - job_name: 'cms.local' # Имя задания, которое будет отображаться в консоли Prometheus Server
    scrape_interval: 5s # Интервал опроса node_exporter
    static_configs:
      - targets: ['cms.local:9100'] # Точка подключения к node_exporter для сбора данных
  - job_name: 'nginx-cms.local' # Имя задания, которое будет отображаться в консоли Prometheus Server
    scrape_interval: 5s # Интервал опроса node_exporter
    static_configs:
      - targets: ['cms.local:9113'] # Точка подключения к node_exporter для сбора данных
...

Перезапускаем сервис Prometheus Server:

# systemctl restart prometheus

#------ Установка и настройка php-fpm_exporter (https://github.com/hipages/php-fpm_exporter?tab=readme-ov-file)

Включаем страницу статитстики PHP FPM, раскомментируем следующие строки в конфиге /etc/php/8.3/fpm/pool.d/www.conf:

pm.status_path = /status
pm.status_listen = 127.0.0.1:9001
ping.path = /ping

Перезапускаем PHP FPM:

systemctl restart php8.3-fpm.service

Скачиваем архив с исполняемым файлом и распаковываем:

$ wget https://github.com/hipages/php-fpm_exporter/releases/download/v2.2.0/php-fpm_exporter_2.2.0_linux_amd64.tar.gz

tar xzfv php-fpm_exporter_2.2.0_linux_amd64.tar.gz

Проверяем запуск экспортера для сбора данных с PHP FPM:

./php-fpm_exporter get --phpfpm.scrape-uri tcp://127.0.0.1:9001/status

Address:                tcp://127.0.0.1:9001/status
Pool:                   www
Start time:             Mon, 06 Jan 2025 11:11:15 +0000
Start since:            397
Accepted connections:   4
Listen Queue:           0
Max Listen Queue:       0
Listen Queue Length:    0
Idle Processes:         2
Active Processes:       0
Total Processes:        2
Max active processes:   1
Max children reached:   0
Slow requests:          0

Проверяем запуск экспортера для сбора данных с PHP FPM в режиме сервера:

./php-fpm_exporter server --phpfpm.scrape-uri tcp://127.0.0.1:9001/status

http://<имя или IP хоста phpfpm>:9253/metrics

Создаем пользователя, перемещаем бинарник в /usr/local/bin, назначаем владельца:

# useradd -rs /bin/false phpfpm-prom-exp

# mv php-fpm_exporter /usr/local/bin/

# chown phpfpm-prom-exp:phpfpm-prom-exp /usr/local/bin/php-fpm_exporter

Создаем сервис

# vim /etc/systemd/system/php-fpm-prometheus-exporter.service
...
[Unit]
Description=PHP-FPM Prometheus Exporter
After=network.target
[Service]
User=phpfpm-prom-exp
Group=phpfpm-prom-exp
Type=simple
ExecStart=php-fpm_exporter server --phpfpm.scrape-uri tcp://127.0.0.1:9001/status
[Install]
WantedBy=multi-user.target
...

Запускаем сервис

# systemctl daemon-reload
# systemctl start php-fpm-prometheus-exporter
# systemctl enable php-fpm-prometheus-exporter

Добавляем php-fpm_exporter в конфигурацию Prometheus Server:

# vim /etc/prometheus/prometheus.yml
...
global:
  scrape_interval: 10s
scrape_configs:
  - job_name: 'prometheus.local' # Имя задания, которое будет отображаться в консоли Prometheus Server
    scrape_interval: 5s # Интервал опроса node_exporter
    static_configs:
      - targets: ['localhost:9090'] # Точка подключения к node_exporter для сбора данных
  - job_name: 'cms.local' # Имя задания, которое будет отображаться в консоли Prometheus Server
    scrape_interval: 5s # Интервал опроса node_exporter
    static_configs:
      - targets: ['cms.local:9100'] # Точка подключения к node_exporter для сбора данных
  - job_name: 'nginx-cms.local' # Имя задания, которое будет отображаться в консоли Prometheus Server
    scrape_interval: 5s # Интервал опроса node_exporter
    static_configs:
      - targets: ['cms.local:9113'] # Точка подключения к node_exporter для сбора данных
  - job_name: 'phpfpm-cms.local' # Имя задания, которое будет отображаться в консоли Prometheus Server
    scrape_interval: 5s # Интервал опроса node_exporter
    static_configs:
      - targets: ['cms.local:9253'] # Точка подключения к node_exporter для сбора данных
...

Перезапускаем сервис Prometheus Server

# systemctl restart prometheus

#------ Установка и настройка PostgreSQL Server Exporter (https://github.com/prometheus-community/postgres_exporter)

Скачиваем архив с исполняемым файлом и распаковываем:

$ wget https://github.com/prometheus-community/postgres_exporter/releases/download/v0.16.0/postgres_exporter-0.16.0.linux-amd64.tar.gz

$ tar xzfv postgres_exporter-0.16.0.linux-amd64.tar.gz

Перемещаем бинарник в /usr/local/bin, назначаем владельца:

$ cd postgres_exporter-0.16.0.linux-amd64/

# mv postgres_exporter /usr/local/bin/

# chown postgres:postgres /usr/local/bin/postgres_exporter

Создаем сервис

# vim /etc/systemd/system/postgres-prometheus-exporter.service
...
[Unit]
Description=PostgreSQL Prometheus Exporter
After=network.target
[Service]
EnvironmentFile=/etc/prometheus/postgres-prometheus-exporter.config
User=postgres
Group=postgres
Type=simple
ExecStart=
[Install]
WantedBy=multi-user.target
...

Создаем папку /etc/prometheus/

Создаем файл /etc/prometheus/postgres-prometheus-exporter.config
...
DATA_SOURCE_NAME="user=postgres host=/var/run/postgresql/ sslmode=disable" postgres_exporter
...

Запускаем сервис

# systemctl daemon-reload
# systemctl start postgres-prometheus-exporter
# systemctl enable postgres-prometheus-exporter

Добавляем postgres_exporter в конфигурацию Prometheus Server

# vim /etc/prometheus/prometheus.yml
...
global:
  scrape_interval: 10s
scrape_configs:
  - job_name: 'prometheus.local' # Имя задания, которое будет отображаться в консоли Prometheus Server
    scrape_interval: 5s # Интервал опроса node_exporter
    static_configs:
      - targets: ['localhost:9090'] # Точка подключения к node_exporter для сбора данных
  - job_name: 'cms.local' # Имя задания, которое будет отображаться в консоли Prometheus Server
    scrape_interval: 5s # Интервал опроса node_exporter
    static_configs:
      - targets: ['cms.local:9100'] # Точка подключения к node_exporter для сбора данных
  - job_name: 'nginx-cms.local' # Имя задания, которое будет отображаться в консоли Prometheus Server
    scrape_interval: 5s # Интервал опроса node_exporter
    static_configs:
      - targets: ['cms.local:9113'] # Точка подключения к node_exporter для сбора данных
  - job_name: 'postgres-cms.local' # Имя задания, которое будет отображаться в консоли Prometheus Server
    scrape_interval: 5s # Интервал опроса node_exporter
    static_configs:
      - targets: ['cms.local:9187'] # Точка подключения к node_exporter для сбора данных
...

Перезапускаем сервис Prometheus Server

# systemctl restart prometheus

#------ Установка и настройка Blackbox exporter (https://github.com/prometheus/blackbox_exporter)

Скачиваем архив с исполняемым файлом и распаковываем:

$ wget https://github.com/prometheus/blackbox_exporter/releases/download/v0.25.0/blackbox_exporter-0.25.0.linux-amd64.tar.gz

$ tar xzfv blackbox_exporter-0.25.0.linux-amd64.tar.gz

Создаем пользователя, перемещаем бинарник в /usr/local/bin, назначаем владельца:

# useradd -rs /bin/false blackbox-prom-exp

$ cd blackbox_exporter-0.25.0.linux-amd64/

# mv blackbox_exporter /usr/local/bin/

# chown blackbox-prom-exp:blackbox-prom-exp /usr/local/bin/blackbox_exporter

Создаем сервис

# vim /etc/systemd/system/blackbox-prometheus-exporter.service
...
[Unit]
Description=Blackbox Prometheus Exporter
After=network.target
[Service]
User=blackbox-prom-exp
Group=blackbox-prom-exp
Type=simple
ExecStart=/usr/local/bin/blackbox_exporter --config.file=/etc/prometheus/blackbox.yml
[Install]
WantedBy=multi-user.target
...

Создаем конфигурационный файл или копируем готовый:

# cp blackbox.yml /etc/prometheus/

# vim /etc/prometheus/blackbox.yml

Запускаем сервис

# systemctl daemon-reload
# systemctl start blackbox-prometheus-exporter
# systemctl enable blackbox-prometheus-exporter

Добавляем blackbox проверки в конфигурацию Prometheus Server:

# vim /etc/prometheus/prometheus.yml
...
global:
  scrape_interval: 10s
scrape_configs:
  - job_name: 'prometheus.local' # Имя задания, которое будет отображаться в консоли Prometheus Server
    scrape_interval: 5s # Интервал опроса node_exporter
    static_configs:
      - targets: ['localhost:9090'] # Точка подключения к node_exporter для сбора данных
  - job_name: 'cms.local' # Имя задания, которое будет отображаться в консоли Prometheus Server
    scrape_interval: 5s # Интервал опроса node_exporter
    static_configs:
      - targets: ['cms.local:9100'] # Точка подключения к node_exporter для сбора данных
  - job_name: 'nginx-cms.local' # Имя задания, которое будет отображаться в консоли Prometheus Server
    scrape_interval: 5s # Интервал опроса node_exporter
    static_configs:
      - targets: ['cms.local:9113'] # Точка подключения к node_exporter для сбора данных
  - job_name: 'postgres-cms.local' # Имя задания, которое будет отображаться в консоли Prometheus Server
    scrape_interval: 5s # Интервал опроса node_exporter
    static_configs:
      - targets: ['cms.local:9187'] # Точка подключения к node_exporter для сбора данных
  - job_name: 'joomla website availability'
    metrics_path: /probe
    params:
      module: [http_2xx]  # Look for a HTTP 200 response.
    static_configs:
      - targets:
        - http://joomla.local       # Target to probe with http.
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: 127.0.0.1:9115  # The blackbox exporter's real hostname:port.
...

Перезапускаем сервис Prometheus Server

# systemctl restart prometheus

#------ Предоставление результата домашнего задания

Зарегистрировался на github.com, создал публичный репозиторий https://github.com/rus987/OTUS-Observability-.git
