---
title: "Миграция старых indices elasticsearch на другую ноду"
layout: post
comments: true
date: 2016-06-03 12:41
tags: [elasticsearch]
categories: [elasticsearch]
---

Я использую `elasticsearch` для централизованного хранения логов с серверов. Логов хранится много и каждый новый индекс ест ресурсы на сервере (статью об оптимизации используемых ресурсов для одной ноды можно посмотреть [здесь](/linux/monitoring/2016/05/12/optimization-standalone-elasticsearch-in-elk-stack.html)). Однако это решение не спасает при хранении логов за очень большие периоды, в связи с чем я решил рассмотреть вариант хранения старых логов на отдельном сервере.

Т.к. речь идет именно о логах, то шансы изменения данных из прошлого стремятся к нулю. Это позволяет нам не думать о необходимости записи в старые индексы и просто хранить их на сервере (например более дешевом с медленными дисками).
Генеральная идея состоит в том, чтобы добавить дополнительную ноду данных в кластер и распределять индексы (indices) в зависимости от времени их создания.

<!--more-->

## Настройка кластера

### Настройка мастер ноды
Пропуская процесс установки `elasticsearch` укажу лишь необходимые настройки:

>elasticsearch.yml
{:.filename}

``` yml
node.master: true
node.data: true
node.node_type: hot
...
discovery.zen.ping.multicast.enabled: false
discovery.zen.ping.unicast.hosts: ["10.0.0.2"]
```

Нода будет иметь тэг `hot` и хранит индексы для быстрого доступа.


### Настройка ноды дынных

>elasticsearch.yml
{:.filename}

``` yml
node.master: false
node.data: true
node.node_type: warm
...
discovery.zen.ping.multicast.enabled: false
discovery.zen.ping.unicast.hosts: ["10.0.0.1"]
```

Нода будет иметь тэг `warm` и мы будем перемещать на нее устаревшие indices.

### Перемещение
После подготовки нод надо перезапустить `elasticsearch`.

Убедиться, что ноды теперь в кластере можно командой:

``` console
$ curl http://localhost:9200/_nodes/process?pretty
```

Следующая команда поставит аттрибут аллокации для всех индексов старше 7 дней:

``` console
$ curator --logfile /var/log/curator.log --loglevel INFO --logformat default --master-only --host localhost --port 9200 allocation --rule node_type=warm indices --time-unit days --older-than 7 --timestring '%Y.%m.%d'
```

Теперь можно добавить команду curator-а, описанную выше, в cron сервера и перемещение будет происходить автоматически.