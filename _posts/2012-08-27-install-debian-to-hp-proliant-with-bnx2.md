---
layout: post
title: "Install debian to HP ProLiant with bnx2"
description: ""
category: [debian, linux]
tags: [debian, bnx2, fireware]
---
### Введение ###
При установки Debian 6 на HP ProLiant DL360 сталкнулся с тем, что в образе netinstall нет fireware для сетевой карточки Broadcom Corporation NetXtreme II. Собственно инсталлятор в курсе этого и предлагает поискать соответствующий fireware на внешнем носителе. В этой заметке я опишу как быстро создать *.img файл с нужными fireware, который в последствии можно смонтировать через ipmi-kvm и скормить инсталлятору.

### Поиск fireware ###
Собственно для debian 6 есть [пакет][1] со всем необходимым, и всё бы ничего, но вот без настроенной сетевой карты поставить этот пакет в систему не простая задача. Но т.к. в пакете есть всё необходимое, то качаем [сырцы][2] пакета, распаковываем и приступаем к созданию образа.

### Создание IMG файла ###
Создайм бланковый файл, который будет у нас образом флоппи-диска, и создаём в нём файловую систему:
{% highlight bash %}
$ dd bs=512 count=2880 if=/dev/zero of=imagefile.img
$ mkfs.msdos imagefile.img
{% endhighlight %}
Далее монтируем наш флоппи-образ:
{% highlight bash %}
$ sudo mkdir /media/floppy1/
$ sudo mount -o loop imagefile.img /media/floppy1/
{% endhighlight %}
Теперь кладём файлы, который просит инстолятор в /media/floppy1/, отмонтируем образ и скармливаем файл с образом инсталятору.

[1]:http://packages.debian.org/squeeze/firmware-bnx2
[2]:http://ftp.de.debian.org/debian/pool/non-free/f/firmware-nonfree/firmware-nonfree_0.28+squeeze1.tar.gz