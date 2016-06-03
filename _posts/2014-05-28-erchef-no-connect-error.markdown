---
layout: post
title: "Erchef no_connect Error"
date: 2014-05-28 09:42
comments: true
categories: [chef]
tags: [chef, chef-server, erchef, error, no_connections]
published: true
---

### Предпосылки
Столкнулся с тем, что при большом количестве нод в [chef-server](http://opscode.com) версий *11.0.4* и *11.0.8* поставленных из omnibus пакетов для Ubuntu, он с большой периодичностью стал отдавать `500 Internal server error` нодам при выполнении куска рецепта, который делает поиск по data_bag'ам. При изучении логов выяснилось следующее:
В логах nginx'a видно, что запросы на поиск отправляются на бэкэнд erchef'а (ядро chef-server написанное на эрланге) и в ответ получаем `500`:

```
192.168.1.111 - - [27/May/2014:11:45:28 +0000] "GET /search/databag1?q=id:node1&sort=X_CHEF_id_CHEF_X%20asc&start=0&rows=1000 HTTP/1.1" 500 "0.051" 36 "-" "Chef Client/11.6.0 (ruby-1.9.3-p429; ohai-6.18.0; x86_64-linux; +http://opscode.com)" "127.0.0.1:8000" "500" "0.046" "11.6.0" "algorithm=sha1;version=1.0;" "auth1" "2014-05-27T11:45:28Z" "2jmj7l5rSw0yVb/vlWAYkK/YBwk=" 1011
```

Смотрим далее в логи `erchef`:

``` erlang
=ERROR REPORT==== 27-May-2014::11:45:28 ===
webmachine error: path="/search/databag1"
{error,
    {error,function_clause,
        [{chef_wm_search,'-make_bulk_get_fun/5-lc$^1/1-2-',
             [{error,no_connections},
             <<"databag1">>],
             [{file,"src/chef_wm_search.erl"},{line,233}]},
         {chef_wm_search,fetch_result_rows,4,
             [{file,"src/chef_wm_search.erl"},{line,351}]},
         {chef_wm_search,make_search_results,5,
             [{file,"src/chef_wm_search.erl"},{line,324}]},
         {chef_wm_search,to_json,2,
             [{file,"src/chef_wm_search.erl"},{line,130}]},
         {webmachine_resource,resource_call,3,
             [{file,"src/webmachine_resource.erl"},{line,166}]},
         {webmachine_resource,do,3,
             [{file,"src/webmachine_resource.erl"},{line,125}]},
         {webmachine_decision_core,resource_call,1,
             [{file,"src/webmachine_decision_core.erl"},{line,48}]},
         {webmachine_decision_core,decision,1,
             [{file,"src/webmachine_decision_core.erl"},{line,532}]}]}}
```
К сожалению то, что erchef пишет в свой лог не информативно. Однако путём длительного перебора удалось понять, что проблема в соединении к базе.

<!--more-->

### Решение
Суть проблемы в том, что erchef открывает *N* соединений к базе *postgresql* (по умолчанию 20) и вываливается с выше указанной ошибкой, если этих 20-и соединений ему не хватает.
Чтобы увеличить количество соединений нужно поправить конфиг erchef'a.

>/var/opt/chef-server/erchef/etc/app.config
{:.filename}

``` erlang
...
{pooler, [
           {pools, [[{name, "sqerl"},
                     {max_count, 60},
                     {init_count, 40},
                     {start_mfa, {sqerl_client, start_link, []}}]]},
           {metrics_module, folsom_metrics}
          ]},
...
```
Правим значения для `max_count` и `init_count` с 20 на большие. В моём случае пока хватает 60 и 40.