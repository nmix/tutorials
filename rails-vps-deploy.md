# Развертывание Rails-приложения на VPS

### Исходное состояние
* VPS (виртуальный сервер)
* Ubuntu 14.04
* Доступ **root**
* Исходный код приложения в репозитории

### Принятые обозначения
`client$` - команды вводятся на стороне *клиентской* консоли
`server$` - команды вводятся на стороне виртуального сервера

### Порядок действий

#### Подготовка VPS

###### Настраиваем доступ для пользователя *deploy*

См. `vds-deploy-user.md`

###### Задаем размер *swap*-раздела

См. `ubuntu-swap-enable.md`

#### Установка пакетов VPS

###### Устанавливаем необходимые пакеты

```bash
server$ sudo apt-get update
server$ sudo apt-get install git-core curl zlib1g-dev build-essential libssl-dev libreadline-dev libyaml-dev libsqlite3-dev sqlite3 libxml2-dev libxslt1-dev libcurl4-openssl-dev python-software-properties libffi-dev nodejs
```

###### Устанавливаем *ruby*

См. `ruby-rbenv-install.md`

###### Устанавливаем nginx и passenger

```bash
server$ cd

# Install Phusion's PGP key to verify packages
gpg --keyserver keyserver.ubuntu.com --recv-keys 561F9B9CAC40B2F7
gpg --armor --export 561F9B9CAC40B2F7 | sudo apt-key add -

# Add HTTPS support to APT
sudo apt-get install apt-transport-https

# Add the passenger repository
sudo sh -c "echo 'deb https://oss-binaries.phusionpassenger.com/apt/passenger trusty main' >> /etc/apt/sources.list.d/passenger.list"
sudo chown root: /etc/apt/sources.list.d/passenger.list
sudo chmod 600 /etc/apt/sources.list.d/passenger.list
sudo apt-get update

# Install nginx and passenger
sudo apt-get install nginx-full passenger
```

###### Запускаем сервер nginx

```bash
server$ sudo service nginx start
```

###### Настраиваем Passenger

```bash
server$ sudo vim /etc/nginx/nginx.conf
```

Меняем имя пользователя
`user deploy;`

В категорию `# Phusion Passenger` добавляем строки
```
passenger_root /usr/lib/ruby/vendor_ruby/phusion_passenger/locations.ini;
passenger_ruby /home/deploy/.rbenv/shims/ruby; # If you use rbenv
```

###### Перезапускаем сервер nginx

```bash
server$ sudo service nginx restart
```

###### Устанавливаем СУБД

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

#### Переносим приложение на сервер

##### Устанавливаем *Capistrano*

В Gemfile нашего Rails проекта в группу development добавляем следующие гемы:
```
capistrano
capistrano-rails
capistrano-rbenv
```

Для определения крайних версий гемов можно заглянуть на [www.rubygems.org]().

 Устанавливаем гемы.
```bash
client$ bundle
client$ bundle --binstubs
```
##### Настраиваем *Capistrano*

```bash
client$ cap install STAGES=production
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
require 'capistrano/passenger'
...
```

Корректируем файл *config/deploy.rb*

```ruby
set :application, 'myapp'
set :repo_url, 'git@github.com:excid3/myapp.git'

set :deploy_to, '/home/deploy/myapp'

set :linked_files, %w{config/database.yml config/secrets.yml}
set :linked_dirs, %w{bin log tmp/pids tmp/cache tmp/sockets vendor/bundle public/system}
```
Переименовываем файл *database.yml* и *secrets.yml* и добавляем их в *.gitignore*
```bash
client$ git mv config/database.yml config/database.yml.examle
client$ git mv config/secrets.yml config/secrets.yml.example
```
```bash
client$ vim .gitignore
```

```
config/database.yml
config/secrets.yml
```

Корректируем файл *config/deploy/production.rb*
```ruby
set :stage, :production
server '127.0.0.1', user: 'deploy', roles: %w{web app db}
```
Вместо 127.0.0.1 необходимо указать фактический IP адрес сервера

Не забываем залить изменения в репозиторий
```bash
client$ git add .
client$ git commit -m "cap setup"
client$ git push
```

##### Запускаем *Capistrano*

###### Первый запуска deploy
```bash
client$ cap production deploy
```

Скорее всего будет выдана ошибка *…/config/database.yml does not exists*

На сервере создаем новые файлы *database.yml* и *secrets.yml*

```bash
server$ cd
server$ app_name/shared/config
server$ touch database.yml
server$ touch secrets.yml
```

Генерируем новый ключ и вводим его в *secrets.yml*
```bash
client$ rake secret
secret_key_output
server$ vim appname/shared/config/secrets.yml
```
```ruby
production:
  secret_key_base: secret_key_output
```

###### Добавляем гем СУБД в проект (на клиенте)

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
Добавляем изменения в репозиторий
```bash
client$ git add .
client$ git commit -m "db config"
client$ git push
```
###### Корректируем database.yml на сервере
PostgreSQL
```ruby
production:
  adapter: postgresql
  database: rtis
  username: deploy
  password: dbpassword
```
MySQL
```ruby
production:
  adapter: mysql2
  encoding: utf8
  reconnect: false
  database: rtis
  pool: 5
  username: dbuser
  password: dbpassword
  host: localhost
```
###### Создаем БД (на сервере)

PostgreSQL
```bash
server$ su
server$ su postgres
server$ psql
postgres=# create database rtis with owner = deploy;
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
mysql> CREATE DATABASE rtis DEFAULT CHARACTER SET utf8;
mysql> GRANT ALL PRIVILEGES ON rtis.* TO 'dbuser'@'localhost' IDENTIFIED BY 'dbpassword';
mysql> exit
```

###### Повторно запускаем deploy

```bash
client$ cap production deploy
```
Процедура достаточно продолжительная и должна пройти корректно с обильным отладочным выводом.

>В нашем случае имела место ошибка при выполнении миграции "AddPaymentMethodToOrder" связанная с добавлением внешнего ключа. 
Для локализации ошибки было выполнено (на локальной машине):
	1) установлен PostgreSql 
	2) создана база данных (см. выше)
	3) откорректирован (создан) файл config/database.yml в части группы production (см. выше)
	4) rake db:setup RAILS_ENV="production"
При этом, база данных заполняется корректно (не считая сообщения об ошибке создания об ошибке создания БД).
Аналогичная команда rake (~/rtis/current/) на сервере также отрабатывает корректно. После выполнения команды на сервере cap deploy production отработал корректно. Оставим пока так. В крайнем случае при развертывании на другом сервере будем "руками" разворачивать БД.

#### Подключаем приложение к nginx

```bash
server$ sudo vim /etc/nginx/sites-enabled/default
```

Вставляем следующий текст

```nginx
server {
        listen 80 default_server;
        listen [::]:80 default_server ipv6only=on;

        server_name mydomain.com;
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

Перезапускаем nginx
```bash
server$ sudo service nginx restart
```

Далее в браузере переходим по адресу сервера и наблюдаем страницу магазина.

#### Автоматический перезапуск rails приложения при deploy

Добавляем в *Gemfile* (на клиенте) следующий гем
```ruby
gem 'capistrano-passenger'
```
```bash
client$ bundle
```
Добавляем в *Capfile*
```ruby
require 'capistrano/passenger'
```
Добавляем в файл *config/deploy.rb*
```ruby
set :passenger_restart_with_touch, true
```
Загружаем изменения в репозиторий и выполняем deploy
```bash
client$ git add .
client$ git commit -m "autorestart"
client$ git push
client$ cap production deploy
```
Теперь, после deploy на сервере будет автоматически вызвана команда touch tmp/restart







