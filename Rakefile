#############################################################################
#
# Modified version of jekyllrb Rakefile
# https://github.com/jekyll/jekyll/blob/master/Rakefile
#
#############################################################################

require 'rake'
require 'yaml'
require 'html-proofer'

CONFIG = YAML.load(File.read('_config.yml'))
USERNAME = CONFIG['username'] || ENV['GIT_NAME']
REPO = CONFIG['repo'] || 'cyberflow.github.com'
# DEST = CONFIG['destination'] || '_deploy'
DEST = '_site'

SOURCE_BRANCH = 'source'
DESTINATION_BRANCH = 'master'

def check_destination
  unless Dir.exist? DEST
    sh "git clone https://#{USERNAME}:#{ENV['GH_TOKEN']}@github.com/#{USERNAME}/#{REPO}.git #{DEST}"
  end
end

namespace :site do
  desc 'Generate the site'
  task :generate do
    puts 'Generate the site:'

    if Dir.exist?(DEST)
      puts "Remove dir #{DEST}"
      sh "rm -rf #{DEST}"
    end

    check_destination

    # sh "git checkout #{SOURCE_BRANCH} and prepare for generate"
    Dir.chdir(DEST) do
      sh "git checkout #{DESTINATION_BRANCH}"
      sh 'rm -rf *'
    end

    # Generate the site
    sh "bundle exec jekyll build -d #{DEST}"
  end

  desc 'Test the site'
  task :test do
    puts 'Test the site:'
    unless Dir.exist? DEST
      Rake::Task['site:generate'].invoke
    end
    HTMLProofer.check_directory(DEST).run
  end

  desc 'Push changes to remote origin'
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

    unless Dir.exist? DEST
      Rake::Task['site:generate'].invoke
    end

    # Commit and push to github
    sha = `git log`.match(/[a-z0-9]{40}/)[0]
    Dir.chdir(DEST) do
      sh "git status -s"
      sh "git add --all ."
      sh "git commit -m 'Updating to #{USERNAME}/#{REPO}@#{sha}.'"
      sh "git remote add deploy git@github.com:#{USERNAME}/#{REPO}.git"
      sh "git push --quiet deploy #{DESTINATION_BRANCH} --force"
      puts "Pushed updated branch #{DESTINATION_BRANCH} to GitHub Pages"
    end
  end
end
