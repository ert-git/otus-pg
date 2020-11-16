# Домашнее задание
Работа с уровнями изоляции транзакции в PostgreSQL  
Цель: 
 * научится работать с Google Cloud Platform на уровне Google Compute Engine (IaaS)
 * научится управлять уровнем изолции транзации в PostgreSQL и понимать особенность работы уровней read commited и repeatable read  
 * **создать новый проект в Google Cloud Platform, например postgres2020-<yyyymmdd>, где yyyymmdd год, месяц и день вашего рождения (имя проекта должно быть уникально на уровне GCP)**
 * дать возможность доступа к этому проекту пользователю postgres202010@gmail.com с ролью Project Editor

```
 Создан проект postgres2020-19870109, GCP сообщил, что нотификация о предоставлении доступа отправлена.
```

 * **далее создать инстанс виртуальной машины Compute Engine с дефолтными параметрами**
 * добавить свой ssh ключ в GCE metadata
 * зайти удаленным ssh (первая сессия), не забывайте про ssh-add
```
Инстанс ВМ создан, получен доступ по ssh.
```
 * поставить PostgreSQL
```
PostgreSQL установлен через apt-get
```
 * зайти вторым ssh (вторая сессия)
 * запустить везде psql из под пользователя postgres
 * выключить auto commit

```sql
postgres=# \c test
You are now connected to database "test" as user "postgres".
test=# \set AUTOCOMMIT OFF
test=# \echo :AUTOCOMMIT
OFF
test=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)
```

 * сделать в первой сессии новую таблицу и наполнить ее данными
```sql
create table persons(id serial, first_name text, second_name text);
insert into persons(first_name, second_name) values('ivan', 'ivanov');
insert into persons(first_name, second_name) values('petr', 'petrov');
commit;
```

* **посмотреть текущий уровень изоляции: show transaction isolation level**  
```sql
test=# show transaction isolation level;
 transaction_isolation
-----------------------
 read committed
(1 row)
```

* **начать новую транзакцию в обеих сессиях с дефолтным (не меняя) уровнем изоляции**
** в первой сессии добавить новую запись
```sql
test=# insert into persons(first_name, second_name) values('sergey', 'sergeev');
INSERT 0 1
test=#
```

** сделать select * from persons во второй сессии
```sql
test=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)
```
** видите ли вы новую запись и если да то почему?
```
Нет, запись не видна, потому что в первой транзакции еще не сделан commit
```
** завершить первую транзакцию - commit;
```sql
test=# commit;
```

** сделать select * from persons во второй сессии
```sql
test=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```sql

** видите ли вы новую запись и если да то почему?
```
Да, запись видна, потому что на уровне read commited можно видеть изменения, сделанные завершенными транзакциями. Аномалия неповторяющегося чтения.
```

** завершите транзакцию во второй сессии
```sql
test=# commit;
```
* начать новые но уже repeatable read транзации - set transaction isolation level repeatable read;
```sql
test=# set transaction isolation level repeatable read;
```

** в первой сессии добавить новую запись
```sql
test=# insert into persons(first_name, second_name) values('sveta', 'svetova');
```

** сделать select * from persons во второй сессии
```sql
test=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```
** видите ли вы новую запись и если да то почему?
```
Нет, первая транзакция еще не завершена, на уровне repeatable read изменения, сделанные неазвершенными транзакциями, не видны.
```
** завершить первую транзакцию - commit;
** сделать select * from persons во второй сессии
```sql
test=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```

** видите ли вы новую запись и если да то почему?
```
Нет, запись не видна, потому что на этом уровне изоляции исключена аномалия неповторяющегося чтения. Транзакция видит то состояние БД, которое было на момент начала транзакции. 
```

- завершить вторую транзакцию
- сделать select * from persons во второй сессии
```sql
test=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
  4 | sveta      | svetova
(4 rows)
```

** видите ли вы новую запись и если да то почему?
```
Да, запись появилась, потому что запрос сделан в новой транзакции, которая начата после завершения транзакции, сделавшей изменения. Читающая транзакция получила новый снимок БД, в котором изменения уже видны всем.
```
** остановите виртуальную машину но не удаляйте ее
