+++
Categories = [""]
Description = ""
Tags = [""]
date = ""
title = ""
Banner = "/img/terminal.jpg"
draft = true
+++

Недавно вышла новая версия Raspberry Pi - Zero W. Это то же самое что просто Zero, только с Wi-fi и Bluetooth на борту. В общем решил заказать, поиграться.

<!--more-->



В качестве носителя Raspberry Pi Zero W использует microSD карту, в отличае от "большого" брата, который использует miniSD. И как и на всех остальных версиях можно запускать Raspbian. Это специальная ОС, которая собрана для запуска на платах Raspberry Pi.  

Чтобы поставить Raspbian 

Raspbian:
https://www.raspberrypi.org/downloads/raspbian/


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
22:22:05 fote:~:$ diskutil list
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





  $ diskutil unmountDisk /dev/disk2



  $ sudo dd bs=1m if=Downloads/2017-09-07-raspbian-stretch-lite.img of=/dev/disk2 conv=sync


логин пароль
  pi
  raspberry



  сеть /etc/wpa_supplicant/wpa_supplicant.conf

  ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev          
update_config=1                           
country=GB                                                                             
network={                        
        ssid="435465"                         
        psk="z54l-f43s-nbf8"                             
        key_mgmt=WPA-PSK                                         
}     




SSH
PermitRootLogin yes


docker run -d --name super5-db -p 127.0.0.1:3306:3306 -e MYSQL_DATABASE=super5-master -e MYSQL_ROOT_PASSWORD=password --restart=always -v /opt/mysql:/var/lib/mysql mysql-charset
