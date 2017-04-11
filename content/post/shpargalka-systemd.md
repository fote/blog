+++
Categories = ["server"]
Description = "Основные команды systemctl jounralctl. Про systemd в Ubuntu 16.04"
Tags = ["server","linux","systemd"]
date = "2017-04-10T18:35:14+03:00"
title = "Самая краткая шпаргалка systemd на примере Ubuntu 16.04"

+++
С версии 16.04 Ubuntu включает в себя systemd как init-систему по умолчанию. На systemd также перешли Debian, CentOS и другие популярные linux-дистрибутивы. Это краткая шпаргалка по командам systemctl и jounrnalctl

![systemctl status nginx](/img/systemctl_status_nginx.png)

<!--more-->

Unit - это базовый термин systemd, он обозначает ресурс, которым может управлять systemd - демон, сокет, точка монтирования и другое. В этой статье подразумевается что юнит - это демон. Файлы юнитов (аналог инит скриптов) лежат в ```/lib/systemd/system``` или ```/etc/systemd/```. Управлять юнитами (аналог команды service) можно с помощью команды systemctl. 

## systemctl

Список всех юнитов:
```
systemctl
```

Информация о статусе юнита:
```
systemctl status nginx
```

Релоад конфигурации демона:
```
systemctl reload nginx
```

Запустить/остановить юнит:
```
systemctl [start|stop] nginx
```

Включить/выключить автозапуск юнита при загрузке системы:
```
systemctl [enable|disable] nginx
```

Список юнитов которые не запустились во время последней загрузки системы:
```
systemctl --failed
```

## journalctl

journald - пишет логи юнитов, запущенных systemd, в бинарном формате. По умолчанию в ubuntu 16.04 лежат в ```/run/log/journal/<machine-id>```. machine-id - уникальный id-сервера, генерируется случайным образом. В 16.04 логи также пишутся с помощью rsyslogd, то есть они также достуны по старинке в ```/var/log```

Все логи всех юнитов с момента последней загрузки:
```
journalctl
```

Лог последней загрузки:
```
journalctl -b
```

Список послдених загрузок системы:
```
journalctl --list-boots
```

Лог определенной загрузки (boot-id - см. предыдущую команду):
```
journalctl -b <boot-id>
```

Логи определенного юнита:
```
journalctl -u nginx
```

Следить (tail -f) за логом определенного юнита:
```
journalctl -f -u nginx
```

Посмотреть сколько занимают логи на диске:
```
journalctl --disk-usage
```

Ограничить объем хранимых логов (journald сам занимается ротацией) можно по размеру:
```
journalctl --vacuum-size=1G
```

Или по времени:
```
journalctl --vacuum-time=1week
```



