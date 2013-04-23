---
layout: post
title: "Настраиваем сервера на Clodo через Opscode Chef"
description: "Инструкция по использованию chef для серверов на хостинге Clodo.ru"
category: howto
tags: [chef, clodo.ru, knife]
published: true
---

Данная статья описывает возможность автоматической настройки виртуальных серверов (на примере хостинга [clodo][]) с помощью [chef](http://www.opscode.com).

Заводим аккаунт в [opscode][]

Добавляем free хостинг на 5 серверов.

При создании необходимо скачать ключик валидотора и конфиг для [knife][] и поместить в `~/.chef/knife.rb`.
{% highlight ruby %}
current_dir = File.dirname(__FILE__)
log_level                :info
log_location             STDOUT
node_name                "cyberflow"
client_key               "#{current_dir}/cyberflow.pem"
validation_client_name   "clodo-test-validator"
validation_key           "#{current_dir}/clodo-test-validator.pem"
chef_server_url          "https://api.opscode.com/organizations/clodo-test"
cache_type               'BasicFile'
cache_options( :path => "#{ENV['HOME']}/.chef/checksums" )
cookbook_path            ["#{current_dir}/../cookbooks"]
{% endhighlight %}

Далее идём в [opscode][], заводим клиента, и ключик для клиента кладём так же в `~/.chef/`.
<!-- more -->
>  При создании пользователя помните, что его имя не должно совпадать с именем аккаунта в [opscode][]. Это приведёт к коллизии и ошибке в доступе к функциям API и панели управления chef сервером.

Теперь можно установить [knife][] и плагин к нему от [clodo][]:
{% highlight bash %}
$ curl -L https://www.opscode.com/chef/install.sh | sudo bash
$ sudo aptitude install libxml2-dev libxslt1-dev
$ sudo gem install knife-clodo
{% endhighlight %}

Определимся с целью. Для тестов допустим, что нам нужно устанавливать ВПС с [wordpress][] на борту. Для это в [opscode][] уже есть соответствующий [рецепт](https://github.com/opscode-cookbooks/wordpress). Но для его использования нам надо залить его(и ещё пару рецептов которые нужны по зависимостям) на наш chef-server. Делается это следующим образом:
{% highlight bash %}
$ mkdir ~/cookbook && cd ~/cookbook
$ git clone git://github.com/opscode-cookbooks/wordpress.git
$ git clone git://github.com/opscode-cookbooks/apache2.git
$ git clone git://github.com/opscode-cookbooks/mysql.git
$ git clone git://github.com/opscode-cookbooks/php.git
$ git clone git://github.com/opscode-cookbooks/openssl.git
$ git clone git://github.com/opscode-cookbooks/build-essential.git
$ knife cookbook upload openssl wordpress php mysql apache2 build-essential xml
{% endhighlight %}

После установки можно приступать к магии. Для начала добавим в конфиг [knife][] данные аккаунта [clodo][]:
{% highlight bash %}
$ cat >> ~/.chef/knife.rb << EOF
knife[:clodo_username] =         'clodo@user.name' 
knife[:clodo_api_key]   =        '12345678890qwertyasdfgh'
knife[:clodo_api_auth_url]      = 'api.clodo.ru'
EOF
{% endhighlight %}


Волшебство
{% highlight bash %}
knife clodo server create -r "recipe[apt]" -c ~/.chef/knife-clodo-test.rb \
--server-disk=5 --server-memory=256 --server-memory-max=512 -I 541 \
--node-name test1 --server-name opscode-test --template-file ~/.chef/chef-full.erb \
--no-ssl-verify-peer --clodo-api-auth-url api.mn.clodo.ru
{% endhighlight %}

[opscode]:	http://www.opscode.com/	      	     	    "Opscode" 
[knife]:	http://wiki.opscode.com/display/chef/Knife/ "Knife"
[clodo]:	http://clodo.ru/			    "Clodo.ru"
[wordpress]:    http://wordpress.org/ 			    "WordPress"
