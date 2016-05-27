---
layout: post
title: "Установка idoutils и icinga-web 1.5.2"
date: 2011-11-03 09:28
comments: true
categories: [monitoring]
tags: [apache2, debian, linux, icinga, icinga-web, ido2db, monitoring]
published: true
---

### Для чего это нужно?
В дефолтной инсталляции [icinga](http://www.icinga.org) пишет все данные в файлики. Но есть возможность перенаправлять эти данные в базу при помощи idoutils. В целом в базе будут храниться все события, perf-data и другие данные.

### Перед тем как начать
Подразумевается, что icinga 1.5.1 уже установлена и дальше будет только описание необходимых действия для установки idoutils и icinga-web 1.5.2. И то и другое требует базу данных. По этому убедитесь, что в вашем debian установлен соответствующий пакет. Если это не так, то установите его. Также необходимо наличии libdbi:

```
# apt-get install mysql-server mysql-client libdbi0 libdbi0-dev libdbd-mysql
```
Для установки icinga-web 1.5.2 необходимы следующие пакеты:
<!--more-->

```
# apt-get install apache2 build-essential libgd2-xpm-dev php5 php5-cli php-pear php5-xmlrpc php5-xsl php5-gd php5-ldap php5-mysql make
```

### Установка idoutils
В бэкпортах debiana есть необходимые пакеты для установки idoutils. Ставим:

```
# aptitude -t squeeze-backports install icinga-idoutils icinga-phpapi
```
В процессе установки пакет должен попытаться сконфигурировать базу данных. Вам будет необходимо ввести пароль от пользователя базы данных icinga.

Копируем пример конфига:

```
cp /etc/icinga/modules/idoutils.cfg-sample /etc/icinga/modules/idoutils.cfg
```
Правим конфиг:

``` diff
--- idoutils.cfg 2011-11-02 14:05:26.000000000 +0000
+++ idoutils.cfg-sample 2011-09-23 14:10:54.000000000 +0000
@@ -7,6 +7,6 @@
 define module{
         module_name     idomod
         module_type     neb
-        path            /usr/lib/icinga/idomod.o
+        path            /usr/sbin/idomod.o
         args            config_file=/etc/icinga/idomod.cfg
  }
```
Правим `/etc/default/icinga`:

``` bash
IDO2DB=yes
```
После этого нужно перезапустить сервисы *ido2bd* и *icinga*:

``` console
#  /etc/init.d/ido2db restart
#  /etc/init.d/icinga restart
```
Если всё настроено правильно, то в логах icinga должна появится запись:

```
...
Event broker module '/usr/lib/icinga/idomod.o' initialized successfully.
...
```

### Установка icinga-web 1.5.2
На данный момент этот веб-интерфей icinga не упакован в пакет для Debian. По этому ставить будем руками.
Для начала необходимо скачать [исходники](https://sourceforge.net/projects/icinga/files/icinga-web/1.5.2/icinga-web-1.5.2.tar.gz/download) с веб интерфейсом.
Скачиваем архив в `/usr/src/` и расспаковываем:

``` console
# tar xzvf icinga-web-1.5.2.tar.gz
```
Icinga-Web имеет некоторые конфигурационные опции, которые помогут в дальнейшей инсталляции:

```
# ./configure
--prefix=/usr/local/icinga-web
--with-web-user=www-data
--with-web-group=www-data
--with-web-apache-path=/etc/apache2/conf.d
--with-db-type=mysql
--with-db-host=localhost
--with-db-port=3306
--with-db-name=icinga_web
--with-db-user=icinga_web
--with-db-pass=icinga_web
--with-conf-folder=etc/conf.d
--with-log-folder=log
```
Имейте в виду, что вы настраиваете БД для Icinga-Web, а не для Icinga. Это разные БД.
Далее выполняем следующие команды:

``` console
# make install && make install-apache-config && make install-done
```

### Подготовка БД
Icinga-Web требует для работы базу данных. Вы можете использовать базу icinga, но рекомендуется создать отдельную БД icinga_web

``` console
# mysql -u root -p
mysql&gt; CREATE DATABASE icinga_web;

GRANT USAGE ON *.* TO 'icinga_web'@'localhost' IDENTIFIED BY 'icinga_web';
GRANT ALL PRIVILEGES ON icinga.* TO 'icinga'@'localhost' IDENTIFIED BY 'icinga';
GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, DROP, ALTER, INDEX ON icinga_web.* TO 'icinga_web'@'localhost';

quit
# make db-initialize
# /usr/local/icinga-web/bin/clearcache.sh
```

### Запуск
Для корректной работы веб-интерфейса необходимо включить модуль `rewrite` веб-сервера:

``` console
# a2enmod rewrite
```
Теперь нужно перечитать настройки *apache*:

``` console
# /etc/init.d/apache2 reload
```
Icinga-web по умолчанию доступна по адресу http://serverhostname/icinga-web/.Также по умолчанию авторизация происходит с помощью логина `root` и пароля `password`.
