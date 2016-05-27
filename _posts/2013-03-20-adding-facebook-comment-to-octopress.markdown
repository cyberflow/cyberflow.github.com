---
layout: post
title: "Добавление Facebook комментариев к Octopress"
date: 2013-03-20 12:11
comments: true
categories: [jekyll]
tags: [jekyll, facebook, comments]
---

В этой небольшой статья я опишу как добавить комментарии facebook к блогу на [octopress](http://octopres.org). В octopress есть поддержка комментариев с использованием [disqus](http://disqus.com/), но мне ближе facebook.

Для начала необходимо [зарегистрировать приложение на facebook](https://developers.facebook.com/apps) для своего блога. Когда регистрация пройдена facebook  должен выдать app id. Теперь можно приступить к настройке. Добавим facebook app id и параметры отображения комментариев в файл конфигурации `_config.yml`

>_config\.yml
{:.filename}

{% highlight yml %}
# Facebook comments
facebook:
  appid: 222612811167194
  num_post: 5
  width: 789
  colorscheme: light
{% endhighlight %}

Следующим шагом добавим facebook javascript API на нашу страницу. Для этого можно воспользоваться уже имеющимся в octopress функционалом facebook like. И так, открываем `siurce/_includes/facebook_like.html` и меняем строчку содержащую `js.src=` заменив в ней цифры на app id полученный от facebook.
<!--more-->
{% highlight js %}
js.src = "//connect.facebook.net/en_US/all.js#appId=222612811167194&xfbml=1";
{% endhighlight %}

Теперь переходим к добавлению комментариев на страницы блога. Создаём файл `source/_includes/post/facebook_comments.html` и добавляем в него следующее:

>facebook_comments\.html
{:.filename}

{% highlight html %}
<noscript>Please enable JavaScript to view the comments powered by facebook</a></noscript>
<div
  class="fb-comments"
  data-href="{% raw %}{{ site.url }}{{ page.url }}{% endraw %}"
  data-num-posts="{% raw %}{{ site.facebook.num_post }}{% endraw %}"
  data-width="{% raw %}{{ site.facebook.width }}{% endraw %}"
  data-colorscheme="{% raw %}{{ site.facebook.colorscheme }}{% endraw %}" ></div>
{% endhighlight %}

Этот шаблон теперь можно вставлять в любое место блога, где вам хочется добавить комментарии. Осталось добавить их в посты и страницы блога. Страницы и посты строятся на основе `post.html` и `page.html` файлов из директории `source/.layouts`. ДОбавляем в них следующий код до или после блока кода, отвечающего за комментарии disqus:
{% highlight html %}
{% raw %}{% if site.facebook.appid and page.comments == true %} {% endraw %}
  <section>
    <h1>Comments</h1>
    <div id="facebook_comments" aria-live="polite">
      {% raw %}{% include post/facebook_comments.html %}{% endraw %}
    </div>
  </section>
{% raw %}{% endif %}{% endraw %}
{% endhighlight %}

В целом это всё, что необходимо для добавления комментариев. Однако есть ещ> одна полезная вещь - добавление модерации. Для этого можно добавить следующую строку в файл `source/includes/custom/head.html`:
{% highlight html %}
{% raw %}{% if site.facebook.appid %}{% endraw %}
<meta property="fb:app_id" content="{% raw %}{{ site.facebook.appid }}{% endraw %}" />
{% raw %}{% endif %}{% endraw %}
{% endhighlight %}

Это должно разрешить модерацию всем админ пользователям вашего преложения в facebook.

В статье использовались материалы с сайта [blog.grambo.me.uk](http://blog.grambo.me.uk/blog/2012/02/20/adding-facebook-comments-to-octopress/)