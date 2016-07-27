---
title: "Типизация полей в elasticsearh для ELK стэка"
layout: post
comments: true
tags: [elasticsearch, ELK, logstash, monitoring]
categories: [elasticsearch]
---

Продолжая тему [оптимизации](/2016/05/optimization-standalone-elasticsearch-in-elk-stack.html) ELK стэка, а именно elasticsearch. Есть еще один действенный и удобный способ - типизировать поля elasticsearch.

По умолчанию после разбора сообщения logstash записывает все поля с типом string. От этого индексы elasticsearch "пухнут" и затрудняется дальнейшая работа с данными, например с цифрами.

Фильтры `grok` умеют типизировать цифровые поля, что значительно уменьшает размер хранимых данных. Однако пока grok умеет типизировать только два типа данных - integer и float.

Для того, чтобы типизация работала в фильтрах или паттернах надо указать тип поля:

<!--more-->

```
%{NUMBER:num:int}
```

После этого, если проверить маппинг в elasticsearch, можно увидеть следующее:

```
curl localhost:9200/Logstash-2014.10.07/_mapping?pretty
...
          "num" : {
            "type" : "long"
          }
...
```