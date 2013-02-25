---
layout: post
title: "Установка node.js на Linux (deb base)"
description: "Краткая документация по установке node.js на Debian подобные дистрибутивы Linux"
category: howto
tags: [debian, nodejs, howto]
---
{% include JB/setup %}

Данная инструкция проверялась и работает на Debian 6.

Устанавливаем пакеты, необходимые для сборки и удовлетворения зависимостей:
{% highlight bash %}
sudo aptitude install g++ curl python libssl-dev apache2-utils git-core checkinstall
{% endhighlight %}

Клонируем репозиторий с [github][2]:
{% highlight bash %}
cd /usr/src && git clone git://github.com/ry/node.git
{% endhighlight %}

Далее запускаем конфигурацию:
{% highlight bash %}
cd node && ./configure
{% endhighlight %}

Теперь можно собрать пакет для нашей системы:
{% highlight bash %}
checkinstall --fstrans=no --install=no --pkgname=node.js --pkgversion "0.5.9" --default
{% endhighlight %}

После сборки пакета его можно поставить:
{% highlight bash %}
dpkg -i node.js_0.5.9-1_amd64.deb
{% endhighlight %}

В статье использовались материалы со следующих сайтов:

[How to install node js on linux][1]

[1]:http://oodavid.tumblr.com/post/15090798307/how-to-install-node-js-on-linux
[2]:http://github.com