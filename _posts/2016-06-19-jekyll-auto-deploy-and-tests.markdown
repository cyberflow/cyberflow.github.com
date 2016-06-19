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
      sh "git config --global user.name '#{USERNAME}'"
      sh "git config --global user.email '#{CONFIG['email']}'"
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
before_install:
  - openssl aes-256-cbc -K $encrypted_d1eeda8dc2f5_key -iv $encrypted_d1eeda8dc2f5_iv -in travis_rsa.enc -out .travis/travis_rsa -d
script:
  - bundle exec jekyll build
  - bundle exec htmlproofer ./_site/
after_success:
  - eval "$(ssh-agent -s)" #start the ssh agent
  - chmod 600 .travis/travis_rsa # this key should have push access
  - ssh-add .travis/travis_rsa
  - bundle exec rake site:deploy
```

Ниже пояснения к блокам `before_install` и `after_success`. Необходимо добавить ключ для travis, чтобы иметь возможность деплоить в репозиторий.

### Добавление ключа для деплоя из travis
Для того, чтобы travis смог деплоить наш сайт необходимо проделать еще один шаг. Надо добавить ключ в репозиторий.
Для начала сгенерим новый ключ (я не советую использовать уже созданные ключи). Посмотреть как это нужно делать можно [тут](https://help.github.com/articles/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent/)

``` console
# ssh-keygen -t rsa -b 4096 -C "travis"
```

После этого добавьте сгенеренный ключ в ваш репозиторий в настройках репозитория на github.
Теперь надо закриптовать ключ для travis:

``` console
$ travis encrypt-file deploy_key
encrypting deploy_key for domenic/travis-encrypt-file-example
storing result as deploy_key.enc
storing secure env variables for decryption

Please add the following to your build script (before_install stage in your .travis.yml, for instance):

    openssl aes-256-cbc -K $encrypted_0a6446eb3ae3_key -iv $encrypted_0a6446eb3ae3_key -in super_secret.txt.enc -out super_secret.txt -d

Pro Tip: You can add it automatically by running with --add.

Make sure to add deploy_key.enc to the git repository.
Make sure not to add deploy_key to the git repository.
Commit all changes to your .travis.yml.
```
