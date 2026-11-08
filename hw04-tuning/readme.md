Кластер развернут на виртуальной машине с 2 Гб оперативной памяти и двумя ядрами процессора.

Создадим тестовую базу:
```sh
$ sudo -u postgres psql
[sudo] пароль для user: 
psql (18.6 (Debian 18.6-1.pgdg13+2))
Введите "help", чтобы получить справку.

postgres=# create database benchmark;
CREATE DATABASE
postgres=#
```

Инициализируем таблицы:
```sh
$ sudo -u postgres pgbench -i benchmark
dropping old tables...
ЗАМЕЧАНИЕ:  таблица "pgbench_accounts" не существует, пропускается
ЗАМЕЧАНИЕ:  таблица "pgbench_branches" не существует, пропускается
ЗАМЕЧАНИЕ:  таблица "pgbench_history" не существует, пропускается
ЗАМЕЧАНИЕ:  таблица "pgbench_tellers" не существует, пропускается
creating tables...
generating data (client-side)...
vacuuming...
creating primary keys...
done in 0.56 s (drop tables 0.00 s, create tables 0.03 s, client-side generate 0.37 s, vacuum 0.08 s, primary keys 0.07 s).
```

Выполним первый запуск бенчмарка (20 клиентов в 2 потока), получим 143 TPS:
```sh
$ sudo -u postgres pgbench -c 20 -j 2 -P 10 -T 60 benchmark
pgbench (18.6 (Debian 18.6-1.pgdg13+2))
starting vacuum...end.
progress: 10.0 s, 141.8 tps, lat 138.167 ms stddev 139.702, 0 failed
progress: 20.0 s, 145.5 tps, lat 137.788 ms stddev 152.429, 0 failed
progress: 30.0 s, 143.3 tps, lat 138.259 ms stddev 150.832, 0 failed
progress: 40.0 s, 142.2 tps, lat 142.283 ms stddev 150.589, 0 failed
progress: 50.0 s, 145.3 tps, lat 137.847 ms stddev 136.131, 0 failed
progress: 60.0 s, 139.2 tps, lat 143.081 ms stddev 149.880, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 20
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 8593
number of failed transactions: 0 (0.000%)
latency average = 139.709 ms
latency stddev = 146.704 ms
initial connection time = 54.994 ms
tps = 142.948657 (without initial connection time)
```

Для подбора параметров используем сервис https://pgtune.fariton.ru/. С его помощью получим следующие значения параметров:
```properties
# Настройки памяти.
shared_buffers = 512MB
effective_cache_size = 1536MB
work_mem = 16MB

# Машина развернута на SSD.
random_page_cost = 1.1
effective_io_concurrency = 200
```

Изменим настройки кластера и перезапустим его:
```sh
$ sudo -u postgres psql
[sudo] пароль для user: 
psql (18.6 (Debian 18.6-1.pgdg13+2))
Введите "help", чтобы получить справку.

postgres=# ALTER SYSTEM SET shared_buffers = '512MB';
ALTER SYSTEM
postgres=# ALTER SYSTEM SET effective_cache_size = '1536MB';
ALTER SYSTEM
postgres=# ALTER SYSTEM SET work_mem = '16MB';
ALTER SYSTEM
postgres=# ALTER SYSTEM SET random_page_cost = '1.1';
ALTER SYSTEM
postgres=# ALTER SYSTEM SET effective_io_concurrency = '200';
ALTER SYSTEM
postgres=#
$ sudo pg_ctlcluster 18 main restart
$ pg_lsclusters 
Ver Cluster Port Status Owner    Data directory       Log file
18  main    5432 online postgres /mnt/pg_data/18/main /var/log/postgresql/postgresql-18-main.log
```

Повторим запуск бенчмарка, получим 161 TPS:
```sh
$ sudo -u postgres pgbench -c 20 -j 2 -P 10 -T 60 benchmark
pgbench (18.6 (Debian 18.6-1.pgdg13+2))
starting vacuum...end.
progress: 10.0 s, 135.8 tps, lat 143.897 ms stddev 152.760, 0 failed
progress: 20.0 s, 143.4 tps, lat 139.921 ms stddev 142.804, 0 failed
progress: 30.0 s, 151.6 tps, lat 131.828 ms stddev 124.028, 0 failed
progress: 40.0 s, 161.8 tps, lat 123.828 ms stddev 133.286, 0 failed
progress: 50.0 s, 203.8 tps, lat 98.277 ms stddev 106.164, 0 failed
progress: 60.0 s, 171.2 tps, lat 115.964 ms stddev 107.948, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 20
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 9696
number of failed transactions: 0 (0.000%)
latency average = 123.827 ms
latency stddev = 128.445 ms
initial connection time = 57.851 ms
tps = 161.272343 (without initial connection time)
```

Для достижения максимальной производительности перейдем на асинхронную работу с диском:
```sh
$ sudo -u postgres psql
psql (18.6 (Debian 18.6-1.pgdg13+2))
Введите "help", чтобы получить справку.

postgres=# ALTER SYSTEM SET synchronous_commit = 'off';
ALTER SYSTEM
postgres=# ALTER SYSTEM SET fsync = 'off';
ALTER SYSTEM
```

Повторим замеры, получим 1835 TPS:
```sh
$ sudo -u postgres pgbench -c 20 -j 2 -P 10 -T 60 benchmark
pgbench (18.6 (Debian 18.6-1.pgdg13+2))
starting vacuum...end.
progress: 10.0 s, 1803.8 tps, lat 11.023 ms stddev 9.650, 0 failed
progress: 20.0 s, 1826.6 tps, lat 10.942 ms stddev 9.402, 0 failed
progress: 30.0 s, 1824.4 tps, lat 10.963 ms stddev 9.615, 0 failed
progress: 40.0 s, 1867.4 tps, lat 10.713 ms stddev 9.648, 0 failed
progress: 50.0 s, 1857.4 tps, lat 10.764 ms stddev 9.615, 0 failed
progress: 60.0 s, 1827.1 tps, lat 10.944 ms stddev 9.476, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 20
number of threads: 2
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 110085
number of failed transactions: 0 (0.000%)
latency average = 10.894 ms
latency stddev = 9.576 ms
initial connection time = 50.487 ms
tps = 1834.841161 (without initial connection time)
```
