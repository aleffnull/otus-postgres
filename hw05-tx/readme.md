Создадим базу данных и таблицу в ней:
```
$ psql -h localhost -U postgres
Password for user postgres: 
psql (18.6 (Ubuntu 18.6-0ubuntu0.26.04.1))
Type "help" for help.

postgres=# create database test_tx;
CREATE DATABASE
postgres=# \c test_tx
You are now connected to database "test_tx" as user "postgres".
test_tx=# create table orders(id serial, created_at timestamptz, amount numeric);
CREATE TABLE
test_tx=# insert into orders(created_at, amount) values (now(), 10), (now(), 20);
INSERT 0 2
test_tx=# select * from orders;
 id |          created_at           | amount 
----+-------------------------------+--------
  1 | 2026-11-09 16:42:06.060624+00 |     10
  2 | 2026-11-09 16:42:06.060624+00 |     20
(2 rows)

test_tx=#
```

Откроем второе подключение к базе. Будем явно управлять транзакциями, отключать автокоммит не нужно. В каждой сессии проверим установленный уровень изоляции транзакций.

**Сессия 1**
```
test_tx= (1) # show transaction_isolation;
 transaction_isolation 
-----------------------
 read committed
(1 row)
```

**Сессия 2**
```
test_tx= (2) # show transaction_isolation;
 transaction_isolation 
-----------------------
 read committed
(1 row)
```

Во второй сессии начнем транзакцию и определим количество и сумму всех заказов. Транзакция читает два существующих заказа.

**Сессия 2**
```
test_tx= (2) # begin;
BEGIN
test_tx=* (2) # select count(*), sum(amount) from orders;
 count | sum 
-------+-----
     2 |  30
(1 row)
```

В сессии 1 добавим новый заказ.

**Сессия 1**
```
test_tx= (1) # begin;
BEGIN
test_tx=* (1) # insert into orders(created_at, amount) values (now(), 30);
INSERT 0 1
test_tx=* (1) # commit;
COMMIT
```

Повторим подсчет заказов во второй сессии. Транзакции стал доступен третий заказ, которого не было во время ее запуска. Произошло фантомное чтение, что является допустимым на текущем уровне изоляции транзакций read committed.

**Сессия 2**
```
test_tx=* (2) # select count(*), sum(amount) from orders;
 count | sum 
-------+-----
     3 |  60
(1 row)

test_tx=* (2) # commit;
COMMIT
```

Повторим сценарий, но установим уровень изоляции транзакций repeatable read. Во второй сессии определим количество и сумму всех заказов. Транзакция читает три существующих заказа.

**Сессия 2**
```
test_tx= (2) # begin transaction isolation level repeatable read;
BEGIN
test_tx=* (2) # select count(*), sum(amount) from orders;
 count | sum 
-------+-----
     3 |  60
(1 row)
```

В сессии 1 добавим новый заказ.

**Сессия 1**
```
test_tx= (1) # begin transaction isolation level repeatable read;
BEGIN
test_tx=* (1) # insert into orders(created_at, amount) values (now(), 40);
INSERT 0 1
test_tx=* (1) # commit;
COMMIT
```

Повторим подсчет заказов во второй сессии, транзакции по-прежнему доступны только три заказа.

**Сессия 2**
```
test_tx=* (2) # select count(*), sum(amount) from orders;
 count | sum 
-------+-----
     3 |  60
(1 row)

test_tx=* (2) # commit;
```

На уровне изоляции repeatable read фантомное чтение в postgres не допускается. Именно этот уровень следует использовать для построения отчета.
