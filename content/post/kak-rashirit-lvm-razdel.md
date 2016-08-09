+++
Categories = ["linux","server"]
Description = "How-to про расширение LVM раздела"
Tags = ["lvm","linux"]
date = "2016-08-09T12:56:02+03:00"
title = "Как расширить LVM раздел"

+++

Очень часто возникает необходимость расширения Linux раздела. Это актуально как для виртуальных машин, так и для железных серверов. Если в системе используется LVM, то сделать это достаточно просто.

<!--more-->

LVM - это прослойка между дисками и ОС. Которая добовляет гибкости в работе с разделами.
В общем виде схема такая:

/dev/sda <-> PV(physical volume) <-> VG(volume group) <-> LV(logical volume) <-> ext4

Файловая система устанавливается поверх LV. LV состоит из VG. VG состоит из PV. PV - состоит из устройств.
Вот картинка которая описывает схему работы LVM:
![LVM schema](/img/lvm_schema.png)

Итак, допустим у нас был диск /dev/sda, свободное место на нем закончилось и нужно расширить рутовый раздел:
 ```
 fdisk -l

Disk /dev/sda: 32.2 GB, 32212254720 bytes
255 heads, 63 sectors/track, 3916 cylinders
Units = cylinders of 16065 * 512 = 8225280 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x00037385

   Device Boot      Start         End      Blocks   Id  System
/dev/sda1   *           1          64      512000   83  Linux
Partition 1 does not end on cylinder boundary.
/dev/sda2              64        3917    30944256   8e  Linux LVM
```

После подключения нового диска, в системе он определился как /dev/sdb.
Создадим LVM раздел на этом диске с помощью fdisk:

```fdisk /dev/sdb```

```
 /dev/sdb
Команда (m для справки): n
Действие команды
   e   расширенный
   p   основной раздел (1-4)
p
Номер раздела (1-4): 1
Первый цилиндр (1045-1305, по умолчанию 1):
Используется значение по умолчанию 1045
Последний цилиндр или +size или +sizeM или +sizeK (1045-1305, по умолчанию
1305):
Используется значение по умолчанию 1305
Меняем тип раздела на LVM (8e):

Команда (m для справки): t
Номер раздела (1-4): 1
Шестнадцатеричный код (введите L для получения списка кодов): 8e
Системный тип раздела 1 изменен на 8e (Linux LVM)
Сохраняем изменения:

Команда (m для справки): w
Таблица разделов была изменена!
```
Здесь нужен ребут. Или если нужно без ребута, то:
```
echo 1 > /sys/block/sdb/device/rescan
```
Это заставит систему перечитать таблицу разделов на указанном диске.

Теперь можно запустить lvm:
```
lvm
```

И создать физический том из раздела /dev/sdb1:
```
lvm> pvcreate /dev/sdb1
```

Узнаем имя VG:
```
vgs
  VG                #PV #LV #SN Attr   VSize  VFree
  VolGroup00         2   2   0 wz--n- 20.50g    0
```
Расширяем VG за счет созданного PV:

```
vgextend VolGroup00 /dev/sdb1
```

Узнаем имя LV:

```
lvs
  LV      VG                Attr     LSize  Pool Origin Data%  Move Log Copy%  Convert
  lv_root VolGroup00 -wi-ao-- 35.53g
  lv_swap VolGroup00 -wi-ao--  3.97g
  ```
Здесь у нас два LV — lv_root и lv_swap, как ясно из названий, один для / и один для swap.
Расширяем lv_root:

```
lvextend -l +100%FREE /dev/VolGroup00/lv_root
```

И последнее, что нужно сделать — расширить файловую систему:

```
resize2fs -p /dev/mapper/VolGroup00-lv_root
```

Всё. После этого можно использовать новый расширенный диск.
