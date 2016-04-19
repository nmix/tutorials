# Настройка доступа к VDS

### Исходное состояние
* VPS
* Ubuntu 14.04
* Доступ **root**

### Принятые обозначения
`client$` - команды вводятся на стороне *клиентской* консоли
`server$` - команды вводятся на стороне виртуального сервера

### Порядок действий

###### Устанавливаем новый пароль для *root*

```bash
client$ ssh root@server-ip-address
server$ passwd
server$ exit
```
###### Копируем ssh-ключ текущего пользователя на сервер

```bash
client$ ssh-copy-id root@server-ip-address
вводим пароль root…
client$ ssh root@server-ip-address
вводим пароль ssh-ключа…
server$ exit
```

###### Создаем нового пользователя для развертывания

```bash
client$ ssh root@server-ip-address
server$ adduser deploy
server$ adduser deploy sudo
server$ exit
```

###### Добавляем ssh-ключ для нового пользователя *deploy*

```bash
client$ ssh-copy-id deploy@server-ip-address
вводим пароль deploy…
client$ ssh deploy@server-ip-address
вводим пароль ssh-ключа (или не вводим, если вводили ранее)
server$ exit
```

###### При необходимости изменяем *hostname* сервера

```bash
client$ ssh deploy@server-ip-address
server$ sudo nano /etc/hostname
изменяем строку, сохраняемся, выходим
server$ sudo reboot
```

###### Назначаем символическое имя серверу

```bash
client$ sudo vim /etc/hosts
добавляем новую строку с адресом и символическим именем сервера (напр. myserver)
client$ ssh deploy@myserver
server$
```