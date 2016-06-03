---
layout: post
title: "Репликация таблиц базы chef-server "
date: 2013-08-09 11:16
comments: true
categories: [chef, ha, linux]
tags: [debian, postgresql, skytools, replication, chef]
published: true
---

### Intro
В данной статье я расскажу как настроить два сервера [chef-server][1] версии 11 с репликацией части таблиц базы postgresql для обеспечения отказоустойчивости. Основная цель репликации это хранения данных о клиентах и нодах для прозрачного доступа нод к любому из [chef][1] серверов и возможности работы без перерегистрации. Предлагаемое [opscode][1] решение репликации данных через DRBD показалось мне не самым удобным и, главное, надёжным. По сему было принято решение искать альтернативные пути. Так как реплицировать все данные из всех таблиц нет необходимости, то было решено посмотреть в сторону [skytools](http://wiki.postgresql.org/wiki/Skytools) и [londiste](http://wiki.postgresql.org/wiki/Skytools#Londiste), которые позволяют реплицировать только определенные таблицы БД.

Однако не обошлось без сюрпризов. [Opscode][1] распространяет shef-server 11 по средствам уже собранных omnibus пакетов, в котором свой postgres 9.2. Этот постгрес собран урезанным и не работает с skytools. В итоге в этой статье будет описано как поставить chef-server из omnibus пакета на дистрибутивный postgres 9.2 и прикрутить репликацию через londiste.

<!--more-->

Мы будем устанавливать всё на [debian 7](http://debian.org). Будет 2 сервера: `chef-master` и `chef-slave` соответственно.

### Добавление репозитория; Установка postgresql 9.2
К сожалению в дистрибутивном репозитории debian нет postgresql 9.2. По этому будем использовать не родной.
Добавляем репозиторий для установки postgres-a и skytools и устанавливаем на обоих серверах:
{% highlight console %}
root@chef-master:~# wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
root@chef-master:~# echo "deb http://apt.postgresql.org/pub/repos/apt/ wheezy-pgdg main" > /etc/apt/sources.list.d/postgresql.list
root@chef-master:~# aptitude update
root@chef-master:~# aptitude install skytools-modules-9.2 skytools3 skytools3-ticker skytools3-walmgr
{% endhighlight %}
<!-- more -->

### Установка chef-server на мастер
Теперь можно начинать устанавливать chef-server:
{% highlight console %}
root@chef-master:~# wget https://opscode-omnibus-packages.s3.amazonaws.com/ubuntu/11.04/x86_64/chef-server_11.0.4-1.ubuntu.11.04_amd64.deb
root@chef-master:~# dpkg -i chef-server_11.0.4-1.ubuntu.11.04_amd64.deb
{% endhighlight %}

Перед запуском `chef-server-ctl reconfigure` нужно немного поправить chef-solo рецепт для postgresql из omnibus пакета. Надо заменить исходный рецепт `/opt/chef-server/embedded/cookbooks/chef-server/recipes/postgresql.rb` на не исходный - [postgresql.rb](/static/files/postgresql.rb)

Запускаем `sudo chef-server-ctl reconfigure`

Собственно после этих действий получаем chef-server на дистрибутивном postgresql 9.2, что уже не плохо. Далее надо повторить эту же операцию на втором сервере(chef-slave).

### Установка chef-server; Подготовка к репликации
Процедура идентична за исключение нескольких дополнительных шагов.
Необходимо настроить postgres на IP адресе, который позволит мастеру производить репликацию и добавить в `pg_hba.conf` разрешение для мастера.

```
root@chef-slave:~# sed -i "s/#listen_addresses = 'localhost'/listen_addresses = '*'/" /etc/postgresql/9.2/mainpostgresql.conf
root@chef-slave:~# echo "host all all <chef-master-ip>/32 trust" >> /etc/postgresql/9.2/main/pg_hba.conf
root@chef-slave:~# service postgresql restart
```
Далее надо почистить текущую базу от данных:

```
root@chef-slave:~# su postgres -c "psql -d 'opscode_chef' -c \"delete from clients; \""
root@chef-slave:~# su postgres -c "psql -d 'opscode_chef' -c \"delete from osc_users; \""
```
Ещё надо подменить содержимое директории `/etc/chef-server` на сервер `chef-slave` на содержимое той же директории с сервер `chef-master`. Там хранятся приватные ключи от системных пользователей chef-server-a.

Теперь всё готово к запуску репликации.

### Настройка репликации
Для начала нужно добавить londiste и pgq в нашу базу. Для этого воспользуемся утилитой qadmin, которая была добавлена в skytools3:

{% highlight console %}
root@chef-master:~# su postgres -c "qadmin -h localhost -U postgres -d opscode_chef -c 'install londiste'"
INSTALL
{% endhighlight %}

Пишем конфиги для мастера и слейва(оба конфига лежат на chef-master):

>/etc/skytools/replication_master.ini
{:.filename}

{% highlight cfg %}
[londiste3]
job_name = replication_src
db = host=master-db dbname=opscode_chef user=postgres
queue_name = replica
logfile = /var/log/skytools/replica_src.log
pidfile = /var/run/skytools/replica_src.pid
{% endhighlight %}

Добавляем мастер:
{% highlight console %}
root@chef-master:~# londiste3 /etc/skytools/replication_master.ini create-root master-server 'dbname=opscode_chef user=postgres'
{% endhighlight nets%}

Конфиг для слейва:

>/etc/skytools/replication_slave.ini
{:.filename}

{% highlight cfg %}
[londiste3]
job_name = replication_dst
db = host=master-db dbname=opscode_chef user=postgres
queue_name = replica
logfile = /var/log/skytools/replica_dst.log
pidfile = /var/run/skytools/replica_dst.pid
{% endhighlight %}

Добавление слейва:
{% highlight console %}
root@chef-master:~# londiste3 /etc/skytools/replication_slave.ini create-leaf slave-server 'dbname=opscode_chef host=chef-slave user=postgres' --provider='host=localhost dbname=opscode_chef user=postgres'
{% endhighlight nets%}

Добавляем конфиг для pgq демона:

>/etc/skytools/pgqd.ini
{:.filename}

{% highlight cgf %}
[pgqd]
logfile = /var/log/skytools/pgqd.log
pidfile = /var/run/skytools/pgqd.pid
base_connstr = host=localhost user=postgres
initial_database = opscode_chef
{% endhighlight %}

Теперь запускаем skytools и проверяем, что всё настроено верно:

```
root@chef-master:~# service skytools start
root@chef-master:~# londiste3 /etc/skytools/replication_master.ini status
Queue: replica   Local node: master-server

master-server (root)
  |                           Tables: 0/0/0
  |                           Lag: 12s, Tick: 32
  +--slave-server (leaf)
                              Tables: 0/0/0
                              Lag: 12s, Tick: 32
```
Видим, что у нас есть 2 сервера но пока ничего не реплицируется.
Добавляем необходимые таблицы:

```
root@chef-master:~# londiste3 /etc/skytools/replication_master.ini add-table clients
root@chef-master:~# londiste3 /etc/skytools/replication_master.ini add-table nodes
root@chef-master:~# londiste3 /etc/skytools/replication_master.ini add-table osc_users
root@chef-master:~# londiste3 /etc/skytools/replication_slave.ini add-table nodes
root@chef-master:~# londiste3 /etc/skytools/replication_slave.ini add-table clients
root@chef-master:~# londiste3 /etc/skytools/replication_slave.ini add-table osc_users
root@chef-master:~# londiste3 /etc/skytools/replication_master.ini status
```
Спустя несколько минут мы можем увидеть, что репликация завершилась:

```
Queue: replica   Local node: master-server

master-server (root)
  |                           Tables: 3/0/0
  |                           Lag: 14s, Tick: 52
  +--slave-server (leaf)
                              Tables: 3/0/0
                              Lag: 14s, Tick: 52
```

[1]: http://www.opscode.com "Opscode"
