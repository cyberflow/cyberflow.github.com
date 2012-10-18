---
layout: post
title: "Установка chef solo на debian 6 squeeze"
description: ""
category: howto
tags: [chef, chef-solo, debian, squeeze]
---
{% include JB/setup %}
Для начала установим необходимые пакеты для установки и работы cfeh-solo.

{% highlight bash %}
apt-get install sudo wget lsb-release
{% endhighlight %}

Далее добавляем репозиторий opscode в списки репозиториев командой:
{% highlight bash %}
echo "deb http://apt.opscode.com/ `lsb_release -cs`-0.10 main" | sudo tee /etc/apt/sources.list.d/opscode.list
{% endhighlight %}

Теперь необходимо добавить ключи к репозиторию:
{% highlight bash %}
sudo mkdir -p /etc/apt/trusted.gpg.d
gpg --keyserver keys.gnupg.net --recv-keys 83EF826A
gpg --export packages@opscode.com | sudo tee /etc/apt/trusted.gpg.d/opscode-keyring.gpg > /dev/null
{% endhighlight %}

Обновим информацию о пакетах с учётом добавленного репозитория и установим opscode-keyring:
{% highlight bash %}
sudo apt-get update && sudo apt-get install opscode-keyring
{% endhighlight %}

Устанавливаем chef:
{% highlight bash %}
sudo apt-get install chef
{% endhighlight %}

При установке будет задан вопрос о пути к серверу chef, т.к. мы делаем установку для chef-solo, то указываем там "none".