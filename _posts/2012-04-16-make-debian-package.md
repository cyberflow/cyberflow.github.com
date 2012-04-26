---
layout: post
title: "Создание Debian пакета"
description: "Создаём пакет для Debian"
category: howto
tags: [debian]
---
{% include JB/setup %}
### Введение ###
Это даже не статья, а небольшая заметка. В ней будет описан процесс создания собственного deb пакета с нуля. 

### Установка необходимых пакетов ###
Для работы нам понадобятся следующие пакеты:
{% highlight bash %}
aptitude install dh-make
{% endhighlight %}

### Создание директории пакета ###
Далее создадим директорию, в которой будет всё необходимо для создания нашего пакета. Можно сразу создать директорию с версией вашего пакета или передавать имя и версию пакета опцией *-p* для *dh_make*:
{% highlight bash %}
mkdir package-0.1
cd package-0.1
export DEBFULLNAME="Name Surname"
dh_make --createorig -e myemain@domain.org -s -c GPL
{% endhighlight %}