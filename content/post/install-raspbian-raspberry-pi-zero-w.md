+++
Categories = ["linux"]
Description = "Как установить Raspbian на Raspberry Pi Zero W и настроить Wi-Fi с помощью wpa_supplicant"
Tags = ["raspberry_pi"]
date = "2017-10-27T01:15:44+03:00"
title = "Обзор и начальная установка Raspberry Pi Zero W"
Banner = "/img/rpi_zero_w_1.jpg"
+++

Недавно вышла новая версия Raspberry Pi Zero W. Это то же самое что просто Zero, только с Wi-fi и Bluetooth на борту. В общем решил заказать, поиграться.

<!--more-->

##  Комплект 
Заказывал с AliExpress-а: вот [здесь](https://ru.aliexpress.com/item/2017-Raspberry-Pi-Zero-W-Board-1GHz-CPU-512MB-RAM-with-WIFI-Bluetooth-RPI-0-W/32802521209.html?spm=a2g0s.9042311.0.0.Lk45wX) сама платка и к ней еще [доп.набор из переходников](https://ru.aliexpress.com/item/7-in-1-Raspberry-Pi-Zero-W-Camera-Holder-Acrylic-Case-Heat-Sink-Mini-HDMI-Adapter/32803652515.html?spm=a2g0s.9042311.0.0.Lk45wX), камеры, корпуса и GPIO пинов. Вот так выглядит весь комплект:

![Весь набор](/img/rpi_zero_w_3.jpg)

Характеристики Raspberry Pi Zero W:

* 1GHz, single-core CPU
* 512MB RAM
* Mini-HDMI port
* Micro-USB On-The-Go port
* Micro-USB power
* HAT-compatible 40-pin header
* Composite video and reset headers
* CSI camera connector
* 802.11n wireless LAN Wi-Fi
* Bluetooth 4.0
* 65mm x 30mm x 5mm


GPIO пины здесь нужно напаять вручную, вот так:
<iframe width="560" height="315" src="https://www.youtube.com/embed/97Or02ihJMo?start=300" frameborder="0" allowfullscreen></iframe>

Распиновка такая же как на моделях Raspberry Pi 2,3,Zero:

![Распиновка Raspberry Pi Zero W](/img/rpi_zero_w_6.png)

[Здесь](https://pinout.xyz/) есть дополнительная информация по каждому пину.


Пока что я их не паял, а решил провести первоначальную установку системы.

## Установка Raspbian

В качестве носителя Raspberry Pi Zero W использует microSD карту, в отличие от старшей модели платы, которая работает на SD. Объем карты должен быть не меньше 2ГБ. После установки на двухгигабайтную карту, свободного места останется около 500МБ — особо не разбежишься. Поэтому если нужно хранить что-то объемное на карте, то лучше взять побольше.

![Raspberry Pi Zero W и microSD карта](/img/rpi_zero_w_5.jpg)

Раньше, когда только появлялись платы Raspberry Pi, еще не было специального дистрибутива Linux для них, и многие использовали обычный Debian, собранный под ARM процессоры. Это было не очень удобно, потому что приходилось вручную ставить разные модули ядра и драйвера. Сейчас же есть прекрасный Raspbian — это тот же Debian, но допиленный для использования на Raspberry Pi. Многие вещи поддерживает "из коробки", есть удобные консольные утилиты для всяческой настройки и легковесный desktop environment, на случай если планируется запускать с GUI интерфейсом.

Скачать образ можно здесь:
[https://www.raspberrypi.org/downloads/raspbian/](https://www.raspberrypi.org/downloads/raspbian/)

Я скачал RASPBIAN STRETCH LITE. Это консольная версия без GUI. Теперь нужно записать его на карту.

### ***Windows***

Можно воспользоваться утилитой [Win32diskimager](https://sourceforge.net/projects/win32diskimager/) или любой похожей - [Rufus](https://rufus.akeo.ie/), [Universal USB Creator](https://www.pendrivelinux.com/universal-usb-installer-easy-as-1-2-3/).

![Win32DiskImager](/img/win32diskimager1.png)

### ***Mac OS***

Подключаем карту и смотрим какие диски есть в системе:

{{< highlight console >}}
$ diskutil list
/dev/disk0 (internal, physical):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:      GUID_partition_scheme                        *251.0 GB   disk0
   1:                        EFI EFI                     209.7 MB   disk0s1
   2:          Apple_CoreStorage Macintosh HD            250.0 GB   disk0s2
   3:                 Apple_Boot Recovery HD             650.0 MB   disk0s3
/dev/disk1 (internal, virtual):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:                  Apple_HFS Macintosh HD           +249.6 GB   disk1
                                 Logical Volume on disk0s2
                                 E3AA7CD7-2FF0-4E8C-A081-D37A05FB0815
                                 Unencrypted
/dev/disk2 (internal, physical):
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:     FDisk_partition_scheme                        *1.9 GB     disk2
   1:                 DOS_FAT_32 TT                      1.9 GB     disk2s1

{{< /highlight >}}

Видно что /dev/disk2 - это наша флешка. Отмонтируем ее:
{{< highlight console >}}
$ diskutil unmountDisk /dev/disk2
{{< /highlight >}}

И теперь запишем образ Raspbian (не надо добавлять номер раздела, просто /dev/disk2):
{{< highlight console >}}
$ sudo dd bs=1m if=2017-09-07-raspbian-stretch-lite.img of=/dev/disk2 conv=sync
{{< /highlight >}}

### ***Linux***

Подключаем карту и смотрим какие диски есть в системе:
{{< highlight console >}}
$ sudo fdisk -l

Disk /dev/sda: 111.8 GiB, 120034123776 bytes, 234441648 sectors

Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0xfaa7714f
Device Boot Start End Sectors Size Id Type
/dev/sda1 * 2048 1026047 1024000 500M 7 HPFS/NTFS/exFAT
/dev/sda2 1026048 234438655 233412608 111.3G 7 HPFS/NTFS/exFAT
........

Disk /dev/sdc: 15 GiB, 16043212800 bytes, 31334400 sectors

Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x690a7a2e
Device Boot Start End Sectors Size Id Type
/dev/sdc1 * 0 2931839 2931840 1.4G 0 Empty
/dev/sdc2 2880880 2885487 4608 2.3M ef EFI (FAT-12/16/32)
{{< /highlight >}}

Наша флешка - ```/dev/sdc```. Если замонтирован раздел, размонтируем:
{{< highlight console >}}
$ sudo umount /dev/sdc1
{{< /highlight >}}

И теперь запишем образ Raspbian (не надо добавлять номер раздела, просто /dev/sdc):
{{< highlight console >}}
$ sudo dd bs=1m if=2017-09-07-raspbian-stretch-lite.img of=/dev/sdc conv=sync
{{< /highlight >}}




## Первый запуск

После записи ОС на microSD карту, вставляем ее в raspberry и подключаем питание, клавиатуру и монитор.

![Raspberry Pi Zero W в сборе](/img/rpi_zero_w_2.jpg)

Что интересно, запускается плата при подключении питания в любой из microUSB портов, но вот клавиатура работает только в определенном. Поэтому, если после загрузки не работает клавиатура — просто поменяйте местами разъемы.

***Логин и пароль по умолчанию в raspbian:***
```
login:      pi
password:   raspberry
```



## Настройка Wi-Fi

Как я писал выше - raspbian "из коробки" отлично подходит для raspberry pi. И все нужное для настройки wi-fi уже есть в системе.

У меня дома стоит обычный wi-fi роутер и создана беспроводная сеть c ***WPA2-PSK***. Чтобы подключить Pi к этой сети, редактируем файл ```/etc/wpa_supplicant/wpa_supplicant.conf```. Моя сеть называется ```4te-WIFI``` и пароль - ```mywifipassword```. Пароль в этом файле хранится в открытом виде. ```key_mgmt``` для WPA2-PSK все равно указывается WPA-PSK.

Вот так выглядит мой конфиг:
{{< highlight console >}}
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev          
update_config=1                           
country=RU                                                                             
network={                        
        ssid="4te-WIFI"                         
        psk="mywifipassword"                             
        key_mgmt=WPA-PSK                                         
}     
{{< /highlight >}}

Рестартуем:
{{< highlight console >}}
$ sudo reboot
{{< /highlight >}}

Вуаля! Интерфейс wlan0 получил IP-адрес от роутера и готов к работе:
{{< highlight console >}}
$ ifconfig
lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1  (Local Loopback)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

wlan0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.1.42  netmask 255.255.255.0  broadcast 192.168.1.255
        inet6 fe80::245e:56a5:398e:34c2  prefixlen 64  scopeid 0x20<link>
        ether b8:27:ea:fe:34:2a  txqueuelen 1000  (Ethernet)
        RX packets 1418  bytes 174338 (170.2 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 901  bytes 280257 (273.6 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
{{< /highlight >}}



Включим ssh и добавим его в автостарт:
{{< highlight console >}}
$ sudo systemctl start ssh
$ sudo systemctl enable ssh
{{< /highlight >}}

![Raspbian boot screen](/img/rpi_zero_w_4.jpg)


Теперь Rasperry Pi настроен и готов к экспериментам!


