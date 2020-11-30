Домашнее задание
Работа с базами данных, пользователями и правами
Цель: 
* создание новой базы данных, схемы и таблицы
* создание роли для чтения данных из созданной схемы созданной базы данных
* создание роли для чтения и записи из созданной схемы созданной базы данных

1 создайте новый кластер PostgresSQL 13 (на выбор - GCE, CloudSQL)  
2 зайдите в созданный кластер под пользователем postgres  
3 создайте новую базу данных testdb  
```
postgres=# create database testdb;
CREATE DATABASE
```
4 зайдите в созданную базу данных под пользователем postgres  
```
postgres=# \c testdb;
You are now connected to database "testdb" as user "postgres".
```
5 создайте новую схему testnm  
```
testdb=# create schema testnm;  
CREATE SCHEMA
```
6 создайте новую таблицу t1 с одной колонкой c1 типа integer  
```
testdb=# create table testnm.t1 (c1 integer);
CREATE TABLE
```
7 вставьте строку со значением c1=1  
```
testdb=# insert into testnm.t1 select 1;
INSERT 0 1
```
8 создайте новую роль readonly  
```
testdb=# CREATE ROLE rouser LOGIN PASSWORD '123456';
CREATE ROLE
```
9 дайте новой роли право на подключение к базе данных testdb  
```
testdb=# GRANT CONNECT ON DATABASE testdb TO rouser;
GRANT
```
```
rsa-key-20201109@instance-1:~$ sudo nano /etc/postgresql/12/main/pg_hba.conf
# "local" is for Unix domain socket connections only
local   all             all                                  md5
...
rsa-key-20201109@instance-1:~$ sudo pg_ctlcluster 12 main restart
```

10 дайте новой роли право на использование схемы testnm  
```
testdb=# \c - rouser
Password for user rouser:
You are now connected to database "testdb" as user "rouser".
testdb=> \dt testnm.*
        List of relations
 Schema | Name | Type  |  Owner
--------+------+-------+----------
 testnm | t1   | table | postgres
(1 row)

testdb=> select * from testnm.t1;
ERROR:  permission denied for schema testnm
LINE 1: select * from testnm.t1;
                      ^
```
```
testdb=> \c - postgres
You are now connected to database "testdb" as user "postgres".
testdb=# GRANT USAGE ON SCHEMA testnm TO rouser;
GRANT
testdb=# \c - rouser
Password for user rouser:
You are now connected to database "testdb" as user "rouser".
testdb=> select * from testnm.t1;
ERROR:  permission denied for table t1
```
11 дайте новой роли право на select для всех таблиц схемы testnm  
```
testdb=> \c - postgres
You are now connected to database "testdb" as user "postgres".
testdb=# grant select on all tables in schema testnm to rouser;
GRANT
testdb=# \c - rouser
Password for user rouser:
You are now connected to database "testdb" as user "rouser".
testdb=> select * from testnm.t1;
 c1
----
  1
(1 row)
```
12 создайте пользователя testread с паролем test123  
```
postgres=# CREATE ROLE testread LOGIN PASSWORD 'test123';
CREATE ROLE
```
13 дайте поль readonly пользователю testread  
```
postgres=#  grant rouser to testread;
GRANT
```
14 зайдите под пользователем testread в базу данных testdb  
```
postgres=> \c testdb
You are now connected to database "testdb" as user "testread".
testdb=>
```
15 сделайте select * from t1;  
```
testdb=> select * from t1;
ERROR:  permission denied for table t1
```
16 получилось? (могло если вы делали сами не по шпаргалке и не упустили один существенный момент про который позже)  
```
На этом пункте могло получиться только в том случае, если бы не было подозрений, что в пункте 6 подразумевается "в схеме testnm". :) А поскольку именно это и напрашивалось, исходя из следующих пунктов, то rouser получил права на таблицу testnm.t1 и не беспокоился о том, что не может сделать выборку из publilc.t1. 
```
17 напишите что именно произошло в тексте домашнего задания  
18 у вас есть идеи почему? ведь права то дали?  
```
Пользователю rouser дали права на таблицы из схемы testnm, и только их testread и мог унаследовать. А запрашиваем таблицу из схемы public (поскольку схема явно не указана). Если запросим из схемы testnm, то все работает.
testdb=> select * from testnm.t1;
 c1
----
  1
(1 row)
```
19 посмотрите на список таблиц
```
testdb=> \dt
        List of relations
 Schema | Name | Type  |  Owner
--------+------+-------+----------
 public | t1   | table | postgres
(1 row)
```
20 подсказка в шпаргалке под пунктом 20  
21 а почему так получилось с таблицей (если делали сами и без шпаргалки то может у вас все нормально)  
22 вернитесь в базу данных testdb под пользователем postgres  
23 удалите таблицу t1  
24 создайте ее заново но уже с явным указанием имени схемы testnm
```
Так изначально и было сделано :) Профдеформация в связи с частой работой с плохо прописанными постановками задач. 
``` 
25 вставьте строку со значением c1=1  
26 зайдите под пользователем testread в базу данных testdb     
27 сделайте select * from testnm.t1;  
28 получилось?   
29 есть идеи почему? если нет - смотрите шпаргалку  
```
Все понятно :)
```
30 как сделать так чтобы такое больше не повторялось? если нет идей - смотрите шпаргалку
31 сделайте select * from testnm.t1;  
32 получилось?  
33 есть идеи почему? если нет - смотрите шпаргалку  
```
Пришлось заглянуть в шпаргалку :)
```
31 сделайте select * from testnm.t1;  
32 получилось?  
33 ура!  
34 теперь попробуйте выполнить команду create table t2(c1 integer); insert into t2 values (2);  
35 а как так? нам же никто прав на создание таблиц и insert в них под ролью readonly?  
```
Права на создание в public есть по умолчанию. Но нельзя делать, например, select по таблицам, созданным в этой схеме дургим пользователем.
```
36 есть идеи как убрать эти права? если нет - смотрите шпаргалку  
```
тоже пришлось заглянуть в подсказку.
Команда REVOKE лишает одну или несколько ролей прав, назначенных ранее. Ключевое слово PUBLIC обозначает неявно определённую группу всех ролей.
Т.о. мы запрещаем всем созданием в схеме public и забираем права на все, что уже было создано.  

testdb=# alter default privileges in schema testnm grant select on tables to rouser;
ALTER DEFAULT PRIVILEGES
testdb=# \c - testread
Password for user testread:
You are now connected to database "testdb" as user "testread".
testdb=> create table t2(c1 integer);
CREATE TABLE
testdb=> \c testdb postgres;
You are now connected to database "testdb" as user "postgres".
testdb=# revoke create on schema public from public;
REVOKE
testdb=# revoke all on database testdb from public;
REVOKE
testdb=# \c testdb testread;
Password for user testread:
You are now connected to database "testdb" as user "testread".
testdb=> create table t3(c1 integer);
ERROR:  permission denied for schema public
LINE 1: create table t3(c1 integer);
                     ^
```
38 теперь попробуйте выполнить команду create table t3(c1 integer); insert into t2 values (2);
39 расскажите что получилось и почему
