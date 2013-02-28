---
layout: post
title: "Настраиваем сервера на Clodo через Opscode Chef"
description: "Инструкция по использованию chef для серверов на хостинге Clodo.ru"
category: howto
tags: [chef, clodo.ru, knife]
published: false
---
{% include JB/setup %}

> Дополнить описанием сути статьи. Что, Зачем и Почему?

Заводим аккаунт в [opscode][]

Добавляем free хостинг на 5 серверов.

При создании необходимо скачать ключик валидотора и конфиг для [knife][] и поместить в ~/.chef/knife.rb.
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

Далее идём в chef, заводим клиента, и ключик для клиента кладём так же в ~/.chef/.
>  При создании пользователя помните, что его имя не должно совпадать с именем аккаунта в [opscode][]. Это приведёт к коллизии и ошибке в доступе к функциям API и панели управления chef сервером.

Теперь можно установить [knife][] и плагин к нему от [clodo][]:
{% highlight bash %}
всякие gem install бла-бла-бла. Надо воспроизвести на другом компе или виртуалке.
{% endhighlight %}

После установки можно приступать к магии. Для начала добавим в конфиг [knife][] данные аккаунта [clodo][]:
{% highlight bash %}
$ cat >> ~/.chef/knife.rb << EOF
knife[:clodo_username] =         'clodo@user.name' 
knife[:clodo_api_key]   =        '12345678890qwertyasdfgh'
knife[:clodo_api_auth_url]      = 'api.clodo.ru'
EOF
{% endhighlight %}

Теперь можно приступать к самому основному - выбору ОС:
{% highlight bash %}
$ knife clodo image list
{% endhighlight %}


[opscode]:	http://www.opscode.com/	      	     	    "Opscode" 
[knife]:	http://wiki.opscode.com/display/chef/Knife/ "Knife"
[clodo]:	http://clodo.ru/			    "Clodo.ru"