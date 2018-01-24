+++
Categories = ["server"]
Description = "Мануал по переносу базы MySQL без даунтайма"
Tags = ["mysql","linux"]
date = "2018-01-24T18:15:24+03:00"
title = "Как перенести базу MySQL на новый сервер без даунтайма"
Banner = "/img/perenos_mysql_0.png"
+++

Часто возникают ситуации с переездами базы между серверами. Здесь расскажу как перенести базу MySQL с одного сервера на другой без даунтайма.

<!--more-->


## Дано

Имеется такая ситуация:
![Перенос mysql 1](/img/perenos_mysql_1.png)

Есть приложение, в настройках которого прописан адрес MySQL сервера в виде домена ```mysql.server.loc```. Этот домен прописан в DNS и резолвится в 1.1.1.1. Есть новый сервер 2.2.2.2. Надо перенести mysql базу со старого сервера на новый. Нацеливать приложение можно как через его настройки, так и через DNS, меняя запись A-типа.

Перенос в несколько шагов:

1. Создать новый сервер
2. Настроить репликацию мастер-мастер. Приложение при этом смотрит на старый сервер.
3. Перенацелить приложение на новый сервер.
4. Выключить репликацию на новом. Выключить старый сервер. 

## Создание нового сервера

В данном случае я использую Ubuntu 16.04.3 и MySQL 5.7. Установим mysql на новый сервер:
{{< highlight console >}}
# apt-get update && apt-get install mysql-server
{{< /highlight >}}

## Настройка репликации master-slave

Для того чтобы работала репликация в конфиге mysql надо добавить опцию записи binlog-ов. В них будут записываться изменения которые происходят в базе в бинарном формате. После этого эти изменения разъезжаются по репликам и тем самым поддерживается консистентное состояние данных.

Включаем binlog в my.cnf:
{{< highlight INI >}}
log-bin = /var/log/mysql
expire-logs-days = 3
max-binlog-size  = 1024M
server-id        = 5
{{< /highlight >}}

* ```log-bin``` - путь где будем хранить бинлоги
* ```expire-logs-days``` - длина логов в днях. За это время надо поднять и запустить слейв.
* ```max-binlog-size``` - максимальный размер логов. Если достигнут максимум, то логи будут ротироваться, не смотря на expire-logs-days
* ```server-id``` - id-сервера в реплике. *должен быть разным на разных серверах*


Создадим пользователя, под которым будет происходить репликация и дадим права на репликацию:
{{< highlight mysql >}}
mysql> CREATE USER 'repluser'@'localhost' IDENTIFIED BY 'repluserpassword';
mysql> GRANT REPLICATION SLAVE ON *.* TO 'repluser'@'%' IDENTIFIED BY 'repluserpassword';
{{< /highlight >}}

Теперь сделаем бэкап базы с помощью innobackupex (это утилита от Percona которая делает бэкап с указанием позиции в bin-логе) в директорию /tmp/mysqlbackup. [Здесь](https://www.percona.com/doc/percona-xtrabackup/LATEST/installation/apt_repo.html#installing-percona-xtrabackup-from-percona-apt-repository) инструкция по инсталяции innobackupex. 
{{< highlight console >}}
# innobackupex --host=127.0.0.1 --user=root --password='rootpassword' /tmp/mysqlbackup 
{{< /highlight >}}

Можно запускать в [докер-контейнере](https://hub.docker.com/r/vadio/innobackupex/):
{{< highlight console >}}
docker run --rm -it -v /var/lib/mysql/:/var/lib/mysql --net=host -v /tmp/mysqlbackup:/innobackupex vadio/innobackupex innobackupex --host=127.0.0.1 --user=root --password='rootpassword'
{{< /highlight >}}

В конце вывода будет что-то типа:
{{< highlight console >}}
180122 14:55:45 Backup created in directory '/tmp/mysqlbackup/2018-01-22_14-55-42/'
MySQL binlog position: filename 'mysql-bin.000001', position '26115'
{{< /highlight >}}

Нам нужны filename и position. Запоминаем их, они пригодятся во время развертывания этого бэкапа на новом сервере. Эти значения также есть в файле ```xtrabackup_binlog_info``` в папке с бэкапом.

Копируем папку с бэкапом на новый сервер. Например rsync-ом (предварительно надо сделать доступ на новый сервер со старого по ssh):
{{< highlight console >}}
# rsync -av /tmp/mysqlbackup/2018-01-22_14-55-42 root@2.2.2.2:/tmp/
{{< /highlight >}}

Когда перенос закончится, логинимся на новый сервер и разворачиваем бэкап. Для этого скопируем содержимое папки бэкапа в /var/lib/mysql и запустим innobackupex:
{{< highlight console >}}
# mv /tmp/2018-01-22_14-55-42/ /var/lib/mysql/
# innobackupex --apply-log /var/lib/mysql/
{{< /highlight >}}

Запускаем MySQL на новом сервере, заходим в mysql-консоль и прописываем адрес мастера и момент времени с которого надо начинать подтягивать данные:
{{< highlight console >}}
mysql> CHANGE MASTER TO MASTER_HOST='1.1.1.1', MASTER_PORT=3306, MASTER_USER='repluser', MASTER_PASSWORD='repluserpassword', MASTER_LOG_FILE='mysql-bin.000001', MASTER_LOG_POS=26115;
{{< /highlight >}}

Проверяем:
{{< highlight console >}}
mysql> show slave status\G;
*************************** 1. row ***************************
               Slave_IO_State:
                  Master_Host: 1.1.1.1
                  Master_User: repluser
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000001
          Read_Master_Log_Pos: 26115
               Relay_Log_File: mysqld-relay-bin.000002
                Relay_Log_Pos: 253
        Relay_Master_Log_File: bin.000001
             Slave_IO_Running: No
            Slave_SQL_Running: No
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 
              Relay_Log_Space: 221131
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File:
           Master_SSL_CA_Path:
              Master_SSL_Cert:
            Master_SSL_Cipher:
               Master_SSL_Key:
        Seconds_Behind_Master: NULL
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 0
               Last_SQL_Error:
  Replicate_Ignore_Server_Ids:
             Master_Server_Id: 5
                  Master_UUID: 92eff853-0263-11e7-a7c4-0242ac110002
             Master_Info_File: /var/lib/mysql/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State:
           Master_Retry_Count: 86400
                  Master_Bind:
      Last_IO_Error_Timestamp:
     Last_SQL_Error_Timestamp:
               Master_SSL_Crl:
           Master_SSL_Crlpath:
           Retrieved_Gtid_Set:
            Executed_Gtid_Set:
                Auto_Position: 0
1 row in set (0.00 sec)
{{< /highlight >}}

Проверяем Master_Host должен быть ip старого сервера, Master_Log_File и Read_Master_Log_Pos должны быть те, которые показал innobackupex.

Стартуем слэйв и смотрим статус:
{{< highlight console >}}
mysql> start slave;
OK.
mysql> show slave status\G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 1.1.1.1
                  Master_User: repluser
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000001
          Read_Master_Log_Pos: 30124
               Relay_Log_File: mysqld-relay-bin.000002
                Relay_Log_Pos: 4262
        Relay_Master_Log_File: bin.000001
             Slave_IO_Running: No
            Slave_SQL_Running: No
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 30124
              Relay_Log_Space: 221131
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File:
           Master_SSL_CA_Path:
              Master_SSL_Cert:
            Master_SSL_Cipher:
               Master_SSL_Key:
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 0
               Last_SQL_Error:
  Replicate_Ignore_Server_Ids:
             Master_Server_Id: 5
                  Master_UUID: 92eff853-0263-11e7-a7c4-0242ac110002
             Master_Info_File: /var/lib/mysql/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State:
           Master_Retry_Count: 86400
                  Master_Bind:
      Last_IO_Error_Timestamp:
     Last_SQL_Error_Timestamp:
               Master_SSL_Crl:
           Master_SSL_Crlpath:
           Retrieved_Gtid_Set:
            Executed_Gtid_Set:
                Auto_Position: 0
1 row in set (0.00 sec)
{{< /highlight >}}

Теперь сервера между собой работают в master-slave режиме. Писать можно только в master, читать из обоих.

![Настройка репликации master-slave](/img/perenos_mysql_2.png)

Ждем пока новый сервер догонит мастера. Время отставания слейва от мастера - Seconds_Behind_Master. Exec_Master_Log_Pos на новом сервере должен быть такой же как position у master ноды. В примере выше слейв уже догнал мастера.
Посмотреть position на мастер ноде можно с помощью команды:
{{< highlight console >}}
mysql> show master status\G
*************************** 1. row ***************************
             File: mysql-bin.000001
         Position: 30124
{{< /highlight >}}


## Настройка репликации master-master

Когда слейв догнал мастера, идем на старый инстанс и добавляем параметры для репликации, чтобы репликация была master+master. Здесь уже не надо указывать MASTER_LOG_POS и MASTER_LOG_FILENAME:
{{< highlight console >}}
mysql> CHANGE MASTER TO MASTER_HOST='2.2.2.2', MASTER_PORT=3306, MASTER_USER='repluser', MASTER_PASSWORD='repluserpassword';

mysql> start slave;
{{< /highlight >}}


Теперь сервера работают как мастер-мастер. В таком режиме лучше всегда писать только в одну голову. Если писать в обеи - может возникнуть неконсистентность. Мастер-мастер у MySQL работает не всегда хорошо.

![Настройка репликации master-master](/img/perenos_mysql_3.png)


## Выключение старого сервера

После того как получилась конструкция мастер-мастер, можно перенацелить приложение, изменив его настройки или изменить A запись в DNS.

![Выключение старого сервера](/img/perenos_mysql_4.png)


Когда к старому серверу кончатся обращения (посмотреть -  ```mysql> show full processlist;```, можно остановить слейв на новом сервере:
{{< highlight console >}}
mysql> stop slave;
{{< /highlight >}}


Всё. MySQL теперь работает на новом сервере, приложение ходит в него. Старый можно выключать.

![Выключение старого сервера](/img/perenos_mysql_5.png)


