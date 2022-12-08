+++
Categories = ["linux"]
Description = "Пример конфига для netplan"
Tags = ["server","linux"]
date = "2020-04-24T12:00:02+03:00"
title = "Пример настройки сети с помощью netplan в Ubuntu 18.04"
Banner = "/img/netplan.png"
+++

Начиная с Ubuntu 18.04, началось внедрение netplan в качестве конфигуратора сети. Здесь в двух словах расскажу как это работает и оставляю пример конфига, чтобы его можно было копипастить.

<!--more-->

## Как это работает
 
Netplan — это еще один уровень абстракции с декларативным конфигом, на основе которого потом генерируются конфиги для NetworkManager-а или других утилит. Это позволит унифицировать формат сетевых конфигов, но на мой взгляд ~~раньшебылолучше~~ проще было писать сразу в ```/etc/network/interfaces```

Вот схема работы с официального сайта:
![Схема работы netplan](/img/netplan1.png)

## Пример конфига

Итак, конфиг для netplan пишется в YAML формате и хранится здесь - ```/etc/netplan/*.yaml```
Вот пример серверного конфига:

{{< highlight yaml >}}
network:
  version: 2
  renderer: networkd
  ethernets:
    eth0:
      addresses: 
        - "192.168.1.2/24"
        - "2001:cafe:face:beef::dead:dead/64"
      gateway4: 192.168.1.1
      gateway6: "2001:dead:beef::1"
      nameservers:
          search: [ example.com ]
          addresses:
              - "8.8.8.8"
              - "8.8.4.4"
    eth1:
      addresses:
        - "172.17.0.1/24"
        - "172.18.0.1/24"
    eth2:
      dhcp4: yes
{{< /highlight >}}

После написания конфига нужно выполнить две команды чтобы конфиг применился.
Сначала перегенировать конфиг для бэкенда:
{{< highlight console >}}
# netplan generate
{{< /highlight >}}

Потом применить его для бэкенда:
{{< highlight console >}}
# netplan apply
{{< /highlight >}}

После этого изменения вступят в силу.
