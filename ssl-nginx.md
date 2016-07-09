# Включаем HTTPS на NGINX

Вольный перевод. [Источник](https://www.digitalocean.com/community/tutorials/how-to-secure-nginx-with-let-s-encrypt-on-ubuntu-14-04).
Важная ссылка - [Letsencrypt](https://letsencrypt.org/getting-started/)

## Получение и установка сертификатов

Устанавливаем `letsencrypt`
```bash
git clone https://github.com/letsencrypt/letsencrypt
cd letsencrypt
./letsencrypt-auto --help
```

Останавливаем `nginx` (если запущен)
```bash
sudo service nginx stop
```

Получаем сертификаты (в процессе получения надо будет ввести свой email и согласиться с условиями использования сервиса).
```bash
./letsencrypt-auto certonly --standalone -d example.ru -d www.example.ru
```
Если всё прошло удачно, то они (сертификаты) материализуются директории `/etc/letsencrypt/live/example.ru/`.

### Улучшенная защита

Создаем файл DH-шифров
```bash
sudo openssl dhparam -out /etc/ssl/certs/dhparam.pem 2048
```

### Настройка nginx

Приводим файл конфигурации сайта (например, `/etc/nginx/site-available/default`)
```nginx
server
{
        listen       443 ssl;
        root            /home/deploy/example.ru/current;
        index           index.html;
        server_name     example.ru www.example.ru;

        ssl_certificate /etc/letsencrypt/live/example.ru/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/example.ru/privkey.pem;

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
        }

        ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
        ssl_prefer_server_ciphers on;
        ssl_dhparam /etc/ssl/certs/dhparam.pem;
        ssl_ciphers 'ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-DSS-AES128-GCM-SHA256:kEDH+AESGCM:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA:ECDHE-ECDSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-DSS-AES128-SHA256:DHE-RSA-AES256-SHA256:DHE-DSS-AES256-SHA:DHE-RSA-AES256-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:AES:CAMELLIA:DES-CBC3-SHA:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!MD5:!PSK:!aECDH:!EDH-DSS-DES-CBC3-SHA:!EDH-RSA-DES-CBC3-SHA:!KRB5-DES-CBC3-SHA';
        ssl_session_timeout 1d;
        ssl_session_cache shared:SSL:50m;
        ssl_stapling on;
        ssl_stapling_verify on;
        add_header Strict-Transport-Security max-age=15768000;
}

server {
    listen 80;
    server_name example.ru www.example.ru;
    return 301 https://$host$request_uri;
}
```

Если проект реализован на `RubyOnRails + Passenger`, то файл конфигурации сайта будет иметь следующий вид:
```nginx
server
{
        listen       443 ssl;
        server_name     example.ru www.example.ru;
        passenger_enabled on;
        rails_env    production;
        root         /home/deploy/example.ru/current/public;

        ssl_certificate /etc/letsencrypt/live/example.ru/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/example.ru/privkey.pem;

        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }

        ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
        ssl_prefer_server_ciphers on;
        ssl_dhparam /etc/ssl/certs/dhparam.pem;
        ssl_ciphers 'ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-DSS-AES128-GCM-SHA256:kEDH+AESGCM:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA:ECDHE-ECDSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-DSS-AES128-SHA256:DHE-RSA-AES256-SHA256:DHE-DSS-AES256-SHA:DHE-RSA-AES256-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:AES:CAMELLIA:DES-CBC3-SHA:!aNULL:!eNULL:!EXPORT:!DES:!RC4:!MD5:!PSK:!aECDH:!EDH-DSS-DES-CBC3-SHA:!EDH-RSA-DES-CBC3-SHA:!KRB5-DES-CBC3-SHA';
        ssl_session_timeout 1d;
        ssl_session_cache shared:SSL:50m;
        ssl_stapling on;
        ssl_stapling_verify on;
        add_header Strict-Transport-Security max-age=15768000;
}

server {
    listen 80;
    server_name example.ru www.example.ru;
    return 301 https://$host$request_uri;
}
```

Запускаем ранее остановленный `nginx`

```bash
sudo service nginx start
```

> ВАЖНО. Сертификат выдается на 3 месяца с возможностью продления.

Не лишним будет ознакомиться со [статьей](https://habrahabr.ru/post/252507/), в которой приводится перечень действий при миграции на HTTPS.

## Продление сертификата

Сертификат выдается на 90 дней. Рекомендуется продливать сертификат каждый 60 дней.

Останавливаем Nginx

```bash
sudo service nginx stop
```

Запускаем обновление сертификата
```bash
cd /lets/encrypt/folder
./letsencrypt-auto renew
...
Requesting root privileges to run certbot...
  /home/deploy/.local/share/letsencrypt/bin/letsencrypt renew
-------------------------------------------------------------------------------
Processing /etc/letsencrypt/renewal/pcspectrum.ru.conf
-------------------------------------------------------------------------------

-------------------------------------------------------------------------------
new certificate deployed without reload, fullchain is
/etc/letsencrypt/live/pcspectrum.ru/fullchain.pem
-------------------------------------------------------------------------------

Congratulations, all renewals succeeded. The following certs have been renewed:
  /etc/letsencrypt/live/pcspectrum.ru/fullchain.pem (success)
```

Запускаем Nginx

```bash
sudo service nginx start
```


