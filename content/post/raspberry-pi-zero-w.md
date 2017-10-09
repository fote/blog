+++
Categories = [""]
Description = ""
Tags = [""]
date = ""
title = ""
Banner = "/img/terminal.jpg"
draft = true
+++

Raspbian:
https://www.raspberrypi.org/downloads/raspbian/



docker run -d --name super5-db -p 127.0.0.1:3306:3306 -e MYSQL_DATABASE=super5-master -e MYSQL_ROOT_PASSWORD=password --restart=always -v /opt/mysql:/var/lib/mysql mysql-charset