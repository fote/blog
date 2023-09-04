+++
Categories = ["server"]
Description = "Быстрый способ установить и настроить OpenVPN сервер на Ubuntu Linux"
Tags = ["server"]
date = "2022-05-31T00:47:42+03:00"
title = "Установка и настройка OpenVPN сервера с помощью Docker"
Banner = "/img/openvpn-docker-install.png"
+++

Расскажу как за пару минут поднять OpenVPN сервер на Linux хосте. В своем примере я использую Ubuntu 20.04, но подойдет любой Linux, так как будем использовать докер.
<!--more-->

## Особенности моей инсталяции

Вообще подобная инструкция есть в разных уголках интернета, но в моей есть пара доработок:

1. **Нестандартный порт**:
VPN-сервер слушает на нестандартном для OpenVPN 1174, это позволяет отбиться от назойливых crawler-ов, которые пытаются брутфорсить все подряд в интернете.

2. **Возможность подключаться под одним пользователем несколько раз одновременно**: по умолчанию OpenVPN не дает такой возможности. Если добавить ```duplicate-cn``` в конфиг, то один пользователь сможет создавать несколько подключений одновременно.

## Подготовка

Итак, берем чистый Ubuntu сервер.

Поднимать OpenVPN мы будем в Docker контейнере, поэтому для начала нужно установить Docker на сервер. Это можно сделать одной командой:
{{< highlight console >}}
curl -fsSL https://get.docker.com -o get-docker.sh && sudo sh get-docker.sh
{{< /highlight >}}

Для запуска OpenVPN мы будем использовать образ [kylemanna/openvpn](https://hub.docker.com/r/kylemanna/openvpn) 

## TL;DR

Вот однострочник, который повторяет все следующие действия в инструкции. Его удобно скопировать в консоль, подставить свои значения, и тут же выполнить.

* Папка с конфигом определяется в переменной configdir перед выполнением;
* Заменить SERVERADDR на адрес своего сервера — IP или хостнейм (порт важно оставить как есть); 
* Заменить CLIENTNAME на свое имя пользователя

{{< highlight console >}}
export configdir=/etc/openvpn && mkdir $configdir && docker run -v $configdir:/etc/openvpn --rm kylemanna/openvpn ovpn_genconfig -u udp://SERVERADDR:11194 -e "duplicate-cn" && docker run -v $configdir:/etc/openvpn --rm -it kylemanna/openvpn ovpn_initpki && docker run -v $configdir:/etc/openvpn -d --name openvpn --restart=always -p 11194:1194/udp --cap-add=NET_ADMIN kylemanna/openvpn && docker run -v $configdir:/etc/openvpn --rm -it kylemanna/openvpn easyrsa build-client-full CLIENTNAME nopass && docker run -v $configdir:/etc/openvpn --rm kylemanna/openvpn ovpn_getclient CLIENTNAME > CLIENTNAME.ovpn
{{< /highlight >}}

После выполнения будет запущен VPN-сервер, и клиентский конфиг файл будет лежать в текущей директории.


## Установка OpenVPN по шагам

Создаем папку для конфигов и генерируем первичный конфиг, вместо ```SERVERADDR``` нужно подставить IP-адрес вашего сервера или его хостнейм:
{{< highlight console >}}
export configdir=/etc/openvpn
mkdir $configdir
docker run -v $configdir:/etc/openvpn --rm kylemanna/openvpn ovpn_genconfig -u udp://SERVERADDR:11194 -e "duplicate-cn"
{{< /highlight >}}

Теперь создаем корневые сертификаты. Потребуется ввести пароль (минимум 5 символов) и CommonName - это имя сервера (может быть любое):
{{< highlight console >}}
docker run -v $configdir:/etc/openvpn --rm -it kylemanna/openvpn ovpn_initpki
{{< /highlight >}}

Конфигурация готова, можно запустить наш сервер:
{{< highlight console >}}
docker run -v $configdir:/etc/openvpn -d --name openvpn --restart=always -p 11194:1194/udp --cap-add=NET_ADMIN kylemanna/openvpn
{{< /highlight >}}

Создаем пользователя. Потребуется пароль, который придумали выше (CLIENTNAME нужно заменить на имя юзера):
{{< highlight console >}}
docker run -v $configdir:/etc/openvpn --rm -it kylemanna/openvpn easyrsa build-client-full CLIENTNAME nopass
{{< /highlight >}}

Скачиваем файл конфигурации из контейнера на хост машину (CLIENTNAME нужно заменить на имя юзера):
{{< highlight console >}}
docker run -v $configdir:/etc/openvpn --rm kylemanna/openvpn ovpn_getclient CLIENTNAME > CLIENTNAME.ovpn
{{< /highlight >}}

Всё. Забираем файл с сервера, ставим [клиент OpenVPN](https://openvpn.net/vpn-client/), и импортируем этот конфиг в него. После подключения, весь траффик клиента будет проходить через VPN.



