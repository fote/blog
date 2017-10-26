+++
Categories = [""]
Description = ""
Tags = [""]
date = ""
title = ""
Banner = "/img/terminal.jpg"
draft=true
+++



docker run --rm -it -v /opt/mysql/:/var/lib/mysql --net=host -v  /tmp/backup2:/innobackupex vadio/innobackupex innobackupex --host=127.0.0.1 --user=root --password=''


rsync -av $PWD root@82.202.253.219:/tmp/backup2



docker run --rm -it -v /var/lib/mysql/:/var/lib/mysql --net=host -v /tmp/backup2/2017-10-26_12-40-09/:/innobackupex vadio/innobackupex innobackupex --apply-log /var/lib/mysql/

