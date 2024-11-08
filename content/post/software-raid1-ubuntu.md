+++
Categories = ["man"]
Description = "Как собрать software raid 1 в ubuntu"
Tags = ["server","raid","linux"]
date = "2017-10-10T00:49:06+03:00"
title = "Создание программного RAID 1 зеркала в Ubuntu"
Banner = "/img/linux-raid.jpg"
+++

Краткий мануал как в Ubuntu 16.04 собрать софтварный RAID, и смонтировать его в /mnt.

<!--more-->

RAID1, он же "зеркало", нужен в тех случаях, когда нужно хранить важную информацию. Обычно зеркало создается из двух дисков, и информация записываtтся одновременно на два диска. Поэтому выход из строя одного диска не приведет к потере информации. Также такой тип RAID-а дает небольшое увеличение скорости чтения.

Когда диск выходит из строя, то он автоматически выпадает из массива, но сам массив продолжает работать на здоровом диске. Лучше до такого не доводить, и, если в S.M.A.R.T.-е появились ошибки, то лучше такой диск заменить. Вот [статья](/post/zamena-diska-v-raide/) про замену диска в рейде.

Итак, дано два диска ```sdc``` и ```sdd```:
{{< highlight console >}}
#fdisk -l /dev/sdc /dev/sdd 
Disk /dev/sdc: 1.8 TiB, 2000398934016 bytes, 3907029168 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes


Disk /dev/sdd: 1.8 TiB, 2000398934016 bytes, 3907029168 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
{{< /highlight >}}

Из них будет собрано зеркало и смонтировано в /mnt в качестве файлового хранилища.
Если разделы больше 2ТБ, то нужно использовать parted и размечать под GPT:
{{< highlight console >}}
#parted -a optimal /dev/sda
{{< /highlight >}}

Если меньше 2ТБ, то можно размечать fdisk-ом. Создадим разделы типа ```fd - Linux raid auto```.
{{< highlight console >}}
#fdisk /dev/sdd

Welcome to fdisk (util-linux 2.27.1).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table.
Created a new DOS disklabel with disk identifier 0xbb8eba44.

Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (1-4, default 1): 
First sector (2048-3907029167, default 2048):
Last sector, +sectors or +size{K,M,G,T,P} (2048-3907029167, default 3907029167):

Created a new partition 1 of type 'Linux' and of size 1.8 TiB.



Command (m for help): t
Selected partition 1
Partition type (type L to list all types): L

 0  Empty           24  NEC DOS         81  Minix / old Lin bf  Solaris
 1  FAT12           27  Hidden NTFS Win 82  Linux swap / So c1  DRDOS/sec (FAT-
 2  XENIX root      39  Plan 9          83  Linux           c4  DRDOS/sec (FAT-
 3  XENIX usr       3c  PartitionMagic  84  OS/2 hidden or  c6  DRDOS/sec (FAT-
 4  FAT16 <32M      40  Venix 80286     85  Linux extended  c7  Syrinx
 5  Extended        41  PPC PReP Boot   86  NTFS volume set da  Non-FS data
 6  FAT16           42  SFS             87  NTFS volume set db  CP/M / CTOS / .
 7  HPFS/NTFS/exFAT 4d  QNX4.x          88  Linux plaintext de  Dell Utility
 8  AIX             4e  QNX4.x 2nd part 8e  Linux LVM       df  BootIt
 9  AIX bootable    4f  QNX4.x 3rd part 93  Amoeba          e1  DOS access
 a  OS/2 Boot Manag 50  OnTrack DM      94  Amoeba BBT      e3  DOS R/O
 b  W95 FAT32       51  OnTrack DM6 Aux 9f  BSD/OS          e4  SpeedStor
 c  W95 FAT32 (LBA) 52  CP/M            a0  IBM Thinkpad hi ea  Rufus alignment
 e  W95 FAT16 (LBA) 53  OnTrack DM6 Aux a5  FreeBSD         eb  BeOS fs
 f  W95 Ext'd (LBA) 54  OnTrackDM6      a6  OpenBSD         ee  GPT
10  OPUS            55  EZ-Drive        a7  NeXTSTEP        ef  EFI (FAT-12/16/
11  Hidden FAT12    56  Golden Bow      a8  Darwin UFS      f0  Linux/PA-RISC b
12  Compaq diagnost 5c  Priam Edisk     a9  NetBSD          f1  SpeedStor
14  Hidden FAT16 <3 61  SpeedStor       ab  Darwin boot     f4  SpeedStor
16  Hidden FAT16    63  GNU HURD or Sys af  HFS / HFS+      f2  DOS secondary
17  Hidden HPFS/NTF 64  Novell Netware  b7  BSDI fs         fb  VMware VMFS
18  AST SmartSleep  65  Novell Netware  b8  BSDI swap       fc  VMware VMKCORE
1b  Hidden W95 FAT3 70  DiskSecure Mult bb  Boot Wizard hid fd  Linux raid auto
1c  Hidden W95 FAT3 75  PC/IX           bc  Acronis FAT32 L fe  LANstep
1e  Hidden W95 FAT1 80  Old Minix       be  Solaris boot    ff  BBT
Partition type (type L to list all types): fd
Changed type of partition 'Linux' to 'Linux raid autodetect'.

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.
{{< /highlight >}}

В итоге получаем два диска и на каждом по одному разделу на весь диск:
{{< highlight console >}}
#fdisk -l /dev/sdc /dev/sdd
Disk /dev/sdc: 1.8 TiB, 2000398934016 bytes, 3907029168 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x24b64209

Device     Boot Start        End    Sectors  Size Id Type
/dev/sdc1        2048 3907029167 3907027120  1.8T fd Linux raid autodetect


Disk /dev/sdd: 1.8 TiB, 2000398934016 bytes, 3907029168 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0xbb8eba44

Device     Boot Start        End    Sectors  Size Id Type
/dev/sdd1        2048 3907029167 3907027120  1.8T fd Linux raid autodetect
{{< /highlight >}}


Теперь собираем непосредственно сам RAID-массив:
{{< highlight console >}}
#mdadm --create --verbose /dev/md2 --level=1 --raid-devices=2 /dev/sdc1 /dev/sdd1
mdadm: Note: this array has metadata at the start and
    may not be suitable as a boot device.  If you plan to
    store '/boot' on this device please ensure that
    your boot-loader understands md/v1.x metadata, or use
    --metadata=0.90
mdadm: size set to 1953382464K
mdadm: automatically enabling write-intent bitmap on large array
Continue creating array? y
mdadm: Defaulting to version 1.2 metadata
mdadm: array /dev/md2 started.
{{< /highlight >}}

После того как система соберет массив, начнется процесс ресинхронизации.
Ждем окончания ресинка:
{{< highlight console >}}
#cat /proc/mdstat
Personalities : [raid1] [linear] [multipath] [raid0] [raid6] [raid5] [raid4] [raid10]
md2 : active raid1 sdd1[1] sdc1[0]
      1953382464 blocks super 1.2 [2/2] [UU]
      [>....................]  resync =  1.3% (25956480/1953382464) finish=166.6min speed=192804K/sec
      bitmap: 15/15 pages [60KB], 65536KB chunk
{{< /highlight >}}

После того как он кончится, создаем файловую систему на устройстве массива:
{{< highlight console >}}
#mkfs.ext4 /dev/md2
{{< /highlight >}}

Теперь, чтобы зеркало автоматом собиралось после ребута, выполняем:
{{< highlight console >}}
#mdadm --examine --scan
ARRAY /dev/md/0  metadata=1.2 UUID=5c8952f8:8456e312:d0b5af49:a7e38514 name=cs37907:0
ARRAY /dev/md/1  metadata=1.2 UUID=2b6d40e1:1d5515f0:5dfe78ca:868250d0 name=cs37907:1
ARRAY /dev/md/2  metadata=1.2 UUID=96fea4eb:5040d522:f83a5802:ea3b6a74 name=cs37907:2
{{< /highlight >}}
Здесь mdadm нашел три рейда, md2 - мы только что создали.
Добавляем строку с md2 в ```/etc/mdadm/mdadm.conf```


Обновляем initramfs:
{{< highlight console >}}
#update-initramfs -u
{{< /highlight >}}

Все готово. Монтируем в /mnt:
{{< highlight console >}}
#mount /dev/md2 /mnt
{{< /highlight >}}

И добавляем в fstab:
{{< highlight console >}}

/dev/md2	/mnt	ext4	defaults	0	0
{{< /highlight >}}
Теперь после перезагрузки RAID будет автоматически пересобираться и монтироваться в /mnt
