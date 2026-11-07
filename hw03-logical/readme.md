Подключившись под пользователем `postgres`, создадим базу, схему и таблицу. Необходимо обратить внимание, что в соответствии с заданием таблица `t1` создается в схеме `public` вместо `test_nm`:
```sh
$ psql -h localhost -U postgres
Password for user postgres: 
psql (18.6 (Ubuntu 18.6-0ubuntu0.26.04.1))
Type "help" for help.

postgres=# create database test_db;
CREATE DATABASE
postgres=# \c test_db
You are now connected to database "test_db" as user "postgres".
test_db=# create schema test_nm;
CREATE SCHEMA
test_db=# create table t1(c1 int);
CREATE TABLE
test_db=# insert into t1 values (1), (2);
INSERT 0 2
test_db=#
```

Создадим роль и назначим ей привилегии:
```sh
test_db=# create role readonly;
CREATE ROLE
test_db=# grant connect on database test_db to readonly;
GRANT
test_db=# grant usage on schema test_nm to readonly;
GRANT
test_db=# grant select on all tables in schema test_nm to readonly;
GRANT
```

Создадим пользователя и назначим ему роль:
```sh
test_db=# create user test_read with password 'test123';
CREATE ROLE
test_db=# grant readonly to test_read;
GRANT ROLE
```

Подключимся под этим пользователем и попытаемся прочитать данные из таблицы. Поскольку роль не имеет прав на использование схемы `public`, чтение из таблицы невозможно:
```sh
$ psql -h localhost -d test_db -U test_read
Password for user test_read: 
psql (18.6 (Ubuntu 18.6-0ubuntu0.26.04.1))
Type "help" for help.

test_db=> select * from t1;
ERROR:  permission denied for table t1
test_db=> 
```

Под пользователем `postgres` переместим таблицу в нужную схему:
```sh
$ psql -h localhost -U postgres -d test_db
Password for user postgres: 
psql (18.6 (Ubuntu 18.6-0ubuntu0.26.04.1))
Type "help" for help.

test_db=# alter table t1 set schema test_nm;
ALTER TABLE
```

Снова попробуем выполнить чтение из таблицы под пользователем `test_read`. Доступа нет, так как при назначении на роль привилегии `select` она действовала на все существующие таблицы в схеме, а нашей таблицы в схеме еще не было:
```sh
$ psql -h localhost -d test_db -U test_read
Password for user test_read: 
psql (18.6 (Ubuntu 18.6-0ubuntu0.26.04.1))
Type "help" for help.

test_db=> select * from test_nm.t1;
ERROR:  permission denied for table t1
test_db=> 
```

Под пользователем `postgres` отредактируем привилегии:
```sh
$ psql -h localhost -U postgres -d test_db
Password for user postgres: 
psql (18.6 (Ubuntu 18.6-0ubuntu0.26.04.1))
Type "help" for help.

test_db=# grant select on table test_nm.t1 to readonly;
GRANT
test_db=#
```

Снова проверим доступ к таблице. Данные доступны:
```sh
$ psql -h localhost -d test_db -U test_read
Password for user test_read: 
psql (18.6 (Ubuntu 18.6-0ubuntu0.26.04.1))
Type "help" for help.

test_db=>  select * from test_nm.t1;
 c1 
----
  1
  2
(2 rows)

test_db=>
```

Попробуем под этим же пользователем создать новую таблицу или добавить данные в существующую. Доступа нет:
```sh
test_db=> create table test_nm.new_table(id int);
ERROR:  permission denied for schema test_nm
LINE 1: create table test_nm.new_table(id int);
                     ^
test_db=> insert into test_nm.t1 values(3);
ERROR:  permission denied for table t1
test_db=> 
```
