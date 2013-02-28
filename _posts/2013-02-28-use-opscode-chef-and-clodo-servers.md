---
layout: post
title: "Настраиваем сервера на Clodo через Opscode Chef"
description: "инструкция по использованию chef для серверов на хостинге Clodo.ru"
category: howto
tags: [chef, clodo.ru, knife]
published: false
---
{% include JB/setup %}

Заводим аккаунт в [opscode][1]

Добавляем free хостинг на 5 серверов.

При создании необходимо скачать ключик валидотора и конфиг для knife и поместить в ~/.chef/.
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



[1]:http://www.opscode.com
