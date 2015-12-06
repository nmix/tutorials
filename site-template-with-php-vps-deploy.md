
# Развертывание сайта на VPS (NGINX + PHP-FPM)

### Исходное состояние
* VPS (виртуальный сервер)
* Ubuntu 14.04
* Доступ **root**
* Исходный код сайта в репозитории

### Принятые обозначения
`client$` - команды вводятся на стороне *клиентской* консоли
`server$` - команды вводятся на стороне виртуального сервера

### Порядок действий

#### Подготовительные операции

###### Настраиваем доступ для пользователя *deploy*

См. `vds-deploy-user.md`

###### Задаем размер *swap*-раздела

См. `ubuntu-swap-enable.md`

#### Переносим исходный код сайта из репозитория на сервер

Для переноса исходников воспользуемся инструментом [Capistrano](http://capistranorb.com/). Все действия выполняются на стороне клиента (`client$`)

###### Устанавливаем *ruby*

См. `ruby-rbenv-install.md`

###### Устанавливаем *capistrano*

```bash
gem install capistrano
```

###### Настраиваем *capistrano*

Переходим в корневую директорию с исходниками сайта

```bash
cd /home/user/site-source
```

Запускаем команду

```bash
cap install
```

В директории появятся новые файлы:

```
├── Capfile
├── config
│   ├── deploy
│   │   ├── production.rb
│   │   └── staging.rb
│   └── deploy.rb
└── lib
    └── capistrano
            └── tasks
```

Корректируем файл *deploy.rb*

```bash
vim config/deploy.rb
```

Устанавливаем следующие опции:

```ruby
set :application, 'example'
set :repo_url, 'путь к репозиторию с исходниками'
set :deploy_to, '/home/deploy/example.com'
```

Корректируем файл *production.rb*

```bash
vim config/deploy/production.rb
```

Устанавливаем следующие опции:

```ruby
set :stage, :production
server 'server-ip-address', user: 'deploy', roles: %w{web app}
```

Отражаем проделанные изменения в репозитории:

```bash
git add .
git commit -m "cap setup"
git push
```

###### Переносим исходный код на сервер

Выполняется из корневой директории репозитория

```bash
cap deploy production
```
После ввода данной команды будет выводится больщое количество отладочной информации.

Проверяем

```bash
client$ ssh deploy@myserver
server$ cd example/current
server$ ls
```


#### Установка *nginx* и *php-fpm*

###### Устанавливем *nginx*

```bash
server$ sudo apt-get install nginx
```

###### Устанавливаем *php-fpm*

```bash
server$ sudo apt-get install php5-cli php5-common php5-mysql php5-gd php5-fpm php5-cgi php5-fpm php-pear php5-mcrypt
```

#### Настройка *php-fpm*

###### Корректируем файл *php.ini*
```bash
server$ sudo vim /etc/php5/fpm/php.ini
```
Устанавливаем опцию:
```
cgi.fix_pathinfo = 0
```

###### Корректируем файл *www.conf*
```bash
sudo vim /etc/php5/fpm/pool.d/www.conf
```
Устанавливаем следующие опции:
```conf
security.limit_extensions = .php .php3 .php4 .php5
listen = /var/run/php5-fpm.sock
listen.owner = deploy
listen.group = deploy
listen.mode = 0660
```

###### Перезапускаем *php-fpm*
```bash
server$ sudo service php5-fpm restart
```

#### Настройка *nginx*

###### Корректируем файл *nginx.conf*

```bash
server$ sudo vim /etc/nginx/nginx.conf
```
Устанавливаем опцию:
```
user deploy;
```

###### Создаем файл конфигурации сайта

```bash
server$ sudo touch /etc/nginx/sites-available/example.com
server$ sudo ln -s /etc/nginx/sites-available/example.com /etc/nginx/sites-enabled/
server$ sudo vim /etc/nginx/sites-available/example.com
```

Приводим файл к следующему виду:

```nginx
server
{
        listen          80;
        root            /home/deploy/exmaple/current;
        index           index.html;
        server_name     example.com;

        location /
        {
                try_files $uri $uri/ /index.php$args;
        }

        location ~ \.php$
        {
                try_files $uri =404;
                fastcgi_pass unix:/var/run/php5-fpm.sock;
                fastcgi_index index.php;
                include fastcgi_params;
                fastcgi_param SCRIPT_FILENAME  $document_root$fastcgi_script_name;
#               fastcgi_param PHP_VALUE "sendmail_path=/usr/sbin/sendmail -t -i -fmail@example.com";
#               fastcgi_param PHP_ADMIN_VALUE "open_basedir=/var/www/example.com/:/var/save_path/:/var/tmp_dir/";
        }
}

```

###### Удаляем файл конфигурации по-умолчанию

```bash
sudo rm /etc/nginx/sites-enabled/default
```

###### Перезапускаем *nginx*

```bash
server$ sudo service nginx restart
```

#### Завершение

После перезапуска *nginx* и *php-fpm* в браузере переходим по адресу (ip) сервера и видим наш сайт. Также проверяем работу сценария отправки почтового сообщения с сайта.




