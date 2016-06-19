---
title: "Тестирование сайта на jekyll и авто-deploy на github pages"
layout: post
comments: true
date: 2016-06-19 12:46
tags: [jekyll, testing]
categories: [jekyll]
---

Как и все в IT сайт нуждается в тестировании. Качество и количество тестов зависит от функциональности и значения сайта. Для своего блога я то же решил использовать тесты. Ну а так как все системные администраторы жутко ленивы, то и настроить автоматическую выкладку новых статей на [github pages](https://pages.github.com) (в моем случае через [travis-ci](http://travis-ci.org))

### html-proofer
Первым инструментом тестирования я выбрал [html-proofer](https://github.com/gjtorikian/html-proofer). Он позволяет проверять ссылки сайта, правильность оформления изображений а так же работоспособность внутренних и внешних скриптов.
Для работы нужно установить gem или добавить строчку в Gemfile:

```
gem 'html-proofer'
```

<!--more-->

Для быстрой ручной проверки можно выполнить следующие команды:

``` console
# bundle exec jekyll build
# bundle exec htmlproofer ./_site
```

Чтобы не проверять генерируемые jekyll кросслинки (например список категорий), можно добавить в шаблоны тэг `data-proofer-ignore`. ТАким образом можно уменьшить количество проверяемых ссылок:

``` html
<a href="http://notareallink" data-proofer-ignore>Not checked.</a>
```

Добавляем конфиг для travis-ci:

>.travis.yml
{:.filename}

``` yml
language: ruby
rvm:
  - 2.1
branches:
  only:
  - source
env:
  global:
  - NOKOGIRI_USE_SYSTEM_LIBRARIES=true
gemfile:
  - Gemfile
script:
  - bundle exec jekyll build
  - bundle exec htmlproofer ./_site/
```

Так же надо добавить строчку в конфиг jekyll:

>_config.yml
{:.filename}

``` yml
exclude: [vendor]
```

`exclude: [vendor]` - крайне важная строчка, т.к. travis ставит gem в корень проекта в директорию vendor. Если не добавить ее в исключения, jekyll будет пытаться обработать фалы gem-ов.

Тесты можно обернуть в `rake` таски для удобства.

### Автодеплой на github pages
После удачного тестирования можно и залить изменения на сайт. Я использую две ветки в репозитории: `source` - для хранения исходников постов и потрошков `jekyll` и, соответственно, `master` - для самого блога. В итоге задача сводится к генерации сайта средствами `jekyll` на стороне CI и push в определенную ветку репозитория.
Для этого можно использовать `rake` таски ruby. Вот пример:

>Rakefile
{:.filename}

``` ruby
#############################################################################
#
# Modified version of jekyllrb Rakefile
# https://github.com/jekyll/jekyll/blob/master/Rakefile
#
#############################################################################

require 'rake'
require 'yaml'

CONFIG = YAML.load(File.read('_config.yml'))
USERNAME = CONFIG['username'] || ENV['GIT_NAME']
REPO = CONFIG['repo'] || "cyberflow.github.com"
DEST = CONFIG['destination'] || '_deploy'

SOURCE_BRANCH = 'source'
DESTINATION_BRANCH = 'master'

def check_destination
  unless Dir.exist? DEST
    sh "git clone https://#{ENV['GIT_NAME']}:#{ENV['GH_TOKEN']}@github.com/#{USERNAME}/#{REPO}.git #{DEST}"
  end
end

namespace :site do
  desc 'Generate the site and push changes to remote origin'
  task :deploy do
    # Detect pull request
    if ENV['TRAVIS_PULL_REQUEST'].to_s.to_i > 0
      puts 'Pull request detected. Not proceeding with deploy.'
      exit
    end

    # Configure git if this is run in Travis CI
    if ENV['TRAVIS']
      puts 'Configure git'
      sh "git config --global user.name '#{ENV['GIT_NAME']}'"
      sh "git config --global user.email '#{ENV['GIT_EMAIL']}'"
      sh 'git config --global push.default simple'
    end

    # Make sure destination folder exists as git repo
    check_destination

    # sh "git checkout #{SOURCE_BRANCH}"
    Dir.chdir(DEST) do
      sh "git checkout #{DESTINATION_BRANCH}"
      sh 'rm -rf *'
    end

    # Generate the site
    sh "bundle exec jekyll build -d #{DEST}"

    # Commit and push to github
    sha = `git log`.match(/[a-z0-9]{40}/)[0]
    Dir.chdir(DEST) do
      sh "git status -s"
      sh "git add --all ."
      sh "git commit -m 'Updating to #{USERNAME}/#{REPO}@#{sha}.'"
      sh "git push --quiet origin #{DESTINATION_BRANCH} --force"
      puts "Pushed updated branch #{DESTINATION_BRANCH} to GitHub Pages"
    end
  end
end
```

и теперь можно добавить деплой после удачного тестирования:

>.travis.yml
{:.filename}

```
...
script:
  - bundle exec jekyll build
  - bundle exec htmlproofer ./_site/
after_success:
  - bundle exec rake site:deploy
```