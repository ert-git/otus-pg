# Домашнее задание Механизм блокировок
## Цель: понимать как работает механизм блокировок объектов и строк

## 1. Настройте сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд. Воспроизведите ситуацию, при которой в журнале появятся такие сообщения.
log_min_duration_statement=200


```
postgres=# show log_min_duration_statement ;
 log_min_duration_statement
----------------------------
 200ms
(1 row)

------------ 1st session --------
postgres=# begin;
postgres=*# update t1 set i1=11 where i1=33;
UPDATE 1
---------------------------------

------------ 2nd session --------
postgres=# begin;
postgres=*# update t1 set i1=0 where i1=33;
---------------------------------

------------ 1st session --------
postgres=*# commit;
COMMIT
---------------------------------

------------ logs ---------------
2020-12-06 14:30:54.744 UTC [4440] postgres@postgres LOG:  duration: 5217.932 ms  statement: update t1 set i1=0 where i1=33;
---------------------------------
```


## 2. Смоделируйте ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах. Изучите возникшие блокировки в представлении pg_locks и убедитесь, что все они понятны. Пришлите список блокировок и объясните, что значит каждая.

```
postgres=# select  locktype, relation::regclass, page , tuple , virtualxid as vxid , transactionid as tr,   virtualtransaction as vtr, pid  ,       mode       , granted as g, fastpath as fp from pg_locks order by pid;
   locktype    | relation | page | tuple | vxid |   tr    | vtr  | pid  |       mode       | g | fp
---------------+----------+------+-------+------+---------+------+------+------------------+---+----
 transactionid |          |      |       |      | 2005557 | 3/12 | 4440 | ExclusiveLock    | t | f
 relation      |    t1    |      |       |      |         | 3/12 | 4440 | RowExclusiveLock | t | t
 virtualxid    |          |      |       | 3/12 |         | 3/12 | 4440 | ExclusiveLock    | t | t

 transactionid |          |      |       |      | 2005557 | 4/10 | 4458 | ShareLock        | f | f
 virtualxid    |          |      |       | 4/10 |         | 4/10 | 4458 | ExclusiveLock    | t | t
 relation      |    t1    |      |       |      |         | 4/10 | 4458 | RowExclusiveLock | t | t
 transactionid |          |      |       |      | 2005558 | 4/10 | 4458 | ExclusiveLock    | t | f
 tuple         |    t1    |    0 |     6 |      |         | 4/10 | 4458 | ExclusiveLock    | t | f

 transactionid |          |      |       |      | 2005559 | 5/52 | 4686 | ExclusiveLock    | t | f
 relation      |    t1    |      |       |      |         | 5/52 | 4686 | RowExclusiveLock | t | t
 virtualxid    |          |      |       | 5/52 |         | 5/52 | 4686 | ExclusiveLock    | t | t
 tuple         |    t1    |    0 |     6 |      |         | 5/52 | 4686 | ExclusiveLock    | f | f

 virtualxid    |          |      |       | 6/8  |         | 6/8  | 4697 | ExclusiveLock    | t | t
 relation      | pg_locks |      |       |      |         | 6/8  | 4697 | AccessShareLock  | t | t
(14 rows)

```

* pid 4440 = первая транзакция (set i1=0). У нее установлен реальный идентификатор транзакции transactionid (флаг намерения изменить данные). Блокировка RowExclusiveLock - блокировка конкретной строки. 

* pid 4458 = вторая транзакция (set i1=1). Аналогично RowExclusiveLock и transactionid. Дополнительно ShareLock на идентификаторе блокирующей транзакции. Выставлен tuple - признак того, что эта транзакция "первая в очереди".

* pid 4686 = третья транзакция (set i1=2). Аналогично RowExclusiveLock и transactionid. Блокировка на tuple 2005558-ой транзакции.

Эту же информацию можно получить и так: 
```
select query, state, waiting, pid from pg_stat_activity where datname = 'postgres' and not (state = 'idle' or pid = pg_backend_pid());
-[ RECORD 1 ]----+--------------------------------
datid            | 13414
datname          | postgres
pid              | 4440
leader_pid       |
usesysid         | 10
usename          | postgres
backend_start    | 2020-12-06 14:27:40.115809+00
xact_start       | 2020-12-06 14:37:26.320055+00
query_start      | 2020-12-06 14:37:48.976511+00
state_change     | 2020-12-06 14:37:48.976744+00
wait_event_type  | Client
wait_event       | ClientRead
state            | idle in transaction
backend_xid      | 2005557
backend_xmin     |
query            | update t1 set i1=0 where i1=11;
backend_type     | client backend
-[ RECORD 2 ]----+--------------------------------
datid            | 13414
datname          | postgres
pid              | 4458
leader_pid       |
usesysid         | 10
backend_start    | 2020-12-06 14:28:33.562334+00
xact_start       | 2020-12-06 14:37:31.670821+00
query_start      | 2020-12-06 14:38:07.931761+00
state_change     | 2020-12-06 14:38:07.931764+00
wait_event_type  | Lock
wait_event       | transactionid
state            | active
backend_xid      | 2005558
backend_xmin     | 2005557
query            | update t1 set i1=1 where i1=11;
backend_type     | client backend
-[ RECORD 3 ]----+--------------------------------
datid            | 13414
datname          | postgres
pid              | 4686
backend_start    | 2020-12-06 14:37:00.802371+00
xact_start       | 2020-12-06 14:37:35.555083+00
query_start      | 2020-12-06 14:38:13.572665+00
state_change     | 2020-12-06 14:38:13.572668+00
wait_event_type  | Lock
wait_event       | tuple
state            | active
backend_xid      | 2005559
backend_xmin     | 2005557
query            | update t1 set i1=2 where i1=11;
backend_type     | client backend
```

Здесь видим
* какая версия строки участвует в транзакциях (xmin=2005557)
* в каком состоянии находятся транзакции (первая в процессе изменений, вторая ее ждет, третья ждет вторую)
* какой запрос выполнится и с каким pid
* время начала транзакции 


## 3. Воспроизведите взаимоблокировку трех транзакций. Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?

Создае три таблицы: t1, t2, t3.
Сессия 1 начинает транзакцию и обновляет строку №1 в t1;
Сессия 2 начинает транзакцию и обновляет строку №1 в t2;
Сессия 3 начинает транзакцию и обновляет строку №1 в t3;
Сессия 1 начинает транзакцию и обновляет строку №1 в t2; (блокируется сессией 2)
Сессия 3 начинает транзакцию и обновляет строку №1 в t1; (блокируется сессией 1)
Сессия 2 начинает транзакцию и обновляет строку №1 в t3; (блокируется сессией 3, которая заблокирована на сессии 1, которая ожидает сессию 2).
```
postgres=*# update t3 set i3=20 where i3=1;
ERROR:  deadlock detected
DETAIL:  Process 4686 waits for ShareLock on transaction 2005577; blocked by process 4458.
Process 4458 waits for ShareLock on transaction 2005575; blocked by process 4440.
Process 4440 waits for ShareLock on transaction 2005576; blocked by process 4686.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,1) in relation "t3"
```

В логе видим достаточно исчерпывающую информацию о том, кто кого и почему. Хотя к ее чтению нужна некоторая привычка. 
```
rsa-key-20201109@instance-1:~$ tail -f  /var/log/postgresql/postgresql-13-main.log

2020-12-06 15:55:47.133 UTC [4686] postgres@postgres ERROR:  deadlock detected
2020-12-06 15:55:47.133 UTC [4686] postgres@postgres DETAIL:  Process 4686 waits for ShareLock on transaction 2005577; blocked by process 4458.
        Process 4458 waits for ShareLock on transaction 2005575; blocked by process 4440.
        Process 4440 waits for ShareLock on transaction 2005576; blocked by process 4686.
        Process 4686: update t3 set i3=20 where i3=1;
        Process 4458: update t1 set i1=30 where i1=1;
        Process 4440: update t2 set i2=10 where i2=1;
2020-12-06 15:55:47.133 UTC [4686] postgres@postgres HINT:  See server log for query details.
2020-12-06 15:55:47.133 UTC [4686] postgres@postgres CONTEXT:  while updating tuple (0,1) in relation "t3"
2020-12-06 15:55:47.133 UTC [4686] postgres@postgres STATEMENT:  update t3 set i3=20 where i3=1;
2020-12-06 15:55:47.134 UTC [4440] postgres@postgres LOG:  duration: 28456.165 ms  statement: update t2 set i2=10 where i2=1;

```

## 4. Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга?
* Попробуйте воспроизвести такую ситуацию.

Документация говорит, что такое возможно, если для обновления будут построены разные планы выполнения. Т.е. если один процесс захватит блокировку на строке, которая должна быть следующей для другого процесса, а сам попытается прочитать запись, которую в данных момент удерживает этот второй процесс.  
Без where такая ситуация маловероятна.  
Как воспроизвести это, сама не догадалась, а перепечатывать документацию нечестно :)
