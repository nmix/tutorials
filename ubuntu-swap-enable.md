# Создание swap-раздела в Ubuntu
[источник]( https://www.digitalocean.com/community/tutorials/how-to-add-swap-on-ubuntu-14-04)

Проверяем систему на наличие *swap*

```bash
sudo swapon -s
```

Если отображается только заголовок, то *swap* отсутствует

Другой способ - команда *free*

```bash
free -m
```

Смотрим столбец total и строку swap. При отсутствии swap значение равно 0

Проверяем наличие места на жестком диске

```bash
df -h
```

Создаем swap файл (например, размером 512 МБ)

```bash
sudo fallocate -l 512M /swapfile
```

Активируем swap

```bash
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
sudo swapon -s
free -m
```

Добавляем swap конфигурацию в автозагрузку

```bash
sudo vim /etc/fstab
```

В конец файла добавляем строку

```bash
/swapfile    none    swap    sw    0    0
```

Перезагружаемся и проверяем.

Дополнительные полезные настройки:

```bash
sudo nano /etc/sysctl.conf
```

Добавляем в конец файла

```
vm.swappiness=10
vm.vfs_cache_pressure = 50
```

Сохраняем и перезагружаемся.


