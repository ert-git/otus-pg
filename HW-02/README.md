# Домашнее задание
Установка и настройка PostgteSQL в контейнере Docker

## Цель: 
* создавать дополнительный диск для уже существующей виртуальной машины, размечать его и делать на нем файловую систему
* переносить содержимое базы данных PostgreSQL на дополнительный диск
* переносить содержимое БД PostgreSQL между виртуальными машинами
* установить PostgreSQL в Docker контейнере
* настроить контейнер для внешнего подключения

## 1 вариант:
* создайте виртуальную машину c Ubuntu 20.04 LTS (bionic) в GCE типа e2-medium в default VPC в любом регионе и зоне, например us-central1-a

```
использован инстанс из предыдущего ДЗ
```

* поставьте на нее PostgreSQL через sudo apt

* проверьте что кластер запущен через sudo -u postgres pg_lsclusters

```
rsa-key-20201109@instance-1:~$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
12  main    5432 online postgres /var/lib/postgresql/12/main /var/log/postgresql/postgresql-12-main.log
```

* зайдите из под пользователя postgres в psql и сделайте произвольную таблицу с произвольным содержимым
```
postgres=# create table test(c1 text);
postgres=# insert into test values('1');
\q
```

```
rsa-key-20201109@instance-1:~$ sudo su - postgres
postgres@instance-1:~$ psql
psql (12.4 (Ubuntu 12.4-0ubuntu0.20.04.1))
Type "help" for help.

postgres=# \c test
You are now connected to database "test" as user "postgres".
test=# create table test(c1 text);
CREATE TABLE
test=# insert into test values('1');
INSERT 0 1
```

* остановите postgres например через sudo -u postgres pg_ctlcluster 13 main stop
```
rsa-key-20201109@instance-1:~$ sudo -u postgres pg_ctlcluster 12 main stop
rsa-key-20201109@instance-1:~$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
12  main    5432 down   postgres /var/lib/postgresql/12/main /var/log/postgresql/postgresql-12-main.log
```

* создайте новый standard persistent диск GKE через Compute Engine -> Disks в том же регионе и зоне что GCE инстанс размером например 10GB
* добавьте свеже-созданный диск к виртуальной машине - надо зайти в режим ее редактирования и дальше выбрать пункт attach existing disk
* проинициализируйте диск согласно инструкции и подмонтировать файловую систему, только не забывайте менять имя диска на актуальное, в вашем случае это скорее всего будет /dev/sdb - https://www.digitalocean.com/community/tutorials/how-to-partition-and-format-storage-devices-in-linux

```
Создан персистентынй раздел:

rsa-key-20201109@instance-1:~$ df -h -x tmpfs -x devtmpfs
Filesystem      Size  Used Avail Use% Mounted on
/dev/root       9.6G  1.9G  7.7G  20% /
...
/dev/sdb1       9.8G   37M  9.3G   1% /mnt/data
```

* сделайте пользователя postgres владельцем /mnt/data - chown -R postgres:postgres /mnt/data/
```
rsa-key-20201109@instance-1:~$ sudo chown -R postgres:postgres /mnt/data/
rsa-key-20201109@instance-1:~$ ll /mnt/
total 12
drwxr-xr-x  3 root     root     4096 Nov 18 17:00 ./
drwxr-xr-x 19 root     root     4096 Nov 18 17:05 ../
drwxr-xr-x  3 postgres postgres 4096 Nov 18 16:59 data/
```

* перенесите содержимое /var/lib/postgres/13 в /mnt/data - mv /var/lib/postgresql/13 /mnt/data
```
rsa-key-20201109@instance-1:~$ sudo mv /var/lib/postgresql/13 /mnt/data
rsa-key-20201109@instance-1:~$ sudo ls -la /mnt/data
total 28
drwxr-xr-x  4 postgres postgres  4096 Nov 18 17:10 .
drwxr-xr-x  3 root     root      4096 Nov 18 17:00 ..
drwx------  2 postgres postgres 16384 Nov 18 16:59 lost+found
drwx------ 19 postgres postgres  4096 Nov 18 17:07 12
```

* попытайтесь запустить кластер - sudo -u postgres pg_ctlcluster 13 main start
* напишите получилось или нет и почему
```
rsa-key-20201109@instance-1:~$ sudo -u postgres pg_ctlcluster 12 main start
Error: /var/lib/postgresql/12/main is not accessible or does not exist

Кластер не запустился, потому что мы переместили его каталог с данными и не сказали ему об этом.
```

* задание: найти конфигурационный параметр в файлах раположенных в /etc/postgresql/10/main который надо поменять и поменяйте его
```
Настройка каталога находится в файле
/etc/postgresql/12/main/postgresql.conf
```

* напишите что и почему поменяли
```
Поменяли путь к каталогу данных

less /etc/postgresql/12/main/postgresql.conf

#data_directory = '/var/lib/postgresql/12/main'         # use data in another directory
data_directory = '/mnt/data/12/main'         
```

* попытайтесь запустить кластер - sudo -u postgres pg_ctlcluster 13 main start
* напишите получилось или нет и почему
```
rsa-key-20201109@instance-1:~$ sudo -u postgres pg_ctlcluster 12 main start
rsa-key-20201109@instance-1:~$ sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory    Log file
12  main    5432 online postgres /mnt/data/12/main /var/log/postgresql/postgresql-12-main.log

Кластер запустился в новом каталоге
```

* зайдите через через psql и проверьте содержимое ранее созданной таблицы
```
Таблица на месте:

test=# \d
               List of relations
 Schema |      Name      |   Type   |  Owner
--------+----------------+----------+----------
 public | persons        | table    | postgres
 public | persons_id_seq | sequence | postgres
 public | test           | table    | postgres
(3 rows)

test=# select * from test;
 c1
----
 1
(1 row)
```

* задание со звездочкой: не удаляя существующий GCE инстанс сделайте новый, поставьте на его PostgreSQL, удалите файлы с данными из /var/lib/postgres, перемонтируйте внешний диск который сделали ранее от первой виртуальной машины ко второй и запустите PostgreSQL на второй машине так чтобы он работал с данными на внешнем диске, расскажите как вы это сделали и что в итоге получилось.

```
Не сделано
```

## 2 вариант:
* сделать в GCE инстанс с Ubuntu 20.04
* поставить на нем Docker Engine
```
https://docs.docker.com/engine/install/ubuntu/
sudo apt-get update
sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common
-- Add Docker’s official GPG key:
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo apt-key fingerprint 0EBFCD88

--Use the following command to set up the stable repository
sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"

-- Install Docker Engine
-- Update the apt package index, and install the latest version of Docker Engine and containerd, or go to the next step to install a specific version:

sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io

```

* сделать каталог /var/lib/postgres
```
sudo mkdir /var/lib/postgres
```
* развернуть контейнер с PostgreSQL 13 смонтировав в него /var/lib/postgres

```
rsa-key-20201109@instance-1:~$ sudo  docker run --name pg-docker --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5433:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:13
eeb5e87806fcda4e8b1a1bc6a79349433e02fe5c91c239e48cc2ed2e253c7de6
rsa-key-20201109@instance-1:~$ sudo docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES
eeb5e87806fc        postgres:13         "docker-entrypoint.s…"   7 seconds ago       Up 6 seconds        0.0.0.0:5433->5432/tcp   pg-docker
```

* развернуть контейнер с клиентом postgres
```
rsa-key-20201109@instance-1:~$ sudo docker run -it --rm --network pg-net --name pg-client postgres:13 psql -h pg-docker -U postgres
Password for user postgres:
psql (13.1 (Debian 13.1-1.pgdg100+1))
Type "help" for help.

postgres=# 
```

* подключится из контейнера с клиентом к контейнеру с сервером и сделать
таблицу с парой строк

```
postgres=# create database test;
CREATE DATABASE
postgres=# \c test
You are now connected to database "test" as user "postgres".
test=# create table test(c1 text);
CREATE TABLE
test=# insert into test values('hello');
INSERT 0 1
test=# insert into test values('world');
INSERT 0 1
test=# select * from test;
  c1
-------
 hello
 world
(2 rows)
```

* подключится к контейнеру с сервером с ноутбука/компьютера извне инстансов GCP
```
Для этого необходимо разрешить доступ в pg_hba.conf:
host     all     all      0.0.0.0/0     md5
```

* удалить контейнер с сервером
```
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES
eeb5e87806fc        postgres:13         "docker-entrypoint.s…"   8 minutes ago       Up 8 minutes        0.0.0.0:5433->5432/tcp   pg-docker
rsa-key-20201109@instance-1:~$ sudo docker stop eeb5e87806fc
eeb5e87806fc
rsa-key-20201109@instance-1:~$ sudo docker rm eeb5e87806fc
eeb5e87806fc
rsa-key-20201109@instance-1:~$ sudo docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
```
* создать его заново
```
 sudo  docker run --name pg-docker --network pg-net -e POSTGRES_PASSWORD=postgres -d  -v /var/lib/postgres:/var/lib/postgresql/data postgres:13
```

* подключится снова из контейнера с клиентом к контейнеру с сервером

```
rsa-key-20201109@instance-1:~$ sudo docker run -it --rm --network pg-net --name pg-client postgres:13 psql -h pg-docker -U postgres
Password for user postgres:
psql (13.1 (Debian 13.1-1.pgdg100+1))
Type "help" for help.
postgres=# \c test

```
* проверить, что данные остались на месте
```
You are now connected to database "test" as user "postgres".
test=# \d
        List of relations
 Schema | Name | Type  |  Owner
--------+------+-------+----------
 public | test | table | postgres
(1 row)
```

