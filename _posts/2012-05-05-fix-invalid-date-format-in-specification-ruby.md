---
layout: post
title: "Fix invalid date format in specification Ruby"
description: "Боримся с ошибкой в Ruby"
category: howto
tags: [ruby]
---
{% include JB/setup %}
После обновления Ubuntu с 11.04 до версии 11.10 столкнулся с постоянной ошибкой при работе с ruby вида:
{% highlight bash %}
Invalid gemspec in [/var/lib/gems/1.8/specifications/directory_watcher-1.4.1.gemspec]: invalid date format in specification: "2011-08-30 00:00:00.000000000Z"
{% endhighlight %}

Версия ruby 1.8.7

Для лечения этой проблемы необходимо поправить строчку в проблемной спеке, заменив строчку:

*s.date = %q{2011-05-21 00:00:00.000000000Z}*

на

*s.date = %q{2011-05-21}*

Для конкретно моего случая надо править */var/lib/gems/1.8/specifications/directory_watcher-1.4.1.gemspec*


WIN!