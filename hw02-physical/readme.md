Используем виртуальную машину, созданную в VirtualBox на базе Debian 13.7.

Установим свежую версию postgres. Для этого сначала установим пакет со скриптом, добавляющим репозиторий, запустим его и установим пакет с postgres 18:
```sh
$ sudo apt install postgresql-common
$ sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh
$ sudo apt install postgresql-18
```

Проверим, что созданный по умолчанию кластер запущен:
```sh
$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
18  main    5432 online postgres /var/lib/postgresql/18/main /var/log/postgresql/postgresql-18-main.log
```

Создадим таблицу с тестовыми данными:
```sh
$ sudo -u postgres psql
psql (18.6 (Debian 18.6-1.pgdg13+2))
Введите "help", чтобы получить справку.

postgres=# create table test(id int);
CREATE TABLE
postgres=# insert into test values (1), (2);
INSERT 0 2
postgres=#
```

Для добавления нового диска необходимо остановить ВМ. Разметим диск и создадим файловую систему:
```sh
$ sudo fdisk /dev/sdb
$ sudo mkfs.ext4 /dev/sdb1
```

Определим UUID раздела и добавим запись в `/etc/fstab`:
```sh
$ sudo blkid
/dev/sdb1: UUID="bc47a105-3c99-444c-bf0e-a3abe94e3639" BLOCK_SIZE="4096" TYPE="ext4" PARTUUID="8b8fece1-01"
$ sudo vim /etc/fstab
UUID=bc47a105-3c99-444c-bf0e-a3abe94e3639 /mnt/pg_data    ext4    defaults          0       2
```

Создадим точку монтирования, смонтируем диск и установим владельца:
```sh
$ sudo mkdir /mnt/pg_data
$ sudo mount /mnt/pg_data
$ sudo chown postgres:postgres /mnt/pg_data/
```

Остановим кластер и убедимся в его остановке:
```sh
$ sudo pg_ctlcluster 18 main stop
$ pg_lsclusters 
Ver Cluster Port Status Owner    Data directory              Log file
18  main    5432 down   postgres /var/lib/postgresql/18/main /var/log/postgresql/postgresql-18-main.log
```

Перенесем файлы с данными в новую папку:
```sh
$ sudo -u postgres mv /var/lib/postgresql/18 /mnt/pg_data/
```

При попытке запуска кластер не находит директорию с данными:
```sh
$ sudo pg_ctlcluster 18 main start
Error: /var/lib/postgresql/18/main is not accessible or does not exist
```

Укажем новую директорию в настройках кластера:
```sh
$ sudo vim /etc/postgresql/18/main/postgresql.conf
data_directory = '/mnt/pg_data/18/main'
```

Запустим кластер, проверим конфигурацию и содержимое тестовой таблицы:
```sh
$ sudo pg_ctlcluster 18 main start
$ sudo -u postgres psql
psql (18.6 (Debian 18.6-1.pgdg13+2))
Введите "help", чтобы получить справку.

postgres=# show data_directory;
    data_directory    
----------------------
 /mnt/pg_data/18/main
(1 строка)

postgres=# select * from test;
 id 
----
  1
  2
(2 строки)

postgres=#
```

Перезагрузим сервер и повторим проверку:
```sh
$ sudo reboot
$ sudo -u postgres psql
psql (18.6 (Debian 18.6-1.pgdg13+2))
Введите "help", чтобы получить справку.

postgres=# show data_directory;
    data_directory    
----------------------
 /mnt/pg_data/18/main
(1 строка)

postgres=# select * from test;
 id 
----
  1
  2
(2 строки)

postgres=#
```