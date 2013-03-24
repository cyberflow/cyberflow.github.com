---
layout: post
title: "Github pages, jekyll и custom plugins"
description: "Как можно использовать плагины для jekyll и хостится на github"
category: howto
tags: [jekyll, github]
---
Не секрет, что [jekyll][2] отличный движок для блогинга. Так же не секрет, что многие пользователи [jekyll][2] используют в качестве хостинга [github pages][3]. Однако, как всегда, есть нюансы. В случае с [github pages][3] это отсутствие поддержки custom plugins, коих для [jekyll][2] имеется в количестве. Есть масса вариантов хостить статику, но в этой статье речь пойдёт о том, как можно продолжать хоститься на [github pages][3] и использовать при этом плагины.

Собственно и в этом вопросе не обошлось без вариантов, но лично для себя я выбрал вариант, который предложил [Alexandre Rademaker][1]. Суть этого решения заключается в том, чтобы отказаться от генерации статики на стороне [github][4], а генерить её локально. Однако красота метода заключается в том, что при этом все исходные данные продолжают находиться под контролем git-a.

Теперь по сути:

Мы будем использовать branch source для хранения сырых данных и самой начинки [jekyll][2], тогда как в master бранче будет только статика, которая и будет раздаваться по средствам [github pages][3].

Далее предполагается, что у нас уже есть репозиторий на [github][4], где в мастер ветке лежит jekyll и сырые данные без статики. Теперь создаём новый branch:
{% highlight bash %}
$ git branch source
$ git push origin source
$ git checkout source
{% endhighlight %}

Теперь создаём что нам надо, добавляем плагины и т.п.
{% highlight bash %}
$ git status / git add / git commit
{% endhighlight %}

Запускаем [jekyll][2]:
{% highlight bash %}
$ jekyll
{% endhighlight %}

Всё готово для выкладки на [github pages][3]:
{% highlight bash %}
$ checkout master
$ cp -r _site/* . && rm -rf _site/ && touch .nojekyll
$ git status > git add > git commit
$ git push -all origin
{% endhighlight %}

В статье используются материалы с сайта:
[http://arademaker.github.com][1]

[1]:http://arademaker.github.com/blog/2011/12/01/github-pages-jekyll-plugins.html
[2]:https://github.com/mojombo/jekyll
[3]:http://pages.github.com/
[4]:http://github.com