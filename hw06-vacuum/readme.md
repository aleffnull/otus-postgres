Создадим базу данных и инициализируем её:
```
$ sudo -u postgres psql
[sudo] пароль для user: 
psql (18.7 (Debian 18.7-1.pgdg13+2))
Введите "help", чтобы получить справку.

postgres=# create database vacuum_test;
CREATE DATABASE
postgres=#

$ sudo -u postgres pgbench -i vacuum_test
dropping old tables...
ЗАМЕЧАНИЕ:  таблица "pgbench_accounts" не существует, пропускается
ЗАМЕЧАНИЕ:  таблица "pgbench_branches" не существует, пропускается
ЗАМЕЧАНИЕ:  таблица "pgbench_history" не существует, пропускается
ЗАМЕЧАНИЕ:  таблица "pgbench_tellers" не существует, пропускается
creating tables...
generating data (client-side)...
vacuuming...
creating primary keys...
done in 0.33 s (drop tables 0.01 s, create tables 0.02 s, client-side generate 0.16 s, vacuum 0.07 s, primary keys 0.07 s).
```

Выполним запуск бенчмарка:
```
$ sudo -u postgres pgbench -c 8 -P 6 -T 60 vacuum_test
pgbench (18.7 (Debian 18.7-1.pgdg13+2))
starting vacuum...end.
progress: 6.0 s, 153.3 tps, lat 51.561 ms stddev 34.858, 0 failed
progress: 12.0 s, 153.7 tps, lat 52.089 ms stddev 40.585, 0 failed
progress: 18.0 s, 155.2 tps, lat 51.510 ms stddev 34.619, 0 failed
progress: 24.0 s, 154.3 tps, lat 51.960 ms stddev 33.690, 0 failed
progress: 30.0 s, 154.5 tps, lat 51.657 ms stddev 33.686, 0 failed
progress: 36.0 s, 155.7 tps, lat 51.466 ms stddev 37.062, 0 failed
progress: 42.0 s, 154.2 tps, lat 51.814 ms stddev 39.063, 0 failed
progress: 48.0 s, 153.8 tps, lat 51.988 ms stddev 33.183, 0 failed
progress: 54.0 s, 154.2 tps, lat 51.940 ms stddev 37.843, 0 failed
progress: 60.0 s, 154.0 tps, lat 51.939 ms stddev 37.036, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 9265
number of failed transactions: 0 (0.000%)
latency average = 51.801 ms
latency stddev = 36.237 ms
initial connection time = 26.561 ms
tps = 154.325550 (without initial connection time)
```

Зафиксируем состояние пользовательских таблиц:
```
$ sudo -u postgres psql -d vacuum_test
psql (18.7 (Debian 18.7-1.pgdg13+2))
Введите "help", чтобы получить справку.

vacuum_test=# select relname, n_dead_tup, last_autovacuum, last_vacuum from pg_stat_user_tables order by 1;
     relname      | n_dead_tup |        last_autovacuum        |          last_vacuum
------------------+------------+-------------------------------+-------------------------------
 pgbench_accounts |       1838 | 2026-11-13 21:56:17.392767+03 | 2026-11-13 21:56:04.162824+03
 pgbench_branches |          0 | 2026-11-13 22:02:17.54964+03  | 2026-11-13 22:00:23.110831+03
 pgbench_history  |          0 | 2026-11-13 22:01:22.55672+03  | 2026-11-13 21:56:04.215025+03
 pgbench_tellers  |          0 | 2026-11-13 22:02:17.927329+03 | 2026-11-13 22:00:23.111742+03
(4 строки)
```

Создадим таблицу с текстовым полем и заполним ее:
```
vacuum_test=# create table text_table(id serial, str text);
CREATE TABLE
vacuum_test=# insert into text_table (str) select 'a' from generate_series(1, 1000000);
INSERT 0 1000000
```

Определим размер таблицы (35 Мб):
```
vacuum_test=# select pg_size_pretty(pg_total_relation_size('text_table'));
 pg_size_pretty 
----------------
 35 MB
(1 строка)
```

Мертвые строки в таблице отсутствуют, так как выполнялась только вставка строк:
```
vacuum_test=# select n_dead_tup from pg_stat_user_tables where relname = 'text_table';
 n_dead_tup
------------
          0
(1 строка)
```

Пять раз выполним обновление всех строк таблицы:
```
vacuum_test=# do $$
begin
    for i in 1..5 loop
        update text_table set str = str || 'b';
    end loop;
end
$$;
DO
```

В таблице 5 миллионов мертвых строк, автовакуум еще не запускался:
```
vacuum_test=# select n_dead_tup, last_autovacuum from pg_stat_user_tables where relname = 'text_table';
 n_dead_tup | last_autovacuum 
------------+-----------------
    5000000 | 
(1 строка)
```

Определим размер таблицы. Стало 230 Мб, таблица увеличилась в 6,5 раз:
```
vacuum_test=# select pg_size_pretty(pg_total_relation_size('text_table'));
 pg_size_pretty 
----------------
 230 MB
(1 строка)
```

Дождемся срабатывания автовакуума. Мертвые строки удалены:
```
vacuum_test=# select n_dead_tup, last_autovacuum from pg_stat_user_tables where relname = 'text_table';
 n_dead_tup |        last_autovacuum        
------------+-------------------------------
          0 | 2026-11-13 22:31:29.209208+03
(1 строка)
```

Размер таблицы не изменился, так как автовакуум не освободил ранее выделенное под мертвые строки место:
```
vacuum_test=# select pg_size_pretty(pg_total_relation_size('text_table'));
 pg_size_pretty 
----------------
 230 MB
(1 строка)
```

Отключим автовакуум для таблицы и выполним 10 обновлений строк:
```
vacuum_test=# alter table text_table set (autovacuum_enabled = false);
ALTER TABLE
vacuum_test=# do $$
begin
    for i in 1..10 loop
        update text_table set str = str || 'b';
    end loop;
end
$$;
DO
```

В таблице возникло 10 миллионов мертвых строк, новых запусков автовакуума не было.
```
vacuum_test=# select n_dead_tup, last_autovacuum from pg_stat_user_tables where relname = 'text_table';
 n_dead_tup |        last_autovacuum        
------------+-------------------------------
   10000000 | 2026-11-13 22:31:29.209208+03
(1 строка)
```

Новый размер таблицы 502 Мб:
```
vacuum_test=# select pg_size_pretty(pg_total_relation_size('text_table'));
 pg_size_pretty 
----------------
 502 MB
(1 строка)
```

При отключенном обслуживании операции редактирования приводят к увеличению размера таблицы, так как предыдущие версии строк становятся мертвыми и продолжают оставаться в таблице.

Включим автовакуум обратно и дождемся его срабатывания, мертвые строки удалены:
```
vacuum_test=# alter table text_table set (autovacuum_enabled = true);
ALTER TABLE
vacuum_test=# select n_dead_tup, last_autovacuum from pg_stat_user_tables where relname = 'text_table';
 n_dead_tup |        last_autovacuum        
------------+-------------------------------
          0 | 2026-11-13 22:48:56.402394+03
(1 строка)
```

Размер таблицы остался прежним:
```
vacuum_test=# select pg_size_pretty(pg_total_relation_size('text_table'));
 pg_size_pretty 
----------------
 502 MB
(1 строка)
```

Выполним полный вакуум для таблицы и проверим ее размер. Таблица уменьшилась, так как была переписана с актуальными строками:
```
vacuum_test=# vacuum full text_table;
VACUUM
vacuum_test=# select pg_size_pretty(pg_total_relation_size('text_table'));
 pg_size_pretty 
----------------
 50 MB
(1 строка)
```
