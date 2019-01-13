+++
Categories = ["server"]
Description = "Как собрать массив ZFS"
Tags = ["linux", "server", "hardware"]
date = "2018-09-12T12:56:42+03:00"
title = "Установка и настройка ZFS в Ubuntu 18.04"
Banner = "/img/zfs.png"
draft = true
+++



apt install zfs zfs-utils



# zpool status
no pools available


создаем пул из сырых дисков 
# zpool create -f mypoolname1 /dev/sdb /dev/sdc /dev/sdd


# zpool list
NAME          SIZE  ALLOC   FREE  EXPANDSZ   FRAG    CAP  DEDUP  HEALTH  ALTROOT
mypoolname1  2.95G    70K  2.95G         -     0%     0%  1.00x  ONLINE  -


Автомаунтит в /
# df -h
Filesystem                   Size  Used Avail Use% Mounted on
udev                         2.0G     0  2.0G   0% /dev
tmpfs                        396M  5.8M  390M   2% /run
/dev/mapper/ubuntu--vg-root  6.6G  5.6G  693M  90% /
tmpfs                        2.0G     0  2.0G   0% /dev/shm
tmpfs                        5.0M     0  5.0M   0% /run/lock
tmpfs                        2.0G     0  2.0G   0% /sys/fs/cgroup
/dev/sda1                    472M   57M  391M  13% /boot
tmpfs                        396M     0  396M   0% /run/user/0
mypoolname1                  2.9G     0  2.9G   0% /mypoolname1

Меняем моунтпоинт:
# zfs set mountpoint=/mnt/data mypoolname1

После выполнения сразу делает umount старой директории и mount в новой




В нормальном состоянии:
# zpool status -x
all pools are healthy


Деградация рейда на zfs выглядит так:

root@ubuntu:~# zpool list
NAME       SIZE  ALLOC   FREE  EXPANDSZ   FRAG    CAP  DEDUP  HEALTH  ALTROOT
testpool  1.97G   141K  1.97G         -     0%     0%  1.00x  DEGRADED  -
root@ubuntu:~# zpool status testpool
  pool: testpool
 state: DEGRADED
status: One or more devices could not be used because the label is missing or
	invalid.  Sufficient replicas exist for the pool to continue
	functioning in a degraded state.
action: Replace the device using 'zpool replace'.
   see: http://zfsonlinux.org/msg/ZFS-8000-4J
  scan: none requested
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

Может быть так:


root@ubuntu:~# zpool status -x
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

