+++
Categories = ["server"]
Description = "Основные команды systemctl jounralctl. Про systemd в Ubuntu 16.04"
Tags = ["server","linux","systemd"]
date = "2017-04-10T18:35:14+03:00"
title = "Самая краткая шпаргалка systemd на примере Ubuntu 16.04"
Banner = "/img/systemctl_status_nginx.png"
+++
С версии 16.04 Ubuntu включает в себя systemd как init-систему по умолчанию. На systemd также перешли Debian, CentOS и другие популярные linux-дистрибутивы. Это краткая шпаргалка по командам systemctl и jounrnalctl

<!--more-->

Unit - это базовый термин systemd, он обозначает ресурс, которым может управлять systemd - демон, сокет, точка монтирования и другое. В этой статье подразумевается что юнит - это демон. Файлы юнитов (аналог инит скриптов) лежат в ```/lib/systemd/system``` или ```/etc/systemd/```. Управлять юнитами (аналог команды service) можно с помощью команды systemctl. 

## systemctl

Список всех юнитов:
{{< highlight console >}}
# systemctl
{{< /highlight >}}

Информация о статусе юнита:
{{< highlight console >}}
# systemctl status nginx
{{< /highlight >}}

Релоад конфигурации демона:
{{< highlight console >}}
# systemctl reload nginx
{{< /highlight >}}

Запустить/остановить юнит:
{{< highlight console >}}
# systemctl [start|stop] nginx
{{< /highlight >}}

Включить/выключить автозапуск юнита при загрузке системы:
{{< highlight console >}}
# systemctl [enable|disable] nginx
{{< /highlight >}}

Список юнитов которые не запустились во время последней загрузки системы:
{{< highlight console >}}
# systemctl --failed
{{< /highlight >}}

## journalctl

journald - пишет логи юнитов, запущенных systemd, в бинарном формате. По умолчанию в ubuntu 16.04 лежат в ```/run/log/journal/<machine-id>```. machine-id - уникальный id-сервера, генерируется случайным образом. В 16.04 логи также пишутся с помощью rsyslogd, то есть они также достуны по старинке в ```/var/log```

Все логи всех юнитов с момента последней загрузки:
{{< highlight console >}}
# journalctl
{{< /highlight >}}

Лог последней загрузки:
{{< highlight console >}}
# journalctl -b
{{< /highlight >}}

Список послдених загрузок системы:
{{< highlight console >}}
# journalctl --list-boots
{{< /highlight >}}

Лог определенной загрузки (boot-id - см. предыдущую команду):
{{< highlight console >}}
# journalctl -b <boot-id>
{{< /highlight >}}

Логи определенного юнита:
{{< highlight console >}}
# journalctl -u nginx
{{< /highlight >}}

Следить (tail -f) за логом определенного юнита:
{{< highlight console >}}
# journalctl -f -u nginx
{{< /highlight >}}

Посмотреть сколько занимают логи на диске:
{{< highlight console >}}
# journalctl --disk-usage
{{< /highlight >}}

Ограничить объем хранимых логов (journald сам занимается ротацией) можно по размеру:
{{< highlight console >}}
# journalctl --vacuum-size=1G
{{< /highlight >}}

Или по времени:
{{< highlight console >}}
# journalctl --vacuum-time=1week
{{< /highlight >}}



