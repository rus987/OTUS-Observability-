# Корпоративный веб сайт
# Проектная работа по мониторингу и логированию

## Содержание проектной работы
В проектной работе используются следующие компоненты:

Пользовательский сервис - веб сайт компании http://joomla.local, состоящий из:

- CMS: Joomla 5.2.6:
- ОС: Ubuntu 22.04.05 LTS;
- Веб сервер: NGINX 1.18.0;
- Скриптовый интерпретатор: PHP-FPM 8.3;
- СУБД: PostgreSQL 15.

Мониторинг - Zabbix (Zabbix агент, Zabbix сервер):

- Агентный мониторинг:
    
    - ОС:
        - Использование ЦП;
        - Использование ОП;
        - Использование диска.
    - Веб сервер NGINX:
        - Статус службы;
        - Доступность порта;
        - Статистика соединений.
    - СУБД PostgreSQL:
        - Статус службы;
        - Доступность порта;
        - Статистика соединений;
    - Скриптовый интерпретатор PHP-FPM:
        - Статус службы;
        - Статистика соединений.
- Безагентный мониторинг:
    
    - Проверка доступности сервера (ping + доступность порта);
    - Проверка доступности веб страницы (HTTP статус, время отклика);
    - Проверка SSL сертификата (срок использования).

- Визуализация:
	- Дашборд в Grafana
- Оповещение:
	- Оповещение в Telegram

Логирование - OpenSearch (Vector, OpenSearch, OpenSearch Dashboards):

- Сбор структурированных логов:
    - Веб сервер NGINX:
        - access.log
        - error.log
    - СУБД PostgreSQL:
        - postgresql-15-main.log
    - Скриптовый интерпретатор PHP-FPM:
        - /var/log/php8.3-fpm.log
- Визуализация: - OpenSearch Dashboards

## Тестовый стенд для проектной работы

Для проектной работы используются следующие серверы (VirtualBox):

Имя: cms.local (CNAME joomla.local)  
- ОС: Ubuntu 22.04.5 LTS (Jammy Jellyfish)  
- IP: 192.168.5.136  
- CPU: 2 vCPU  
- RAM: 2 GB  
- Disk: 25 GB  
- Назначение: CMS Joomla (nginx , phpfpm 8.3, postgresql 15)

Имя: zabbix.local  
- ОС: Ubuntu 22.04.5 LTS (Jammy Jellyfish)  
- IP: 192.168.5.141  
- CPU: 2 vCPU  
- RAM: 2 GB  
- Disk: 25 GB  
- Назначение: Система мониторинга Zabbix, система визуализации Grafana OSS

Имя: opensearch.local  
- ОС: Ubuntu 22.04.5 LTS (Jammy Jellyfish)  
- IP: 192.168.5.139  
- CPU: 2 vCPU  
- RAM: 4 GB  
- Disk: 25 GB  
- Назначение: Система централизованного логрования OpenSearch (Datapreper, Opensearch, Opensearch Dashboard)

## Мониторинг

Агентный мониторинг:
- ОС - официальный шаблон «Linux by Zabbix agent»
- Веб сервер NGINX - официальный шаблон «Nginx by Zabbix agent»
- СУБД PostgreSQL - официальный шаблон «PostgreSQL by Zabbix agent 2»
- Скриптовый интерпретатор PHP-FPM - официальный шаблон «PHP-FPM by Zabbix agent»

Безагентный мониторинг:
- Доступность сервера - самописный шаблон "Linux Server Availability"
- Доступность веб страницы - самописный шаблон «Website monitoring»
- SSL сертификат - официальный шаблон "Website certificate by Zabbix agent 2"

![image](https://github.com/user-attachments/assets/76b0599f-5904-4913-97a6-3796ec0936fc)

![image](https://github.com/user-attachments/assets/07c97f7e-e448-468d-baee-803887560330)


Оповещение:
- Telegram - встроенный тип отправки с помощью бота

![image](https://github.com/user-attachments/assets/55b72eeb-e541-4771-a231-3fe77d827793)

![image](https://github.com/user-attachments/assets/3ed31e22-f4ea-49ad-a51b-fc1583bda65b)

![image](https://github.com/user-attachments/assets/32453d57-0474-41cc-b271-835cbfbae457)



Визуализация:
- Grafana - официальный плагин «», подключен через API Zabbix с помощью токена. 
- Дашборды самописные Linux Server и Web site

![image](https://github.com/user-attachments/assets/4951c482-b53e-4b8c-8cf6-1bdd17704b35)

![image](https://github.com/user-attachments/assets/dc66dfd8-76d5-4b25-9941-1abd2590ffd6)



## Логирование

OpenSearch (Vector, OpenSearch, OpenSearch Dashboards):
Логи на сервере cms.local собираются с помощью Vector
Отправляются в Vector на сервере opensearch.local для обработки
Отправляются в OpenSearch на этом же сервере

![image](https://github.com/user-attachments/assets/d364dc46-9233-4227-ba21-b9f86031689c)

![image](https://github.com/user-attachments/assets/c191b43b-2360-489b-b4a8-803d69adca1a)

![image](https://github.com/user-attachments/assets/c513505f-7e98-4994-8a23-0f0c5439ca3b)

![image](https://github.com/user-attachments/assets/3e35e093-9382-4682-a70e-ecede6b05738)

![image](https://github.com/user-attachments/assets/14869722-db0a-402f-af71-3bc9915bdb85)


На дашборде созданы для примера две визуализации с кодами ошибок и количеством ошибок

![image](https://github.com/user-attachments/assets/a4d28315-b91e-49f8-b2b4-855d20a06424)
