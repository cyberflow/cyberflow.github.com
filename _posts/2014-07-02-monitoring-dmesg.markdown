---
layout: post
title: "Мониторинг dmesg"
date: 2014-07-02 09:58
comments: true
tags: [monitoring, dmesg, oom, logs, logstash, netconsole, linux, shinken]
categories: [monitoring, linux]
published: true
---

### Задача
Необходимо организовать централизованный сбор логов, в частности сообщений ядра `dmesg` и реакции на них со стороны событийного мониторинга (например *nagios*).

### Передача сообщений ядра на сервер логов
Для передачи сообщения ядра можно использовать модуль `netconsole`. Он позволяет передавать сообщения журнала ядра (dmesg) на удаленный компьютер по сети, без участия пространства пользователя (например, syslogd).
Для сбора данных можно использовать, например [logstash](http://logstash.net/).

Netconsole может быть встроен в ядро, так и загружен как модуль. Я использую загрузка модуля после загрузки системы, чтобы гарантировать, что сеть уже работает  настроена.

```
# dmesg -n 7
# modprobe netconsole netconsole="6665@10.13.77.99/eth1,6666@10.13.77.1/d4:ae:52:cf:33:96"
```
Первая команда устанавливает уровень логирования для `dmesg`.
Вторая загружает модуль `netconsole` с параметрами, где:
<!--more-->

- 6665 - порт отправки сообщений
- 10.13.77.99 - ip адрес хоста, с которого отправляются сообщения (текущий сервер)
-  eth1 - интерфейс с которого отправляются сообщения
-  6666 - порт, на который принимаются сообщения
-  10.13.77.1 - ip адрес хоста, на который принимаются сообщения
-  MAC - mac адрес интерфейса, на который принимаются сообщения

Установка `logstash` не является темой этой статьи, поэтому я буду показывать лишь части конфига необходимые для реализации задачи. Для получения сообщений от `netconsole` надо указать следующий кусок конфига в блоке `input`:

``` properties
input {
    udp {
        type => "netconsole"
        port => "6666"
        host => "10.13.77.1" }
    }
```

где `host` - это ip адрес интерфейса на который будут присылаться сообщения ядра.

Теперь все сообщения ядра отправляются в `logstash`.

### Оповещения при критических сообщениях ядра
В моём случае я хочу отслеживать наличие OOM сообщений и рассылать оповещения об этом. `Logstash` может отправлять письма с оповещениями сам, но мне интересно делать это событийным мониторингом.

В качестве событийного мониторинга я использую [shinken](www.shinken-monitoring.org). Для реализации оповещения админов, по моему, лучше использовать сервисные чеки `is_volatile`. Эта опция позволяет отправлять оповещения на каждый статус non-OK даже если он не менялся. Подробнее об этой опции можно почитать [тут](https://shinken.readthedocs.org/en/latest/07_advanced/volatileservices.html).

Чтобы не повторяться в описании каждого сервиса напишем template для volatile сервисов:

```
define service {
  name                          volatile-service
  check_command                 dummy_ok
  max_check_attempts            1
  check_interval                5
  retry_interval                2
  active_checks_enabled         0
  passive_checks_enabled        1
  check_period                  24x7
  check_freshness               1
  freshness_threshold           300
  flap_detection_enabled        0
  notification_interval         60
  notification_period           24x7
  notification_options          u,c,w
  notifications_enabled         1
  contact_groups                sysadmin
  register                      0
}
```
В данном описании мы определяем пассивные сервисы, которые будут автоматически сбрасываться в статус ОК через 5 минут после изменения статуса. Так же важно в опции `max_check_attempts` указать 1, чтобы сервис сразу после первого сообщения переходил в `hard-state`.

Теперь можно описать сам сервис используя вышеописанный template:

```
define service {
  hostgroup_name                linux
  service_description           log_event
  servicegroups                 logs
  initial_state                 o
  register                      1
  use                           volatile-service
}
```

С настройкой событийного мониторинга закончили. Переходим к настройке `logstash`.

Т.к. все сообщения ядра приходят от IP адреса сервера, который его сгенерил, то первое что я рекомендую сделать это настроить для серверов обратную зону, что позволит упростить и дальнейшую работу с логами и передачу сообщений в *shinken*. Если обратная зона есть, то можно добавить в блок *filter* следующий кусок конфига:

``` properties
filter {
    ...
    dns {
            'action' => "replace"
            'reverse' => ["host"]
            'type' => "netconsole"
        }
    ...
}
```
Теперь в поле `host` вместо IP адреса сервера будет подставляться последний уровень домена хоста (т.е. если полный домен myserver.my.domain.example.com, то в поле попадёт только myserver). Это упростит жизнь в будущем.

Далее добавим фильтрацию сообщений, на появления которых хотим получать оповещения. Для примера возьмём строку сообщающую о приходи *oom* киллера и будем тагать записи такого типа:

``` properties
    filter {
        ...
        if [message] =~ /.+Out.of.memory.+Kill.process.+/ {
           noop {
                  'add_tag' => ["notify"]
              }
        }
        ...
    }
```
Осталось научить *logstash* отправлять сообщения с тэгом `notify`. Для отправки на сервер *Nagios* можно использовать уже готовые плагины *logstash* [nagios](http://logstash.net/docs/1.4.2/outputs/nagios) или [nagios_nsca](http://logstash.net/docs/1.4.2/outputs/nagios). Я использую *shinken* и хоть он и совместим с *Nagios* в большинстве случаев, в этом я предпочёл использовать API *shinken*. Для отправки сообщений я написал простенький скрипт на `bash`:

>log_event_sender.sh
{:.filename}

``` bash
#!/bin/bash
# WS_Arbiter config
USER=username;
PASS=password;
WS_IP=192.168.0.55;
WS_PORT=7760;

HOST_NAME=$1;
SERVICE=`echo $2 | sed -r -e 's/[[:space:]]/%20/g'`;
RCODE=2;
MESSAGE=`echo $3 | sed -r -e 's/[[:space:]]/%20/g'`;

DATA="time_stamp=$(date +%s)&host_name=$HOST_NAME&service_description=$SERVICE&return_code=$RCODE&output=$MESSAGE";
SEND=$(curl -s -u ${USER}:${PASS} -d $DATA http://${WS_IP}:${WS_PORT}/push_check_result)
```
Добавляем отправку сообщений с тэгом `notify` в конфиг *logstash*:

``` properties
output {
    exec {
       'tags' => ["notify"]
       'command' => "/usr/local/log_event_sender.sh %{host} log_event '%{message}'"
  }
}
```
Дальнейшее расширение очевидно. Если нужно добавить оповещение о сообщениях другого типа, то нужно просто отлавливать их в *logstash* и навешивать тэг `notify`.