Развертывание Rails-приложения на VPS
=====================================

### Исходное состояние

-   VPS (виртуальный сервер)

-   Ubuntu 14.04

-   Доступ **root**

-   Исходный код приложения в репозитории

### Принятые обозначения

`client$` - команды вводятся на стороне *клиентской* консоли `server$` - команды
вводятся на стороне виртуального сервера.
Предполагается, что данный сервер является *staging* (промежуточным), т.е. сюда будет осуществляться предварительное развертывание. После тестирования приложение развертывается на *production*-сервере.

## Подготовка VPS

### Настраиваем доступ для пользователя *deploy*

См. `01-vds-deploy-user.md`

### Задаем размер *swap*-раздела

См. `02-ubuntu-swap-enable.md`

### Устанавливаем необходимые пакеты

```bash
server$ sudo apt-get update
server$ sudo apt-get install git-core curl zlib1g-dev build-essential libssl-dev libreadline-dev libyaml-dev libsqlite3-dev sqlite3 libxml2-dev libxslt1-dev libcurl4-openssl-dev python-software-properties libffi-dev nodejs
```

### Устанавливаем *ruby*

См. `03-ruby-rbenv-install.md`

### Устанавливаем nginx и passenger

For Ubuntu 14.04 https://www.phusionpassenger.com/library/install/nginx/install/oss/trusty/

For Ubutntu 16.04 https://www.phusionpassenger.com/library/install/nginx/install/oss/xenial/

### Запускаем сервер nginx

```bash
server$ sudo service nginx start
```

### Настраиваем Passenger

```bash
server$ sudo vim /etc/nginx/nginx.conf
```

Меняем имя пользователя `user deploy;`

В категорию `# Phusion Passenger` добавляем строки

```
passenger_root /usr/lib/ruby/vendor_ruby/phusion_passenger/locations.ini;
passenger_ruby /home/deploy/.rbenv/shims/ruby; # If you use rbenv
```

```bash
server$ sudo service nginx restart
```

### Устанавливаем СУБД

PostgreSQL

```bash
server$ sudo apt-get install postgresql postgresql-contrib libpq-dev
server$ sudo su - postgres
server$ createuser deploy --pwprompt
```

MySQL

```bash
server$ sudo apt-get install mysql-server mysql-client libmysqlclient-dev
```

### Создаем БД

PostgreSQL

```bash
server$ su
server$ su postgres
server$ psql
postgres=# create database myapp with owner = deploy;
CREATE DATABES
postgres=# \q
server$ exit 
# => to root
server$ exit 
# => to deploy
server$ psql --user deploy --password rtis
Password for user deploy: dbpassword
rtis=> \q
server$ 
```

MySQL

```bash
$mysql -u root -p
mysql> CREATE DATABASE myapp DEFAULT CHARACTER SET utf8;
mysql> GRANT ALL PRIVILEGES ON myapp.* TO 'dbuser'@'localhost' IDENTIFIED BY 'dbpassword';
mysql> exit
```

Подготовка приложения
---------------------

### Конфигурация базы данных

Добавляем гем СУБД в *Gemfile* и устанавливаем его

PostgreSQL
```ruby
gem "pg", :group => :production
```

MySQL
```ruby
gem "mysql2", "~> 0.3.18", :group => :production
```

```bash
client$ bundle
```

Копируем файлы *database.yml* и *secrets.yml* и добавляем их в *.gitignore*

```bash
client$ cp config/database.yml config/database.yml.examle
client$ cp config/secrets.yml config/secrets.yml.example
```

```bash
client$ vim .gitignore
```

Добавляем строки
```
config/database.yml
config/secrets.yml
```

Корректируем *database.yml* с учетом подготовленных БД на сервере

PostgreSQL
```ruby
production:
  adapter: postgresql
  database: myapp
  username: deploy
  password: dbpassword
```

MySQL

```ruby
production:
  adapter: mysql2
  encoding: utf8
  reconnect: false
  database: myapp
  pool: 5
  username: dbuser
  password: dbpassword
  host: localhost
```

Добавляем изменения в репозиторий
```bash
client$ git add .
client$ git commit -m "db config"
client$ git push
```

### Capistrano

В Gemfile нашего Rails проекта в группу development добавляем следующие гемы:

```ruby
group :development do
  gem 'capistrano'
  gem 'capistrano-rails'
  gem 'capistrano-rbenv'
  gem 'capistrano-passenger'
  gem 'capistrano-rails-console'
end
```

Для определения крайних версий гемов можно заглянуть на [rubygems.org](http://rubygems.org)

Устанавливаем гемы.
```bash
client$ bundle
client$ bundle --binstubs
```

Устанавливаем Capistrano и указываем сценарии развертывания (*staging*,
*production*).

```bash
client$ cap install STAGES=staging,production
```

Корректируем файл *Capfile* (находится в корневой директории)
```ruby
require 'capistrano/setup'
require 'capistrano/deploy'

require 'capistrano/rbenv'
set :rbenv_type, :user
set :rbenv_ruby, '2.2.3'

require 'capistrano/bundler'
require 'capistrano/rails'
require 'capistrano/rails/console'
require 'capistrano/passenger'
...
```

Корректируем файл *config/deploy.rb*
```ruby
set :application, 'myapp'
set :repo_url, 'my-repository-path'

set :deploy_to, '/home/deploy/myapp'

set :linked_files, %w{config/database.yml config/secrets.yml}
set :linked_dirs, %w{bin log tmp/pids tmp/cache tmp/sockets vendor/bundle public/system}
```

Корректируем файл *config/deploy/staging.rb*
```ruby
set :stage, :production
server 'staging-server-ip-address', user: 'deploy', roles: %w{web app db}
set :branch, 'staging-branch'
```

Корректируем файл *config/deploy/production.rb*
```ruby
set :stage, :production
server 'production-server-ip-address', user: 'deploy', roles: %w{web app db}
set :branch, 'production-branch'
```
Создаем файл с заданиями для сервера *lib/capistrano/tasks/setup.rake*
```ruby
namespace :setup do
  desc "Upload database.yml file."
  task :upload_yml do
    on roles(:app) do
      execute "mkdir -p #{shared_path}/config"
      upload! StringIO.new(File.read("config/database.yml")), "#{shared_path}/config/database.yml"
      upload! StringIO.new(File.read("config/secrets.yml")), "#{shared_path}/config/secrets.yml"
    end
  end

  desc "Copy static stylesheets (fonts) to public"
  task :copy_static do
    on roles(:app) do
      # --- copy static files (fonts and css)
    end
  end

  desc "Seed the database."
  task :seed_db do
    on roles(:app) do
      within "#{current_path}" do
        with rails_env: :production do
          execute :rake, "db:seed"
        end
      end
    end
  end
end
```

Сохраняем изменения в репозиторий
```bash
client$ git add .
client$ git commit -m "cap setup"
client$ git push
```

## Развертывание
Запускаем развертывание на staging-сервер
```bash
client$ cap staging setup:upload_yml
client$ cap staging deploy
client$ cap staging setup:seed_db
client$ cap staging setup:copy_static
```
Для доступа к rails console на сервере вводим
```bash
client$ cap staging rails:console
```

## Подключение к nginx

Создаем файл конфигурации сайта
```bash
server$ sudo vim /etc/nginx/sites-available/myapp
```

Вставляем следующий текст
```nginx
server {
        listen 80 default_server;
        listen [::]:80 default_server ipv6only=on;

        server_name myapp.ru;
        passenger_enabled on;
        rails_env    production;
        root         /home/deploy/myapp/current/public;

        # redirect server error pages to the static page /50x.html
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
}
```
Заменяем ссылку на *default* страницу *nginx* на наш сайт:
```bash
server$ cd /etc/nginx/site-enabled
server$ sudo rm default
server$ sudo ln -s /etc/nginx/sites-available/myapp myapp
```

Перезапускаем nginx
```bash
server$ sudo service nginx restart
```

Далее в браузере переходим по адресу staging-сервера и наблюдаем страницу сайта.

- - -

Если всё хорошо, то необходимо повторить операции, описанные в разделах *Подготовка VPS*, *Развертывание*, *Подключение к nginx* для *production*-сервера
