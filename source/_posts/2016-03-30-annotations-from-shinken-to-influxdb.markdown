---
layout: post
title: "annotations-from-shinken-to-influxdb"
date: 2016-03-30 15:58
comments: true
tags: [monitoring, linux, grafana, influxdb, shinken]
categories: [notes, howto, linux, monitoring]
published: false
---

Собирать метрики с серверов дело полезное, но при анализе графиков часто необходимо учитывать события, которые происходили с сервером на протяжении его работы. Создатели [Grafana](http://grafana.org) подумали и об этом и добавили `Annotations`, которые позволяют отображать mark points на графиках.

Собственно эта заметка о том, как добавить события из мониторинга (shinken) на графики в Grafana.

[картинка для привлечения внимания]
<!--more-->
### Настройка Influxdb
Эти настройки можно произвести по разному, но мне больше нравится делать это из консоли. Ничего сложного в этом пункте нет. Надо добавить базу и пользователя:
```
~# influx
create database shinken
create user shinken with password 'secret'
GRANT ALL ON shinken to shinken
```

### Настройка Shinken
Настройка `shinken` заключается в подключении и настройке модуля [mod-influxdb](http://shinken.io/package/mod-influxdb). Помимо записи всех событий в базу этот модуль будет записывать всю `performance data` от сервисов, что, несомненно, замечательно.

Установку модуля можно осуществить силами самого `shinken`:
```
~# shinken install mod-influxdb
```
Обнако этого не достаточно и необходимо установить модуль `python` для работы `mod-influxdb`:
```
~# pip install influxdb
```
Конфигурация модуля mod-influx:
``` bash /etc/shinken/modules/influxdb.cfg
## Module:      mod-influxdb
## Loaded by:   Broker
# Export host and service performance data and events to InfluxDB.
# InfluxDB is an open-source distributed time series database with no external
# dependencies. http://influxdb.com/
define module {
    module_name     influxdb
    module_type     influxdb_perfdata
    host            127.0.0.1
    port            8086
    user            shinken
    password        secret
    database        shinken
    #use_udp        1 ; default value is 0, 1 to use udp
    #udp_port        4444
    #tick_limit     300 ; Default value 300
}
```
Конфигурация брокера:
``` bash /etc/shinken/broker/broker.cfg
define broker {
    broker_name broker
    address shinken.com
    port 7772
    modules influxdb
    check_interval 60
    data_timeout 120
    max_check_attempts 3
    spare 0
    timeout 3
    weight 1
}
```
После настройки перезапустить `shinken`