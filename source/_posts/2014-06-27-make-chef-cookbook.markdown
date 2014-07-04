---
layout: post
title: "make chef cookbook"
date: 2014-06-27 10:58
comments: true
categories: [howto]
tags: [chef, opscode, cookbook, test, ci]
published: false
---
ЭТО ПРЕДВАРИТЕЛЬНЫЙ ШАБЛОН!!!! СТАТЬЯ В РАБОТЕ!!!!!!

И так - эта статья будет о том как правильно писать кукбуки для chef.

Создаеём кукбук:
```
knife create cookbook nameofcookbook -o .
```

основная задача найти удобные способы автотестирования (на основе CI?)

вот тут есть про то, как добавить `foodcritic` в [Travis CI](https://travis-ci.org/):
http://technology.customink.com/blog/2012/06/04/mvt-foodcritic-and-travis-ci/

а тут рассказывается как добавить `knife test` в [Travis CI](https://travis-ci.org/):
http://technology.customink.com/blog/2012/07/06/mvt-knife-test-and-travisci/