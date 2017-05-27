+++
Categories = ["server"]
Description = "Как заменить диск в софтовом RAID массиве в linux"
Tags = ["raid", "linux", "server", "hardware"]
date = "2016-10-12T12:56:42+03:00"
title = "Замена диска в RAID-е"
Banner = "/img/zamena_diska_v_raid.png"
+++

Замена сбойного диска на сервере - самая распространенная задача.
С ней приходится сталкиваться с завидной регулярностью. Здесь описан пример замены сбойного диска в
софтовом RAID-е.

<!--more-->

Имеется такая конфигурация:
{{< highlight console >}}
# cat /proc/mdstat
Personalities : [linear] [multipath] [raid0] [raid1] [raid6] [raid5] [raid4] [raid10]
md0 : active raid1 sdb1[1] sda1[0]
      20964672 blocks super 1.0 [2/2] [UU]

md1 : active raid1 sdb2[1] sda2[0]
      1932547008 blocks super 1.0 [2/2] [UU]
{{< /highlight >}}

На сервере два диска, из них собраны два зеркала. И хотя ядро еще не пометило ни один из них как сбойный, sdb уже требует замены, так как на нем есть ошибки и пока он не умер совсем лучше его заменить.

![the end is near](/img/the_end_is_near.gif?w=50&h=50&c=true)

Симптомы неисправного диска:
в dmesg:
{{< highlight console >}}
[16634569.419928] ata2.00: exception Emask 0x0 SAct 0x3000 SErr 0x0 action 0x0
[16634569.420334] ata2.00: irq_stat 0x40000008
[16634569.420573] ata2.00: failed command: READ FPDMA QUEUED
[16634569.420887] ata2.00: cmd 60/00:60:aa:92:a5/01:00:74:00:00/40 tag 12 ncq 131072 in
[16634569.420887]          res 41/40:00:08:93:a5/00:01:74:00:00/00 Emask 0x409 (media error) <F>
[16634569.421841] ata2.00: status: { DRDY ERR }
[16634569.422080] ata2.00: error: { UNC }
[16634569.459283] ata2.00: configured for UDMA/133
[16634569.459294] sd 1:0:0:0: [sdb] Unhandled sense code
[16634569.459295] sd 1:0:0:0: [sdb]
[16634569.459297] Result: hostbyte=0x00 driverbyte=0x08
[16634569.459298] sd 1:0:0:0: [sdb]
[16634569.459299] Sense Key : 0x3 [current] [descriptor]
[16634569.459301] Descriptor sense data with sense descriptors (in hex):
[16634569.459302]         72 03 11 04 00 00 00 0c 00 0a 80 00 00 00 00 00
[16634569.459308]         74 a5 93 08
[16634569.459312] sd 1:0:0:0: [sdb]
[16634569.459313] ASC=0x11 ASCQ=0x4
[16634569.459315] sd 1:0:0:0: [sdb] CDB:
[16634569.459316] cdb[0]=0x28: 28 00 74 a5 92 aa 00 01 00 00
[16634569.459321] end_request: I/O error, dev sdb, sector 1957008136
[16634569.474340] md/raid1:md1: sdb2: rescheduling sector 1915078392
[16634569.489490] ata2: EH complete
{{< /highlight >}}

В SMART-е:
{{< highlight console >}}
# smartctl -x /dev/sdb | grep -i error
       					was completed without error.
       					without error or no self-test has ever
Error logging capability:        (0x01)	Error logging supported.
       					SCT Error Recovery Control supported.
  1 Raw_Read_Error_Rate     POSR--   079   063   044    -    88344976
  7 Seek_Error_Rate         POSR--   084   060   030    -    247156947
184 End-to-End_Error        -O--CK   100   100   099    -    0
191 G-Sense_Error_Rate      -O--CK   100   100   000    -    0
199 UDMA_CRC_Error_Count    -OSRCK   200   200   000    -    0
                            ||||___ R error rate
SMART Log at address 0x01 has    1 sectors [Summary SMART error log]
SMART Log at address 0x02 has    5 sectors [Comprehensive SMART error log]
GP    Log at address 0x03 has    5 sectors [Ext. Comprehensive SMART error log]
GP    Log at address 0x10 has    1 sectors [NCQ Command Error]
GP    Log at address 0x21 has    1 sectors [Write stream error log]
GP    Log at address 0x22 has    1 sectors [Read stream error log]
SMART Extended Comprehensive Error Log Version: 1 (5 sectors)
Device Error Count: 11
       	ER     = Error register
Error 11 [10] occurred at disk power-on lifetime: 29432 hours (1226 days + 8 hours)
  When the command that caused the error occurred, the device was active or idle.
  40 -- 51 00 00 00 00 74 a5 9b bf 00 00  Error: UNC at LBA = 0x74a59bbf = 1957010367
  Commands leading to the command that caused the error were:
...
...
Error 4 [3] occurred at disk power-on lifetime: 29432 hours (1226 days + 8 hours)
  When the command that caused the error occurred, the device was active or idle.
  40 -- 51 00 00 00 00 74 a5 93 08 00 00  Error: UNC at LBA = 0x74a59308 = 1957008136
  Commands leading to the command that caused the error were:
SCT Error Recovery Control:
0x0001  2            0  Command failed due to ICRC error
{{< /highlight >}}


Так выглядит здоровый диск:
{{< highlight console >}}
# smartctl -x /dev/sda | grep -i error
       					was completed without error.
       					without error or no self-test has ever
Error logging capability:        (0x01)	Error logging supported.
       					SCT Error Recovery Control supported.
  1 Raw_Read_Error_Rate     POSR--   082   063   044    -    180498332
  7 Seek_Error_Rate         POSR--   087   060   030    -    590780361
184 End-to-End_Error        -O--CK   100   100   099    -    0
191 G-Sense_Error_Rate      -O--CK   100   100   000    -    0
199 UDMA_CRC_Error_Count    -OSRCK   200   200   000    -    0
                            ||||___ R error rate
SMART Log at address 0x01 has    1 sectors [Summary SMART error log]
SMART Log at address 0x02 has    5 sectors [Comprehensive SMART error log]
GP    Log at address 0x03 has    5 sectors [Ext. Comprehensive SMART error log]
GP    Log at address 0x10 has    1 sectors [NCQ Command Error]
GP    Log at address 0x21 has    1 sectors [Write stream error log]
GP    Log at address 0x22 has    1 sectors [Read stream error log]
SMART Extended Comprehensive Error Log Version: 1 (5 sectors)
No Errors Logged
SCT Error Recovery Control:
0x0001  2            0  Command failed due to ICRC error
{{< /highlight >}}

Стоит заметить, что для дисков Seagate и  Samsung характерны огромные числа в полях Raw_Read_Error_Rate и
Seek_Error_Rate. У WD должен быть 0 или, по крайней мере, меньше порогового (threshold) значения.
Такие огромные значения связаны с тем, что это 48-битное число в котором содержится отношение
ошибок при позиционировании головки на общее количество позиционирований.


В общем, будем менять /dev/sdb. Для этого помечаем его как сбойный в обоих массивах:
{{< highlight console >}}
# mdadm /dev/md0 --fail /dev/sdb1
# mdadm /dev/md1 --fail /dev/sdb2
{{< /highlight >}}

После этого mdstat выглядит вот так:
{{< highlight console >}}
# cat /proc/mdstat
Personalities : [linear] [multipath] [raid0] [raid1] [raid6] [raid5] [raid4] [raid10]
md0 : active raid1 sdb1[1](F) sda1[0]
      20964672 blocks super 1.0 [2/1] [U_]

md1 : active raid1 sdb2[1](F) sda2[0]
      1932547008 blocks super 1.0 [2/1] [U_]
{{< /highlight >}}
[U_] означает что один из дисков помечен как сбойный и выведен из массива, если дисков, например 4, то будет [U_UU]
Теперь удаляем диски из массива:
{{< highlight console >}}
# mdadm /dev/md0 --remove /dev/sdb1
# mdadm /dev/md1 --remove /dev/sdb2
{{< /highlight >}}

{{< highlight console >}}
# cat /proc/mdstat
Personalities : [linear] [multipath] [raid0] [raid1] [raid6] [raid5] [raid4] [raid10]
md0 : active raid1 sda1[0]
      20964672 blocks super 1.0 [2/1] [U_]

md1 : active raid1 sda2[0]
      1932547008 blocks super 1.0 [2/1] [U_]

unused devices: <none>
{{< /highlight >}}


Если диск меняет инженер дата-центра, то для безопасности стоит удалить таблицу разделов и забить диск нулями, чтобы данные невозможно было восстановить. Для этого потребуется достаточно много времени(несколько часов):
{{< highlight console >}}
# sgdisk -Z /dev/sdb
# dd if=/dev/zero of=/dev/sdb
{{< /highlight >}}



Всё. Теперь явно удаляем диск из системы:
{{< highlight console >}}
# echo 1 >/sys/block/sdb/device/delete
{{< /highlight >}}

После отключения в dmesg появляется запись об этом:
{{< highlight console >}}
[17319158.829015] sd 1:0:0:0: [sdb] Synchronizing SCSI cache
[17319158.830979] sd 1:0:0:0: [sdb] Stopping disk
[17319159.833285] ata2.00: disabled
{{< /highlight >}}

Теперь можно менять диск в сервере физически. Значение ata2 - это номер шасси в котором он установлен,
по нему можно найти диск в большинстве случаев. Также серийный номер поможет в поисках. Впрочем, многое
зависит от конкретного случая, и описывать как найти диск в сервере я не буду.


После того как диск будет заменен - его надо добавить в массивы, после чего начнется resync.

Если диск не появляется в системе автоматически, можно заставить систему перечитать список подключенных устройств:
{{< highlight console >}}
# echo "- - -" >/sys/class/scsi_host/hostX/scan
{{< /highlight >}}
Где X - число от 0 до 6. Цифры соответствуют контроллерам ata1, ata2 и т.д., только у host нумерация с 0.

Когда диск появится в системе, то копируем разметку с соседнего диска (важно не перепутать в этой команде приемник и источник,
  ```sgdisk -G -R [куда_копируем] [откуда копируем])``` :
{{< highlight console >}}
# sgdisk -G -R /dev/sdb /dev/sda
{{< /highlight >}}

И, наконец, добавляем разделы нового диска обратно в соответствующие рейды и смотрим результат:
{{< highlight console >}}
# mdadm /dev/md0 --add /dev/sdb1
# mdadm /dev/md1 --add /dev/sdb2
{{< /highlight >}}

Если добавление прошло успешно, начнется ресинхронизация массива. Этот процесс идет в фоне и может занять несколько часов. За ним можно наблюдать в ```/proc/mdstat```. Ресинк аффектит приложения которые совершают ввод-вывод в это время. Чтобы снизить это влияние можно подрезать скорость ресинхронизации.
Для этого крутим параметры ```/proc/sys/dev/raid/speed_limit_min``` и  ```/proc/sys/dev/raid/speed_limit_max```. Значения по умолчания min=1000, max=200000. Уменьшим до 20000, например:

{{< highlight console >}}
# echo 20000 > /proc/sys/dev/raid/speed_limit_max
{{< /highlight >}}

На этом все, по окончанию мы имеем RAID с двумя здоровыми винтами.

![woohoo](/img/woohoo.gif)
