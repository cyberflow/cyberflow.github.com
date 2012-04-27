---
layout: post
title: "Создание chef рецептов"
description: "Создаём рецепт для chef"
category: notice
tags: [chef]
---
{% include JB/setup %}
Для использования необходим настроенный knif.

### Создаём шаблон при помощи knife ###
Начнём с создания шаблона. Для этого выполним команду:
{% highlight bash %}
knife cookbook create <recipe_name> -e email@domain.ru
{% endhighlight %}