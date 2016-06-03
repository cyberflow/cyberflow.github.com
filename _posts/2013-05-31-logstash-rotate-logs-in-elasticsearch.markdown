---
layout: post
title: "logstash, ротация логов в elasticsearch"
date: 2013-05-31 19:24
comments: true
categories: [notice, linux, elasticsearch]
tags: [elasticsearch, logs, logstash]
published: true
---

Небольшая заметка как организовать ротацию логов, собирающихся при помощи [logstash](http://logstash.net/) в [elasticsearch](http://www.elasticsearch.org/).

В крон добавляем:
{% highlight bash %}
cat <<'EOF' > /etc/cron.daily/logstash
#!/bin/bash
logdate=$(date -d'-15 days' +"%Y.%m.%d")
curl -XDELETE "http://localhost:9200/logstash-${logdate}"
EOF
{% endhighlight %}

Этот скрипт будет запускаться ежедневно и удалять все логи из elasticsearch 15-дневной давности.
