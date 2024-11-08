+++
Categories = ["linux"]
Description = "How-to про расширение LVM раздела"
Tags = ["lvm","linux","server","man"]
date = "2016-08-09T12:56:02+03:00"
title = "Как расширить LVM раздел"
Banner = "/img/lvm.png"
+++

Очень часто возникает необходимость расширения Linux раздела. Это актуально как для виртуальных машин, так и для железных серверов. Если в системе используется LVM, то сделать это достаточно просто.

<!--more-->

LVM — это прослойка между дисками и ОС, которая добавляет гибкости при работе с дисками.
В общем виде схема такая:

`/dev/sda <-> PV(physical volume) <-> VG(volume group) <-> LV(logical volume) <-> ext4`

Файловая система устанавливается поверх LV. LV состоит из VG. VG состоит из PV. PV - состоит из устройств.
Вот картинка которая описывает схему работы LVM


![LVM schema](/img/lvm_schema.png)


Итак, допустим у нас был диск /dev/sda, свободное место на нем закончилось, и нужно расширить рутовый раздел:
{{< highlight console >}}
#fdisk -l

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
{{< /highlight >}}

После подключения нового диска, в системе он определился как /dev/sdb.

Если разделы больше 2ТБ, то нужно использовать parted и размечать под GPT:
{{< highlight console >}}
#parted -a optimal /dev/sdb
{{< /highlight >}}

Если меньше 2ТБ, то можно размечать fdisk-ом. Создадим LVM раздел на этом диске с помощью fdisk:

{{< highlight console >}}#fdisk /dev/sdb

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
{{< /highlight >}}
Здесь нужен ребут. Или если нужно без ребута, то:
{{< highlight console >}}
#echo 1 > /sys/block/sdb/device/rescan
{{< /highlight >}}
Это заставит систему перечитать таблицу разделов на указанном диске.

Теперь можно запустить lvm:
{{< highlight console >}}
#lvm
{{< /highlight >}}

И создать физический том из раздела /dev/sdb1:
{{< highlight console >}}
#lvm> pvcreate /dev/sdb1
{{< /highlight >}}

Узнаем имя VG:
{{< highlight console >}}
#vgs
  VG                #PV #LV #SN Attr   VSize  VFree
  VolGroup00         2   2   0 wz--n- 20.50g    0
{{< /highlight >}}
Расширяем VG за счет созданного PV:

{{< highlight console >}}
#vgextend VolGroup00 /dev/sdb1
{{< /highlight >}}

Узнаем имя LV:

{{< highlight console >}}
#lvs
  LV      VG                Attr     LSize  Pool Origin Data%  Move Log Copy%  Convert
  lv_root VolGroup00 -wi-ao-- 35.53g
  lv_swap VolGroup00 -wi-ao--  3.97g
 {{< /highlight >}}
Здесь у нас два LV — lv_root и lv_swap, как ясно из названий, один для / и один для swap.
Расширяем lv_root:

{{< highlight console >}}
#lvextend -l +100%FREE /dev/VolGroup00/lv_root
{{< /highlight >}}

И последнее, что нужно сделать — расширить файловую систему:

{{< highlight console >}}
#resize2fs -p /dev/mapper/VolGroup00-lv_root
{{< /highlight >}}

Всё. После этого можно использовать новый расширенный диск.
