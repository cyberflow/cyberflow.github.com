---
layout: post
title: "Redmine 2.x обнуление пароля admin-a"
description: "Как обнулить пароль от admin-a в redmine 2.x"
category: notice
tags: [redmine,ruby]
---
Для обнуления пароля от пользователя admin в redmine 2.x выполните следующую команду из консоли находясь в директории с redmine
{% highlight bash %}
ruby script/rails runner 'user = User.find(:first, :conditions => {:admin => true}) ; user.password, user.password_confirmation = "password"; user.save!' -e production
{% endhighlight %}