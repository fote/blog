+++
Categories = ["server", "postgres"]
Description = "Список основных команд PostgreSQL для успешной копипасты в консоль"
Tags = ["server","linux","postgres", "cheatsheet","man"]
date = "2023-02-07T00:35:14+03:00"
title = "Шпаргалка PostgreSQL"
Banner = "/img/shpargalka-postgres.jpg"
+++
Здесь собрал команды, которые часто приходится выполнять работая с PostgreSQL в виде краткой шпаркалки.
<!--more-->

## Базовые команды

Подключение к инстансу на локалхосте под пользователем postgres:
{{< highlight console >}}
$psql -U postgres -h localhost
{{< /highlight >}}

Список баз:
{{< highlight postgresql >}}
\l
{{< /highlight >}}

Подключиться к базе:
{{< highlight postgresql >}}
\c dbname
{{< /highlight >}}

Cписок таблиц в базе:
{{< highlight postgresql >}}
\dt
{{< /highlight >}}

Cписок таблиц в базе, в названии которых есть *mytable*:
{{< highlight postgresql >}}
\dt *mytable*
{{< /highlight >}}

Cписок индексов:
{{< highlight postgresql >}}
\di
{{< /highlight >}}


Включает или выключает вывод результата списком, а не таблицей:
{{< highlight postgresql >}}
\x
{{< /highlight >}}

Вот пример вывода таблицей и списком:
{{< highlight postgresql >}}
postgres=# select * from pg_user;
     usename     | usesysid | usecreatedb | usesuper | userepl | usebypassrls |  passwd  | valuntil | useconfig
-----------------+----------+-------------+----------+---------+--------------+----------+----------+-----------
 postgres        |       10 | t           | t        | t       | t            | ******** |          |
 testuser        |    16321 | f           | f        | f       | f            | ******** |          |
(2 rows)

postgres=# \x
Expanded display is on.
postgres=# select * from pg_user;
-[ RECORD 1 ]+----------------
usename      | postgres
usesysid     | 10
usecreatedb  | t
usesuper     | t
userepl      | t
usebypassrls | t
passwd       | ********
valuntil     |
useconfig    |
-[ RECORD 2 ]+----------------
usename      | testuser
usesysid     | 16321
usecreatedb  | f
usesuper     | f
userepl      | f
usebypassrls | f
passwd       | ********
valuntil     |
useconfig    |
{{< /highlight >}}


## Работа с пользователями

Список пользователей:
{{< highlight postgresql >}}
\du
{{< /highlight >}}

Создать пользователя:
{{< highlight postgresql >}}
CREATE USER username WITH PASSWORD 'password';
{{< /highlight >}}

Выдать права на базу:
{{< highlight postgresql >}}
GRANT ALL PRIVILEGES ON DATABASE dbname TO username;
{{< /highlight >}}

Изменить пароль пользователя:
{{< highlight postgresql >}}
ALTER USER username WITH PASSWORD 'new_password';
{{< /highlight >}}

Удалить пользователя:
{{< highlight postgresql >}}
DROP USER IF EXISTS username;
{{< /highlight >}}


## Обслуживание

Посмотреть размер всех баз:
{{< highlight postgresql >}}
SELECT datname, pg_size_pretty(pg_database_size(datname))
FROM pg_database
ORDER BY pg_database_size(datname) DESC;
{{< /highlight >}}

Размер таблиц и индексов:
{{< highlight postgresql >}}
SELECT
    TABLE_NAME,
    pg_size_pretty(table_size) AS table_size,
    pg_size_pretty(indexes_size) AS indexes_size,
    pg_size_pretty(total_size) AS total_size
FROM (
    SELECT
        TABLE_NAME,
        pg_table_size(TABLE_NAME) AS table_size,
        pg_indexes_size(TABLE_NAME) AS indexes_size,
        pg_total_relation_size(TABLE_NAME) AS total_size
    FROM (
        SELECT ('"' || table_schema || '"."' || TABLE_NAME || '"') AS TABLE_NAME
        FROM information_schema.tables
    ) AS all_tables
    ORDER BY total_size DESC
) AS pretty_sizes;
{{< /highlight >}}


Список всех работающих запросов:
{{< highlight postgresql >}}
SELECT 
  pid,
  age(clock_timestamp(), query_start),
  usename,
  query 
FROM pg_stat_activity 
WHERE query != '<IDLE>' AND query NOT ILIKE '%pg_stat_activity%' 
ORDER BY query_start desc;
{{< /highlight >}}

Список запросов работающих дольше 5 минут:
{{< highlight postgresql >}}
SELECT
  pid,
  now() - pg_stat_activity.query_start AS duration,
  query,
  state
FROM pg_stat_activity
WHERE (now() - pg_stat_activity.query_start) > interval '5 minutes';
{{< /highlight >}}

Остановить запрос по pid:
{{< highlight postgresql >}}
SELECT pg_cancel_backend(pid);
{{< /highlight >}}

Принудительно остановить запрос:
{{< highlight postgresql >}}
SELECT pg_terminate_backend(pid);
{{< /highlight >}}

Остановить запросы работающие дольше 5 минут:
{{< highlight postgresql >}}
SELECT
  pg_terminate_backend(pid),
  now() - pg_stat_activity.query_start AS duration,
  query,
  state
FROM pg_stat_activity
WHERE (now() - pg_stat_activity.query_start) > interval '5 minutes';
{{< /highlight >}}

Остановить все запросы:
{{< highlight postgresql >}}
SELECT pg_terminate_backend(pg_stat_activity.pid)
FROM pg_stat_activity
WHERE datname = current_database()  
  AND pid <> pg_backend_pid();
{{< /highlight >}}

## Бэкапы

Для бэкапов есть консольные утилиты *pg_dump* и *pg_dumpall*. Первая делает sql-дамп отдельно взятой базы, а вторая, как следует из названия, делает то же самое для всех баз (+бэкапит пользователей). Есть еще утилита *pg_basebackup*, она делает полный бэкап инстанса на уровне файлов.

Дамп базы в сжатый файл (опция *-C* добавляет команду CREATE DATABASE в дамп):
{{< highlight console >}}
$pg_dump -C -U username -h hostname dbname | gzip > dump.sql.gz
{{< /highlight >}}

Полный дамп всех баз и пользователей в сжатый файл:
{{< highlight console >}}
$pg_dumpall -U username -h hostname | gzip > dump.sql.gz
{{< /highlight >}}

Восстановление базы (или полного дампа, но тогда не нужно указывать dbname) из сжатого файла:
{{< highlight console >}}
$zcat dump.sql.gz | psql -U username -h hostname dbname
{{< /highlight >}}

