---
layout: post
title: "Hyperdb to Wordpress"
date: 2013-03-22 18:02
comments: true
categories: [clusters, linux, ha]
tags: [availability, cluster, db, hyperdb, mysql, scaling, wordpress howto]
---

[Hyperdb](http://wordpress.org/extend/plugins/hyperdb/) - это замена стандартоного класса `wpdb`, позволяющая использовать несколько баз данных.
В данной статье я опишу настройку hyperdb для распределенной инфраструктуры, состоящей из двух frontend серверов и двух серверов баз данных на базе дистрибутивов Debian 6, в качестве основной задачи ставилось не только распределение запросов на чтение между серверами БД, но и отслеживания отставаний слэйв сервера и, в случае отставания, чтение актуальных данных с мастера.

Настройка самих фронтендов оставим за кадром, т.к. это выходит за рамки данной статьи. Что касается серверов БД, то мы будем использовать мастер-слэйв репликацию и mk-heartbeat для отслеживания lag-ов на слэйве. Сервера геораспределены и каждый сервер БД располагается в отдельном ДЦ в паре с фронтендом. На настройке баз я остановлючь подробнее:

Настройка репликации DB:
------------------------

Здесь не описано ничего принципиально нового. Используется стандартная [репликация Мастер-Слэйв](http://dev.mysql.com/doc/refman/5.1/en/replication.html) средствами MySql.

<!--more-->

Все, описанные ниже команды выполнялись под пользователем `root`, что является не безопасным. Будьте осторожны, при работе под суперпользователем.

Устанавливаем сервер баз данных и пакет с `mk-hertbeat`:
{% highlight console%}
# aptitude install mysql-server maatkit
{% endhighlight %}

Далее, используя документацию создаём базу wordpress на db1.
Этот сервер будет мастером. В файл `/etc/mysql/my.cnf` в секции [mysqld] необходимо добавить:
{% highlight cfg %}
init-connect='SET NAMES utf8;'
server-id                = 1
log_bin                  = /var/log/mysql/mysql-bin
expire_logs_days         = 5
max_binlog_size          = 100M
binlog_do_db             = wordpress
{% endhighlight %}
Далее надо перезапустить mysql:
{% highlight console %}
# service mysql restart
{% endhighlight %}
Теперь блокируем базу на запись, чтобы избежать рассинхронизации до окончания настройки репликации:
{% highlight console %}
mysql> FLUSH TABLES WITH READ LOCK;
mysql> SET GLOBAL read_only = ON;
{% endhighlight %}
После этого делаем дамп базы wordpress и копируем на слэйв сервер БД (db2).

На db1 создаём пользователя для репликации:
{% highlight console %}
mysql> GRANT REPLICATION SLAVE, REPLICATION CLIENT ON wordpress.* TO repl@'%' IDENTIFIED BY 'replicationsuperpassword' with grant option;
mysql> SHOW MASTER STATUS;
{% endhighlight %}
Из вывода последней команды нам понадобится `Position` и `File` для конфигурации репликации на втором сервере БД.
На втором сервере(db2) в `/etc/mysql/my.cnf` в секции [mysqld] необходимо добавить:
{% highlight cfg %}
server-id                = 2
{% endhighlight %}
На db2 выполняем следующие команды подставляя значения из вывода `SHOW MASTER STATUS` c db1:
{% highlight console %}
mysql> STOP SLAVE;
mysql> RESET SLAVE;
mysql> CHANGE MASTER TO MASTER_HOST='db1.host.ru',
MASTER_PORT=3306,
MASTER_USER='repl',
MASTER_PASSWORD='replicationsuperpassword',
MASTER_LOG_FILE='mysql-bin.000001',
MASTER_LOG_POS=106,
MASTER_CONNECT_RETRY=10;
mysql> START SLAVE;
{% endhighlight %}
Проверяем, что репликация работает:
{% highlight console %}
mysql> SHOW SLAVE STATUS\G
{% endhighlight %}
В выводе должно присутствовать:
{% highlight cfg %}
Slave_IO_Running: Yes
Slave_SQL_Running: Yes
{% endhighlight %}
Если видим эти строки, то можно включать мастер БД(db1) на чтение:
{% highlight console %}
mysql> SET GLOBAL read_only = OFF;
mysql> UNLOCK TABLES;
{% endhighlight %}
Переходим к настройке Hyperdb.

Установка и настройка Hyperdb:
------------------------------
Сама установка класса проста и не затейлива.
Скачиваем архивчик с классами с официального сайта [Wordpress](http://wordpress.org/extend/plugins/hyperdb/). Распаковываем на всех web-серверах. В архиве всего 3 файла: `db-config.php`, `db.php`, `readme.txt`.

Правим конфигурационный файл для web1 (он находится в том же сегменте, что и db1, т.е. рядом с мастером):
{% highlight ruby %}
//описываем параметры подключения к мастеру.
//Т.к. он ближе всех, то и читать будем с него, ставим приоритет чтения 1.
$wpdb->add_database(array(
 'host'     => 'db1.hostname.ru',
 'user'     => 'wordpress',
 'password' => 'wordpressdbpass',
 'name'     => 'wordpress',
 'write'    => 1,
 'read'     => 1,
));

// добавляем слэйв сервер. Т.к. он в другом сегменте, то приоритет на чтение ниже.
$wpdb->add_database(array(
 'host'     => 'db2.hostname.ru',
 'user'     => 'wordpress',
 'password' => 'wordpressdbpass',
 'name'     => 'wordpress',
 'write'    => 0,
 'read'     => 2,
));
{% endhighlight %}
Такая конфигурация позволит читать с ближайшего сервера или со слейва, если мастер будет не доступен.
Конфиг для web2:
{% highlight ruby %}
//описываем параметры подключения к мастеру.
//Пишем мы всегда и только на мастер, читать с него будем только в случае отказа слэйва
//ставим приоритет чтения 2.
$wpdb->add_database(array(
 'host'     => 'db1.hostname.ru',
 'user'     => 'wordpress',
 'password' => 'wordpressdbpass',
 'name'     => 'wordpress',
 'write'    => 1,
 'read'    => 2,
));

// добавляем слэйв сервер. Т.к. он рядом, то приоритет на чтение 1.
$wpdb->add_database(array(
 'host'     => 'db2.hostname.ru',
 'user'     => 'wordpress',
 'password' => 'wordpressdbpass',
 'name'     => 'wordpress',
 'write'    => 0,
 'read'     => 1,
 'lag_threshold' => 60, //определяем допустимое время отставания слэйва в секундах
));
{% endhighlight %}
Теперь, если скопировать конфиги в корень wordpress, а `db.php` в директорию `/wp-content/`, то мы уже начнём работать с БД по распределенной схеме.

Но цель данной статьи не просто распределять запросы на чтение по разным БД, но и реагировать на отставание слэйв-сервера, чтобы отдавать посетителям максимально актуальные данные.

Настройка обнаружения отсутствия слейва для hyperdb:
----------------------------------------------------

Первое что хочу отметить, что настраивать обнаружение лагов мы будем только для web2, т.к. web1 ходит читать в мастер, а в слэйв только по крайней необходимости и в таком случае лучше читать хоть что-то, чем вообще не вернуть никаких данных, хоть бы и не актуальных.

В конфиге `db-config.php` уже есть пример функций, позволяющих отслеживать лаги слэйва. Я лишь немного опишу процесс и как настроить сервера БД, чтобы эти функции работали и работали правильно. Для наглядности сразу приведу эти функции, которые нам надо будет добавить в конец файла `db-config.php` на web2:

>db-config\.php
{:.filename}

{% highlight php %}
$wpdb->lag_cache_ttl = 30;
$wpdb->shmem_key = ftok( __FILE__, "Y" );
$wpdb->shmem_size = 128 * 1024;

$wpdb->add_callback( 'get_lag_cache', 'get_lag_cache' );
$wpdb->add_callback( 'get_lag',       'get_lag' );

function get_lag_cache( $wpdb ) {
 $segment = shm_attach( $wpdb->shmem_key, $wpdb->shmem_size, 0600 );
 $lag_data = @shm_get_var( $segment, 0 );
 shm_detach( $segment );

 if ( !is_array( $lag_data ) || !is_array( $lag_data[ $wpdb->lag_cache_key ] ) )
  return false;

 if ( $wpdb->lag_cache_ttl < time() - $lag_data[ $wpdb->lag_cache_key ][ 'timestamp' ] )
  return false;

 return $lag_data[ $wpdb->lag_cache_key ][ 'lag' ];
}

function get_lag( $wpdb ) {
 $dbh = $wpdb->dbhs[ $wpdb->dbhname ];

 if ( !mysql_select_db( 'wordpress', $dbh ) ) // вот здесь мы поменяем имя базы на wordpress. Остальное как в примере.
  return false;

 $result = mysql_query( "SELECT UNIX_TIMESTAMP() - UNIX_TIMESTAMP(ts) AS lag FROM heartbeat LIMIT 1", $dbh );

 if ( !$result || false === $row = mysql_fetch_assoc( $result ) )
  return false;

 // Cache the result in shared memory with timestamp
 $sem_id = sem_get( $wpdb->shmem_key, 1, 0600, 1 ) ;
 sem_acquire( $sem_id );
 $segment = shm_attach( $wpdb->shmem_key, $wpdb->shmem_size, 0600 );
 $lag_data = @shm_get_var( $segment, 0 );

 if ( !is_array( $lag_data ) )
  $lag_data = array();

 $lag_data[ $wpdb->lag_cache_key ] = array( 'timestamp' => time(), 'lag' => $row[ 'lag' ] );
 shm_put_var( $segment, 0, $lag_data );
 shm_detach( $segment );
 sem_release( $sem_id );

 return $row[ 'lag' ];
}
{% endhighlight %}

Итак, как работает отслеживание:

*   Перед тем, как устанавливать соединение с базой, мы смотрим нет ли у нас данных в кэшэ о статусе её отставания.
    Время жизни кэша задаётся через `$wpdb->lag_cache_ttl`. Использования кэша позволяет нам не генерировать дополнительных запросов в базу без необходимости.
*   Если данных нет или кэш протух, проверяем отставание с помощью запроса к табличке heartbeat. Получаем время отставания и записываем в кэш.
*   Первая или вторая функция вернёт нам время отставания слэйва. Оно сравнивается с `lag_threshold`, которое определяется при добавлении сервера в конфиге.
*   Если время отставания не превышает `lag_threshold`, то читаем со слэйва, в противном случае закрываем все соединения с этой базой и читаем с мастера.

Конфиг поправили, как работает поняли. Осталось настроить `mk-heartbeat` для мониторинга отставания.

Настройка mk-heartbeat
----------------------

[mk-heartbeat](http://www.maatkit.org/doc/mk-heartbeat.html) - утилита, входящая в пакет `maatkit`, для мониторинга лагов репликации баз данных. В начале статьи мы уже установили нужный пакет на мастер(db1).

Два слова о том, как происходит мониторинг:

*   Демон, запущенный на мастере, раз в дельту времени пишет timestamp в табличку heartbeat в реплицируемой базе данных.
*   Средствами репликации эти данные попадают на слейв.
*   Функция get_lag, в hyperdb, сравнивает текущий timestamp с записью в табличке, получая время расхождения репликации.

Запускаем демон. Он автоматически создаст нужную таблицу:
{% highlight console %}
# mk-heartbeat --daemonize --create-table --table heartbeat -D wordpress --interval 27 --update
{% endhighlight %}
Эта команда запустит демон, который создаст таблицу heartbeat в базе wordpress и будет записывать timestamp раз в 27 секунд (этого достаточно для проверки отставания не более минуты).

При желании можно обернуть запуск демона в инит-скрипт и запускать при старте сервера.

Собственно теперь мы отслеживаем лаги репликации и отдаём пользователям наисвежайшие данные из баз.