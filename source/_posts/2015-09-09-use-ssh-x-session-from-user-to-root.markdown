---
layout: post
title: "Пробрасывание X сессии от пользователя к root"
date: 2015-09-09 15:24
comments: true
categories: [notes, howto, workaround, linux]
published: true
---

Сталкнулся с необходимостью запуска графической утилиты на удаленном сервере под пользователем root, но доступ для root по ssh закрыт.

В итоге нашел такой workaround:

Заходим на сервер под пользователем прокинув X сессию через ssh:
``` bash
~$ ssh -X user@hostname
```

Находим X сессию, потом свитчемся в root:
``` bash
~$ xauth list
hostname/unix:10  MIT-MAGIC-COOKIE-1  f714ef310193878cae851635b871d840
~$ sudo -s
```

Добавляем имеющуюся сессию пользователю root
``` bash
~# xauth add hostname/unix:10  MIT-MAGIC-COOKIE-1 f714ef310193878cae851635b871d840
```
Все! Можно запускать X приложение под root-ом.
