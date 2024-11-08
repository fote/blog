+++
Categories = ["man"]
Description = "Некоторые советы при работе в bash"
Tags = ["bash"]
date = "2014-02-04 15:31:18 +0400"
title = "Полезные советы bash"
Banner = "/img/terminal.jpg"
+++
Здесь собраны несколько полезных советов при работе с bash в коммандной строке. Также их можно применять при написании скриптов.
Данные комманды были собраны из различных источников и опробованы.
<!--more-->


Вставить аргумент из предыдущей команды (в терминале работает сочетание клавиш *Alt + .*; очень удобная штука) :
{{< highlight console >}}
#!$
{{< /highlight >}}

Запустить последнюю команду, например, если при выполнении команды не хватило прав и нужно добавить *sudo*, то чтобы не печатать команду заново удобно пользоваться:
{{< highlight console >}}
#sudo !!
{{< /highlight >}}


Проверить открыт ли порт без telnet/nc:
{{< highlight console >}}
#echo >/dev/tcp/8.8.8.8/53 && echo "open"
{{< /highlight >}}

Выполнить команды из файла в текущем шелле:
{{< highlight console >}}
#source /etc/profile.d/rvm
{{< /highlight >}}

Подсктрока из переменной(первые 5 символов):
{{< highlight console >}}
#${my_variable:0:5}
{{< /highlight >}}

Создать несколько папок:
{{< highlight console >}}
#mkdir -p /tmp/{folder1,folder2,folder3}
{{< /highlight >}}

Посмотреть дерево процессов (с форками):
{{< highlight console >}}
#ps axwef
{{< /highlight >}}

 Проверить скорость записи диска. Данная команда пишет 512 МБ на диск и потом показывает скорость. Важным ключом является *conv=fdatasync*. Она заставляет dd сделать sync и убедится что все данные действительно записаны на диск. Если не использовать этот ключ, то dd будет писать в память и бенчмарк получится неверный:
{{< highlight console >}}
#dd if=/dev/zero of=/tmp/speedtest bs=1M count=512 conv=fdatasync; rm -f /tmp/speedtest
{{< /highlight >}}
Примерные значения скорости диска:

 - на обычных дисках (VMWare VM) - **~60 MB/sec - 150 MB/sec** (в зависимости от RAID-level)
 - на SSD (DigitalOcean VPS) - **~250 MB/sec**



Скорость чтения с диска:
{{< highlight console >}}
#hdparm -Tt /dev/sda
{{< /highlight >}}
Примерные значения:

 - обычные диски (VMware VM): cached read = **5200-6000 MB/sec**; buffered disk read = **100-350 MB/sec**
 - SSD (DigitalOcean SSD): cached read = **5900-6500 MB/sec**; buffered disk read = **460-560 MB/sec**

<br />

Разархивировать в новую директорию:
{{< highlight console >}}
#tar zxvf archive.tar.gz -C new_dir
{{< /highlight >}}

Быстро создать бэкап файла:
{{< highlight console >}}
#cp nginx.conf{,.bak}
{{< /highlight >}}

Доступ на расшареный ресурс Windows. После авторизации доступна команда *ls*. Чтобы скачать файл - *get somefilename.jpg*:
{{< highlight console >}}
#smbclient -U "DOMAIN\user" //192.168.1.1/shared
{{< /highlight >}}

Unzip в директорию:
{{< highlight console >}}
#unzip package.zip -d some_dir
{{< /highlight >}}

Переодически запускать команду и показывать вывод (по умолчанию раз в 2 секунды, менять интервал ключом *-n*):
{{< highlight console >}}
#watch ps aux
{{< /highlight >}}

Создать RAM-диск:
{{< highlight console >}}
#mount -t tmpfs tmpfs /tmpram -o size=512m
{{< /highlight >}}

Конвертация табуляций в пробелы:
{{< highlight console >}}
#expand tabsfile.txt > spacefile.txt
{{< /highlight >}}

Вернуться в предыдущую директорию:
{{< highlight console >}}
#cd -
{{< /highlight >}}

Когда *Ctrl + c* не работает:
{{< highlight console >}}
Ctrl + \
{{< /highlight >}}
