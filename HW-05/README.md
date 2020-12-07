# Домашнее задание "Работа с журналами"
# Цель: 
* уметь работать с журналами и контрольными точками
* уметь настраивать параметры журналов


# 1. Настройте выполнение контрольной точки раз в 30 секунд.
## контрольная точка раз в 30 сек
```
 /etc/postgresql/13/main/postgresql.conf 
checkpoint_timeout = 30s              # range 30s-1d
#checkpoint_completion_target = 0.5     # checkpoint target duration, 0.0 - 1.0
```

## сбрасываем статистику
```
postgres=#  SELECT pg_stat_reset_shared('bgwriter');
postgres=# SELECT * FROM pg_stat_bgwriter \gx
-[ RECORD 1 ]---------+------------------------------
checkpoints_timed     | 0
checkpoints_req       | 0
checkpoint_write_time | 0
checkpoint_sync_time  | 0
buffers_checkpoint    | 0
buffers_clean         | 0
maxwritten_clean      | 0
buffers_backend       | 0
buffers_backend_fsync | 0
buffers_alloc         | 18
stats_reset           | 2020-12-06 12:25:37.560769+00
```

## замеряем объем журналов до начала нагрузки
```
rsa-key-20201109@instance-1:~$ sudo du -sh /var/lib/postgresql/13/main/pg_wal
33M     /var/lib/postgresql/13/main/pg_wal
```

# 2. 10 минут c помощью утилиты pgbench подавайте нагрузку.

## Запускаем нагрузку
```
postgres@instance-1:~$  pgbench -c8 -P 60 -T 600
starting vacuum...end.
progress: 60.0 s, 631.1 tps, lat 12.625 ms stddev 14.453
progress: 120.0 s, 632.7 tps, lat 12.600 ms stddev 21.003
progress: 180.0 s, 634.3 tps, lat 12.564 ms stddev 16.266
progress: 240.0 s, 624.4 tps, lat 12.766 ms stddev 14.250
progress: 300.0 s, 632.8 tps, lat 12.597 ms stddev 16.723
progress: 360.0 s, 611.0 tps, lat 13.047 ms stddev 13.412
progress: 420.0 s, 617.1 tps, lat 12.918 ms stddev 31.036
progress: 480.0 s, 643.5 tps, lat 12.387 ms stddev 14.250
progress: 540.0 s, 644.9 tps, lat 12.358 ms stddev 11.024
progress: 600.0 s, 658.3 tps, lat 12.109 ms stddev 9.321
number of transactions actually processed: 379819
latency average = 12.592 ms
latency stddev = 17.112 ms
tps = 633.002796 (including connections establishing)
tps = 633.004878 (excluding connections establishing)
```

# 3. Измерьте, какой объем журнальных файлов был сгенерирован за это время. Оцените, какой объем приходится в среднем на одну контрольную точку.
## объем журналов после нагрузки
```
rsa-key-20201109@instance-1:~$ sudo du -sh /var/lib/postgresql/13/main/pg_wal
65M     /var/lib/postgresql/13/main/pg_wal
```


Таким образом, выросли на ~32М. Если объем записей равен checkpoint_completion_target * 1, то на одну полную точку пришлось 2/3 объема, т.е. ~22 Мб.


# 4. Проверьте данные статистики: все ли контрольные точки выполнялись точно по расписанию. Почему так произошло?
## проверяем статистику контрольных точек
```
postgres=# SELECT * FROM pg_stat_bgwriter \gx
-[ RECORD 1 ]---------+------------------------------
checkpoints_timed     | 20
checkpoints_req       | 0
checkpoint_write_time | 327211
checkpoint_sync_time  | 347
buffers_checkpoint    | 45419
buffers_clean         | 0
maxwritten_clean      | 0
buffers_backend       | 3761
buffers_backend_fsync | 0
buffers_alloc         | 6013
stats_reset           | 2020-12-06 12:25:37.560769+00
```

## в т.ч. по логу
```
rsa-key-20201109@instance-1:~$ cat  /var/log/postgresql/postgresql-13-main.log | grep ' checkpoint starting: time'
2020-12-05 17:55:24.417 UTC [6944] LOG:  checkpoint starting: time
2020-12-05 17:55:54.113 UTC [6944] LOG:  checkpoint starting: time
2020-12-05 17:56:24.069 UTC [6944] LOG:  checkpoint starting: time
2020-12-05 17:56:54.125 UTC [6944] LOG:  checkpoint starting: time
2020-12-05 17:57:24.073 UTC [6944] LOG:  checkpoint starting: time
2020-12-05 17:57:54.109 UTC [6944] LOG:  checkpoint starting: time
2020-12-05 17:58:24.117 UTC [6944] LOG:  checkpoint starting: time
2020-12-05 17:58:54.061 UTC [6944] LOG:  checkpoint starting: time
2020-12-05 17:59:24.097 UTC [6944] LOG:  checkpoint starting: time
2020-12-05 17:59:54.053 UTC [6944] LOG:  checkpoint starting: time
2020-12-05 18:00:24.133 UTC [6944] LOG:  checkpoint starting: time
2020-12-05 18:00:54.097 UTC [6944] LOG:  checkpoint starting: time
2020-12-05 18:01:24.129 UTC [6944] LOG:  checkpoint starting: time
2020-12-05 18:01:54.073 UTC [6944] LOG:  checkpoint starting: time
2020-12-05 18:02:24.109 UTC [6944] LOG:  checkpoint starting: time
2020-12-05 18:02:54.045 UTC [6944] LOG:  checkpoint starting: time
2020-12-05 18:03:24.081 UTC [6944] LOG:  checkpoint starting: time
2020-12-05 18:03:54.113 UTC [6944] LOG:  checkpoint starting: time
2020-12-05 18:04:24.057 UTC [6944] LOG:  checkpoint starting: time
2020-12-05 18:04:54.113 UTC [6944] LOG:  checkpoint starting: time
2020-12-05 18:05:24.037 UTC [6944] LOG:  checkpoint starting: time
```


Таким образом, за 10 минут выполнилось столько точек, сколько и ожидалось, т.е. 20 контрольных точек. Наверное, исходя из вопроса, предполагалось, что могло быть больше. Но чтобы точки выполнялись внепланово, нужно, чтобы успел переполняться WAL (max_wal_size). По умолчанию размер 1Gb, а у нас даже до min_wal_size не дотянуло. Поэтому точки выполнялись по расписанию, каждая успевала за отведенный ей интервал (менее 15 сек). 


# 5. Сравните tps в синхронном/асинхронном режиме утилитой pgbench. Объясните полученный результат.

## проверяем режимы - синхронный и асинхронный

### выключаем fsync (off)
```
postgres@instance-1:~$  pgbench -c8 -P 60 -T 600
starting vacuum...end.
progress: 60.0 s, 1810.0 tps, lat 4.347 ms stddev 1.874
progress: 120.0 s, 1815.3 tps, lat 4.328 ms stddev 1.890
progress: 180.0 s, 922.3 tps, lat 8.519 ms stddev 23.075
progress: 240.0 s, 886.7 tps, lat 8.906 ms stddev 23.636
progress: 300.0 s, 916.2 tps, lat 8.575 ms stddev 23.148
progress: 360.0 s, 918.8 tps, lat 8.576 ms stddev 23.169
progress: 420.0 s, 915.9 tps, lat 8.565 ms stddev 23.135
progress: 480.0 s, 922.0 tps, lat 8.519 ms stddev 23.081
progress: 540.0 s, 906.0 tps, lat 8.700 ms stddev 23.361
progress: 600.0 s, 905.1 tps, lat 8.688 ms stddev 23.298
number of transactions actually processed: 655147
latency average = 7.206 ms
latency stddev = 19.134 ms
tps = 1091.844852 (including connections establishing)
tps = 1091.848427 (excluding connections establishing)
postgres@instance-1:~$
```

### включаем fsync (on), выключаем synchronous_commit (off)

```
postgres@instance-1:~$ pgbench -c8 -P 60 -T 600
starting vacuum...end.
progress: 60.0 s, 1573.6 tps, lat 5.042 ms stddev 1.822
progress: 120.0 s, 1593.6 tps, lat 4.978 ms stddev 1.809
progress: 180.0 s, 821.5 tps, lat 9.657 ms stddev 24.478
progress: 240.0 s, 814.9 tps, lat 9.741 ms stddev 24.568
progress: 300.0 s, 817.1 tps, lat 9.711 ms stddev 24.534
progress: 360.0 s, 806.7 tps, lat 9.830 ms stddev 24.688
progress: 420.0 s, 827.9 tps, lat 9.596 ms stddev 24.416
progress: 480.0 s, 817.6 tps, lat 9.693 ms stddev 24.516
progress: 540.0 s, 829.1 tps, lat 9.559 ms stddev 24.351
progress: 600.0 s, 812.3 tps, lat 9.737 ms stddev 24.530
number of transactions actually processed: 582870
latency average = 8.164 ms
latency stddev = 20.267 ms
tps = 971.423584 (including connections establishing)
tps = 971.426884 (excluding connections establishing)
```

Таким образом, асихронный режим по fsync дает больший выигрыш, чем по synchronous_commit (хотя и не столь существенный, как для синхронного режима). Но по документации второй режим считается более безопасным, к тому же, его можно выставлять для отдельной транзакции. Или наоборот, если все большинство данных не особо важные и часть важных, то можно synchronous_commit выключить глобально и включать для важных данных.  
Выигрыш обеспечивается за счет того, что мы не ждем, пока данные действительно запишутся на диск. Т.е. отправили и побежали работать дальше.


## **Непонятно: почему в первые 2 минуты наблюдается скачок. Повторилось как для fsync, так и для synchronous_commit. Для синхронного режима скачка нет.**


# 6. Создайте новый кластер с включенной контрольной суммой страниц. Создайте таблицу. Вставьте несколько значений. Выключите кластер. Измените пару байт в таблице. Включите кластер и сделайте выборку из таблицы. Что и почему произошло? как проигнорировать ошибку и продолжить работу?

## включаем контрольные суммы
```
rsa-key-20201109@instance-1:~$ sudo pg_ctlcluster 13 main stop
rsa-key-20201109@instance-1:~$ sudo su - postgres -c '/usr/lib/postgresql/13/bin/pg_checksums --enable -D "/var/lib/postgresql/13/main"'
Checksum operation completed
Files scanned:  931
Blocks scanned: 9237
pg_checksums: syncing data directory
pg_checksums: updating control file
Checksums enabled in cluster
rsa-key-20201109@instance-1:~$ sudo pg_ctlcluster 13 main start
rsa-key-20201109@instance-1:~$  pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
13  main    5432 online postgres /var/lib/postgresql/13/main /var/log/postgresql/postgresql-13-main.log
```

## создаем таблицу
```
postgres@instance-1:~$ psql
postgres=# SHOW data_checksums;
 data_checksums
----------------
 on
(1 row)

postgres=# create table t1(i1 integer);
CREATE TABLE
postgres=# insert into t1 select 1
postgres-# ;
INSERT 0 1
postgres=# insert into t1 select 2;
INSERT 0 1
postgres=# select * from t1;
 i1
----
  1
  2
(2 rows)
postgres=# SELECT pg_relation_filepath('t1');
 pg_relation_filepath
----------------------
 base/13414/16415
(1 row)
```

## ломаем файлик  /var/lib/postgresql/13/main/base/13414/16415
```
postgres@instance-1:~$  dd if=/dev/zero of=/var/lib/postgresql/13/main/base/13414/16415 oflag=dsync conv=notrunc bs=1 count=8
8+0 records in
8+0 records out
8 bytes copied, 0.0090488 s, 0.9 kB/s
```
postgres=# select * from t1;
WARNING:  page verification failed, calculated checksum 61581 but expected 24535
ERROR:  invalid page in block 0 of relation base/13414/16415
```

## чтобы продолжить работу с таблицей, можно попробовать проигнорировать проверку checksum
```
postgres=# SET ignore_checksum_failure = on;
postgres=# select * from t1;
WARNING:  page verification failed, calculated checksum 61581 but expected 24535
 i1
----
  1
  2
(2 rows)
```

```
Однако все зависит от того, где и что за сбой произошел. Например, если мы сломаем первый байт таблицы, то никакой ignore_checksum_failure нам не поможет. 
```
