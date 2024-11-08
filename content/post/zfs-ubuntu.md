+++
Categories = ["man"]
Description = "Инструкция как установить zfs и собрать массив аналогичный raid0 raid1 raid10"
Tags = ["linux", "server", "hardware"]
date = "2019-01-14T16:35:42+03:00"
title = "Установка и создание пулов ZFS аналогичных RAID в Ubuntu"
Banner = "/img/zfs.png"
+++

ZFS - это файловая система на стероидах. С помощью ZFS можно собрать подобие RAID-массивов, но с дополнительными функциями, которые могут быть полезны в хозяйстве. Здесь покажу как установить zfs и собрать аналог RAID-10 массива на примере Ubuntu 18.04


<!--more-->

## Установка zfs

Изначально ZFS была доступна только в Solaris, но с помощью модулей ядра можно установить и в Linux.

На последних версиях Ubuntu установка абсолютно элементарная:
{{< highlight console >}}
#apt install zfs zfs-utils
{{< /highlight >}}

Автоматически устанавливаются модули ядра, а также тулзы для создания разделов.

В терминах zfs — сначала создается pool из разделов или дисков, потом на этот pool накатывается файловая система. Все как и в случае с raid-ами.


## Создание pool-а

Пока никаких пулов не создано:
{{< highlight console >}}
#zpool status
no pools available
{{< /highlight >}}

Создаем пул из сырых дисков. Пул можно также создавать из разделов диска (/dev/sdb1, /dev/sdc2 etc.):
{{< highlight console >}}
#zpool create -f mypoolname1 /dev/sdb /dev/sdc /dev/sdd
{{< /highlight >}}
Эта команда создала пул аналогичный RAID0(stripe) - без резервирования. Автоматически на этот пул накатывается ФС.


Смотрим созданные в системе пулы:
{{< highlight console >}}
#zpool list
NAME          SIZE  ALLOC   FREE  EXPANDSZ   FRAG    CAP  DEDUP  HEALTH  ALTROOT
mypoolname1  2.95G    70K  2.95G         -     0%     0%  1.00x  ONLINE  -
{{< /highlight >}}

После создания пула, просиходит его автомаунт в /
{{< highlight console >}}
#df -h
Filesystem                   Size  Used Avail Use% Mounted on
....
....
tmpfs                        396M     0  396M   0% /run/user/0
mypoolname1                  2.9G     0  2.9G   0% /mypoolname1
{{< /highlight >}}

Меняем маунтпоинт:
{{< highlight console >}}
#zfs set mountpoint=/mnt/data mypoolname1
{{< /highlight >}}
После выполнения сразу произойдет umount старой директории и mount в новой


## Аналог RAID1 и RAID10

Выше мы создавали пул аналогичный RAID0(stripe) без какого-либо резервирования данных. Если умрет хотя бы один диск из пула, то развалится весь пул. Чтобы повысить отказоустойчивость, создадим зеркалированный пул.

Вот так можно создать RAID1(mirror) с помощью zfs:
{{< highlight console >}}
#zpool create -f mymirrorpoolname1 mirror sdb sdc
{{< /highlight >}}

Создание RAID10 из шести (sdb, sdc, sdd, sde, sdf, sdg) дисков выглядит так:
{{< highlight console >}}
#zpool create -f myraid10pool mirror sdb sdc mirror sdd sde mirror sdf sdg
{{< /highlight >}}
цепочки ```mirror disk1 disk2``` можно продолжать и дальше




## Проверяем состояние пула

Проверить состояние всех пулов можно так:
{{< highlight console >}}
#zpool status -x
all pools are healthy
{{< /highlight >}}

Вот так выглядит zpool status когда умер один диск:
{{< highlight console >}}
#zpool status -x
  pool: testpool
 state: DEGRADED
status: One or more devices could not be used because the label is missing or
	invalid.  Sufficient replicas exist for the pool to continue
	functioning in a degraded state.
action: Replace the device using 'zpool replace'.
   see: http://zfsonlinux.org/msg/ZFS-8000-4J
  scan: resilvered 65K in 0h0m with 0 errors on Mon Oct  8 19:08:25 2018
config:

	NAME        STATE     READ WRITE CKSUM
	testpool    DEGRADED     0     0     0
	  mirror-0  ONLINE       0     0     0
	    sdb     ONLINE       0     0     0
	    sdc     ONLINE       0     0     0
	  mirror-1  DEGRADED     0     0     0
	    sdd     ONLINE       0     0     0
	    sde     UNAVAIL      0     0     0

errors: No known data errors
{{< /highlight >}}

## Как заменить диск в zfs пуле

Если случилась беда, и умер один диск в пуле с резервированием, то его можно легко заменить. Процедура аналогична замене диска в RAID-массиве.

Вот пример из жизни когда в zfs-RAID10 массиве сломался один диск:
{{< highlight console >}}
#zpool status
  pool: data
 state: DEGRADED
status: One or more devices could not be used because the label is missing or
	invalid.  Sufficient replicas exist for the pool to continue
	functioning in a degraded state.
action: Replace the device using 'zpool replace'.
   see: http://zfsonlinux.org/msg/ZFS-8000-4J
  scan: none requested
config:

	NAME        STATE     READ WRITE CKSUM
	data       DEGRADED     0     0     0
	  mirror-0  ONLINE       0     0     0
	    sda     ONLINE       0     0     0
	    sdb     ONLINE       0     0     0
	  mirror-1  ONLINE       0     0     0
	    sdc     ONLINE       0     0     0
	    sdd     ONLINE       0     0     0
	  mirror-2  DEGRADED     0     0     0
	    sde     ONLINE       0     0     0
	    sdf     UNAVAIL      0     0     0

errors: No known data errors
{{< /highlight >}}

Здесь видим, что ```sdf``` приказал долго жить и требует замены. В данном случае — это железный сервер и пул построен на целых дисках. Находим нужный диск в железном сервере, и "на горячую" меняем старый диск на новый такого же размера. В ```dmesg``` можно увидеть как определяется новый диск. В моем примере я вынул ```/dev/sdf``` из сервера, вставил новый диск, и он определился в системе с таким же именем.

Теперь меняем диск в пуле. Мы говорим: *заменить диск, который именовался в пуле как ```sdf``` на диск, который в системе именуется ```/dev/sdf``` (так же как и старый)*:
{{< highlight console >}}
#zpool replace -f data sdf /dev/sdf
{{< /highlight >}}

После этого начинается ресинк процесс, который может занять некоторое количество времени, в зависимости от размеров. Важный нюанс — чтобы ресинк прошел до конца, нельзя делать снапшоты во время ресинка, иначе процесс доходит до 99% и останавливается (https://forums.freebsd.org/threads/resilver-taking-very-long-time.61643/).
{{< highlight console >}}
#zpool status
  pool: data
 state: DEGRADED
status: One or more devices is currently being resilvered.  The pool will
	continue to function, possibly in a degraded state.
action: Wait for the resilver to complete.
  scan: resilver in progress since Wed Nov  7 20:21:08 2018
    38,9M scanned out of 2,22T at 4,86M/s, 133h17m to go
    13,0M resilvered, 0,00% done
config:

	NAME             STATE     READ WRITE CKSUM
	data            DEGRADED     0     0     0
	  mirror-0       ONLINE       0     0     0
	    sda          ONLINE       0     0     0
	    sdb          ONLINE       0     0     0
	  mirror-1       ONLINE       0     0     0
	    sdc          ONLINE       0     0     0
	    sdd          ONLINE       0     0     0
	  mirror-2       DEGRADED     0     0     0
	    sde          ONLINE       0     0     0
	    replacing-1  UNAVAIL      0     0     0
	      old        UNAVAIL      0     0     0
	      sdf        ONLINE       0     0     0  (resilvering)

errors: No known data errors
{{< /highlight >}}

После окончания пул снова в строю!

