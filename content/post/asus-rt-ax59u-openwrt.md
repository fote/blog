+++
Categories = ["server"]
Description = "Как я прошил роутер Asus RT-AX59U на OpenWRT"
Tags = ["network","linux","server","self-hosted"]
date = "2024-10-15 13:20:21 +0400"
title = "Прошивка Asus RT-AX59U на OpenWRT"
Banner = "/img/asus-rt.jpg"
+++

Давно хотел покрутить в руках OpenWRT, который уже давным давно существует, но до этого я всегда обходился дефолтными прошивками роутеров. Покажу на примере Asus RT-AX59U как прошивал. Процесс несложный, но есть пара граблей, о которых я хотел бы знать заранее, поэтому пишу. 
<!--more-->

## Что за зверь


![Порты](/img/asus-rt2.jpg#floatleft)

* CPU: MediaTek MT7986A
* RAM: 512 MB
* Flash: 128 MB
* Wi-Fi6

Антенны спрятаны внутри корпуса, поэтому коробка выглядит лаконично. Есть крепления на стену.

Цена: 10 тыс.руб.

{{< rawhtml >}}
<br clear="left"/>
{{< /rawhtml >}}

## План

Прошивка проходит в два этапа. Сначала мы на заводскую накатываем временный образ из trx-файла с initramfs. Это такая версия OpenWRT, которая использует RAM в качестве диска. Типа как LiveCD какой-нибудь ОС. Вторым этапом мы накатываем постоянную squashfs-версию (bin-файл), которая использует уже Flash-память роутера. 

![План](/img/asus-rt4.jpg)

Подробнее о разметке диска в прошивках OpenWRT можно прочитать [здесь](https://openwrt.org/docs/techref/flash.layout).

## Прошивка

**После установки OpenWRT не будет работать Wi-Fi (он выключен по умолчанию). Поэтому прошивать нужно только используя провод**

0. Запускаем роутер и подключаемся к LAN порту (патч-корд есть в комплекте)
1. Открываем [http://192.168.1.1](http://192.168.1.1) и проходим визард первоначальной настройки. Если роутер уже был настроен, то его нужно сбросить с помощью кнопки Reset на корпусе.

2. В админке, на странице [http://192.168.1.1/Advanced_FirmwareUpgrade_Content.asp](http://192.168.1.1/Advanced_FirmwareUpgrade_Content.asp) загружаем trx-файл c initramfs-прошивкой. Для Asus RT-AX59U файл можно скачать на [странице роутера на сайте OpenWRT](https://openwrt.org/toh/asus/rt-ax59u) (там ссылка на google drive). Вот [тут](/files/openwrt-23_rt-ax59u-initramfs.trx) положил копию, на всякий.
![Загрузка trx](/img/asus-rt3.jpg)

3. После загрузки заходим сюда – [http://192.168.1.1/cgi-bin/luci/admin/system/flash](http://192.168.1.1/cgi-bin/luci/admin/system/flash). Логин – root, пароль пустой. Напоминаю, что Wi-Fi не работает, и попасть в админку роутера можно ТОЛЬКО используя провод.
![Загрузка squashfs-sysupgrade](/img/asus-rt5.jpg)
Загружаем bin-файл с squashfs-sysupgrade-прошивкой. Его тоже качаем с  [сайта OpenWRT](https://openwrt.org/toh/asus/rt-ax59u). Или [здесь](/files/openwrt-23.05.5-mediatek-filogic-asus_rt-ax59u-squashfs-sysupgrade.bin)

4. Вы великолепны! Радуемся OpenWRT на роутере. Можно приступать к настройке. По умолчанию логин – root, пароль пустой.
![Финал](/img/asus-rt6.jpg)






