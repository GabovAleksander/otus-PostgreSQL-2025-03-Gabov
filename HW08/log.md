#### 1.	Настройте выполнение контрольной точки раз в 30 секунд.
Проверим настройку периода создания контрольной точки:
```sql
postgres=# show checkpoint_timeout;
 checkpoint_timeout
--------------------
 5min
(1 row)
```

Установим период создания контрольной точки равный 30 секундам:

```sql
ALTER SYSTEM SET checkpoint_timeout = '30s';
ALTER SYSTEM
```

Перезагрузим postgresql и проверим значение параметра:
postgres=# show checkpoint_timeout;
 checkpoint_timeout
--------------------
 30s
(1 row)

#### 2.	10 минут c помощью утилиты pgbench подавайте нагрузку.


Подготовим БД

```sql
steel@otuspg:~$ pgbench -i -h localhost -p 5432 postgres -U postgres
Password:
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.02 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.29 s (drop tables 0.00 s, create tables 0.01 s, client-side generate 0.11 s, vacuum 0.04 s, primary keys 0.12 s).

postgres=# select pg_current_wal_lsn();
 pg_current_wal_lsn
--------------------
 0/21CF178
(1 row)
```

Нагрузка 10 минут:

```sql
steel@otuspg:~$ pgbench -c10 -P 60 -T 600 -h localhost -p 5432 -U postgres postgres
Password:
pgbench (15.12 (Ubuntu 15.12-1.pgdg24.10+1))
starting vacuum...end.
progress: 60.0 s, 683.2 tps, lat 14.604 ms stddev 11.326, 0 failed
progress: 120.0 s, 671.5 tps, lat 14.880 ms stddev 11.498, 0 failed
progress: 180.0 s, 630.0 tps, lat 15.864 ms stddev 12.912, 0 failed
progress: 240.0 s, 644.6 tps, lat 15.502 ms stddev 12.485, 0 failed
progress: 300.0 s, 648.2 tps, lat 15.417 ms stddev 12.617, 0 failed
progress: 360.0 s, 607.3 tps, lat 16.456 ms stddev 14.220, 0 failed
progress: 420.0 s, 629.4 tps, lat 15.873 ms stddev 13.267, 0 failed
progress: 480.0 s, 646.9 tps, lat 15.448 ms stddev 12.033, 0 failed
progress: 540.0 s, 642.9 tps, lat 15.544 ms stddev 12.546, 0 failed
progress: 600.0 s, 633.6 tps, lat 15.768 ms stddev 13.104, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 10
number of threads: 1
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 386275
number of failed transactions: 0 (0.000%)
latency average = 15.521 ms
latency stddev = 12.612 ms
initial connection time = 81.892 ms
tps = 643.848072 (without initial connection time)
```

#### 3.	Измерьте, какой объем журнальных файлов был сгенерирован за это время. Оцените, какой объем приходится в среднем на одну контрольную точку.####

```sql
postgres=# select pg_current_wal_lsn();
 pg_current_wal_lsn
--------------------
 0/1F9641B8
(1 row)


postgres=# select '0/1F9641B8'::pg_lsn - '0/21CF178'::pg_lsn as bytes;
   bytes
-----------
 494489664
(1 row)

**В соответствии с настройками контрольная точка создается каждые 30 секунд, соответственно за 10 минут будет создано 20 контрольных точек, соответственно каждая точка будет занимать 24MB.**
```

#### 4.	Проверьте данные статистики: все ли контрольные точки выполнялись точно по расписанию. Почему так произошло?


```sql
postgres=# SELECT * FROM pg_stat_bgwriter \gx
-[ RECORD 1 ]---------+------------------------------
checkpoints_timed     | 33
checkpoints_req       | 4
checkpoint_write_time | 592215
checkpoint_sync_time  | 411
buffers_checkpoint    | 45078
buffers_clean         | 0
maxwritten_clean      | 0
buffers_backend       | 4339
buffers_backend_fsync | 0
buffers_alloc         | 5809
stats_reset           | 2025-05-11 09:25:37.699503+00


steel@otuspg:~$ sudo tail -n 100 /var/log/postgresql/postgresql-15-main.log | grep checkpoint
2025-05-11 09:33:27.210 UTC [924] LOG:  checkpoint starting: shutdown immediate
2025-05-11 09:33:27.227 UTC [924] LOG:  checkpoint complete: wrote 3 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.005 s, sync=0.002 s, total=0.020 s; sync files=2, longest=0.002 s, average=0.001 s; distance=0 kB, estimate=0 kB
2025-05-11 09:36:52.087 UTC [1247] LOG:  checkpoint starting: shutdown immediate
2025-05-11 09:36:52.130 UTC [1247] LOG:  checkpoint complete: wrote 5 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.013 s, sync=0.010 s, total=0.049 s; sync files=4, longest=0.006 s, average=0.003 s; distance=2 kB, estimate=2 kB
2025-05-11 13:14:46.460 UTC [930] LOG:  checkpoint starting: shutdown immediate
2025-05-11 13:14:46.481 UTC [930] LOG:  checkpoint complete: wrote 3 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.006 s, sync=0.003 s, total=0.024 s; sync files=2, longest=0.002 s, average=0.002 s; distance=0 kB, estimate=0 kB
2025-05-11 13:15:22.430 UTC [1158] LOG:  checkpoint starting: time
2025-05-11 13:15:49.045 UTC [1158] LOG:  checkpoint complete: wrote 1709 buffers (10.4%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.587 s, sync=0.011 s, total=26.615 s; sync files=54, longest=0.003 s, average=0.001 s; distance=12824 kB, estimate=12824 kB
2025-05-11 13:16:22.059 UTC [1158] LOG:  checkpoint starting: time
2025-05-11 13:16:22.491 UTC [1158] LOG:  checkpoint complete: wrote 5 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.403 s, sync=0.006 s, total=0.432 s; sync files=5, longest=0.005 s, average=0.002 s; distance=9 kB, estimate=11543 kB
2025-05-11 13:16:52.494 UTC [1158] LOG:  checkpoint starting: time
2025-05-11 13:17:19.053 UTC [1158] LOG:  checkpoint complete: wrote 2128 buffers (13.0%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.480 s, sync=0.032 s, total=26.559 s; sync files=23, longest=0.007 s, average=0.002 s; distance=23705 kB, estimate=23705 kB
2025-05-11 13:17:22.056 UTC [1158] LOG:  checkpoint starting: time
2025-05-11 13:17:49.097 UTC [1158] LOG:  checkpoint complete: wrote 1965 buffers (12.0%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.976 s, sync=0.020 s, total=27.041 s; sync files=16, longest=0.007 s, average=0.002 s; distance=24526 kB, estimate=24526 kB
2025-05-11 13:17:52.100 UTC [1158] LOG:  checkpoint starting: time
2025-05-11 13:18:19.142 UTC [1158] LOG:  checkpoint complete: wrote 2045 buffers (12.5%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.975 s, sync=0.035 s, total=27.042 s; sync files=13, longest=0.011 s, average=0.003 s; distance=24658 kB, estimate=24658 kB
2025-05-11 13:18:22.145 UTC [1158] LOG:  checkpoint starting: time
2025-05-11 13:18:49.101 UTC [1158] LOG:  checkpoint complete: wrote 2195 buffers (13.4%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.879 s, sync=0.024 s, total=26.957 s; sync files=16, longest=0.007 s, average=0.002 s; distance=25219 kB, estimate=25219 kB
2025-05-11 13:18:52.105 UTC [1158] LOG:  checkpoint starting: time
2025-05-11 13:19:19.046 UTC [1158] LOG:  checkpoint complete: wrote 2039 buffers (12.4%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.876 s, sync=0.017 s, total=26.942 s; sync files=10, longest=0.006 s, average=0.002 s; distance=24481 kB, estimate=25146 kB
2025-05-11 13:19:22.049 UTC [1158] LOG:  checkpoint starting: time
2025-05-11 13:19:49.096 UTC [1158] LOG:  checkpoint complete: wrote 2126 buffers (13.0%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.982 s, sync=0.018 s, total=27.047 s; sync files=16, longest=0.007 s, average=0.002 s; distance=23831 kB, estimate=25014 kB
2025-05-11 13:19:52.099 UTC [1158] LOG:  checkpoint starting: time
2025-05-11 13:20:19.116 UTC [1158] LOG:  checkpoint complete: wrote 2026 buffers (12.4%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.977 s, sync=0.011 s, total=27.018 s; sync files=9, longest=0.006 s, average=0.002 s; distance=23860 kB, estimate=24899 kB
2025-05-11 13:20:22.119 UTC [1158] LOG:  checkpoint starting: time
2025-05-11 13:20:49.056 UTC [1158] LOG:  checkpoint complete: wrote 2110 buffers (12.9%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.875 s, sync=0.022 s, total=26.937 s; sync files=18, longest=0.006 s, average=0.002 s; distance=23815 kB, estimate=24790 kB
2025-05-11 13:20:52.056 UTC [1158] LOG:  checkpoint starting: time
2025-05-11 13:21:19.115 UTC [1158] LOG:  checkpoint complete: wrote 2012 buffers (12.3%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.988 s, sync=0.013 s, total=27.059 s; sync files=8, longest=0.006 s, average=0.002 s; distance=23542 kB, estimate=24666 kB
2025-05-11 13:21:22.117 UTC [1158] LOG:  checkpoint starting: time
2025-05-11 13:21:49.055 UTC [1158] LOG:  checkpoint complete: wrote 2116 buffers (12.9%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.878 s, sync=0.018 s, total=26.938 s; sync files=18, longest=0.006 s, average=0.001 s; distance=23586 kB, estimate=24558 kB
2025-05-11 13:21:52.058 UTC [1158] LOG:  checkpoint starting: time
2025-05-11 13:22:19.089 UTC [1158] LOG:  checkpoint complete: wrote 2015 buffers (12.3%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.973 s, sync=0.010 s, total=27.031 s; sync files=8, longest=0.005 s, average=0.002 s; distance=23142 kB, estimate=24416 kB
2025-05-11 13:22:22.091 UTC [1158] LOG:  checkpoint starting: time
2025-05-11 13:22:49.038 UTC [1158] LOG:  checkpoint complete: wrote 2085 buffers (12.7%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.882 s, sync=0.023 s, total=26.947 s; sync files=16, longest=0.008 s, average=0.002 s; distance=22295 kB, estimate=24204 kB
2025-05-11 13:22:52.041 UTC [1158] LOG:  checkpoint starting: time
2025-05-11 13:23:19.125 UTC [1158] LOG:  checkpoint complete: wrote 1992 buffers (12.2%); 0 WAL file(s) added, 0 removed, 2 recycled; write=27.025 s, sync=0.011 s, total=27.084 s; sync files=9, longest=0.005 s, average=0.002 s; distance=22923 kB, estimate=24076 kB
2025-05-11 13:23:22.128 UTC [1158] LOG:  checkpoint starting: time
2025-05-11 13:23:49.063 UTC [1158] LOG:  checkpoint complete: wrote 2090 buffers (12.8%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.879 s, sync=0.018 s, total=26.936 s; sync files=14, longest=0.005 s, average=0.002 s; distance=22939 kB, estimate=23962 kB
2025-05-11 13:23:52.066 UTC [1158] LOG:  checkpoint starting: time
2025-05-11 13:24:19.101 UTC [1158] LOG:  checkpoint complete: wrote 1987 buffers (12.1%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.978 s, sync=0.014 s, total=27.035 s; sync files=9, longest=0.006 s, average=0.002 s; distance=23028 kB, estimate=23869 kB
2025-05-11 13:24:22.104 UTC [1158] LOG:  checkpoint starting: time
2025-05-11 13:24:49.053 UTC [1158] LOG:  checkpoint complete: wrote 2316 buffers (14.1%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.887 s, sync=0.015 s, total=26.949 s; sync files=14, longest=0.005 s, average=0.002 s; distance=23227 kB, estimate=23805 kB
2025-05-11 13:24:52.055 UTC [1158] LOG:  checkpoint starting: time
2025-05-11 13:25:19.093 UTC [1158] LOG:  checkpoint complete: wrote 1969 buffers (12.0%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.980 s, sync=0.018 s, total=27.038 s; sync files=10, longest=0.009 s, average=0.002 s; distance=23006 kB, estimate=23725 kB
2025-05-11 13:25:22.096 UTC [1158] LOG:  checkpoint starting: time
2025-05-11 13:25:49.081 UTC [1158] LOG:  checkpoint complete: wrote 2097 buffers (12.8%); 0 WAL file(s) added, 0 removed, 2 recycled; write=26.887 s, sync=0.014 s, total=26.985 s; sync files=14, longest=0.007 s, average=0.001 s; distance=22871 kB, estimate=23639 kB
2025-05-11 13:25:52.083 UTC [1158] LOG:  checkpoint starting: time
2025-05-11 13:26:19.112 UTC [1158] LOG:  checkpoint complete: wrote 1950 buffers (11.9%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.975 s, sync=0.015 s, total=27.029 s; sync files=10, longest=0.007 s, average=0.002 s; distance=22731 kB, estimate=23548 kB
2025-05-11 13:26:22.115 UTC [1158] LOG:  checkpoint starting: time
2025-05-11 13:26:49.026 UTC [1158] LOG:  checkpoint complete: wrote 2321 buffers (14.2%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.880 s, sync=0.009 s, total=26.911 s; sync files=16, longest=0.003 s, average=0.001 s; distance=23065 kB, estimate=23500 kB
2025-05-11 13:27:22.043 UTC [1158] LOG:  checkpoint starting: time
2025-05-11 13:27:49.023 UTC [1158] LOG:  checkpoint complete: wrote 1818 buffers (11.1%); 0 WAL file(s) added, 0 removed, 1 recycled; write=26.959 s, sync=0.007 s, total=26.980 s; sync files=14, longest=0.003 s, average=0.001 s; distance=12439 kB, estimate=22394 kB


show checkpoint_completion_target;
 checkpoint_completion_target
------------------------------
 0.9
(1 row)
```

**Со значением 0.9, заданным по умолчанию, можно ожидать, что PostgreSQL завершит процедуру контрольной точки незадолго до следующей запланированной (примерно на 90% выполнения предыдущей контрольной точки), что соответствует 27 секундам.
Дополнительно были созданы 4 точки, о чем говорит checkpoints_req — по требованию (в том числе по достижению max_wal_size)**

#### 5.	Сравните tps в синхронном/асинхронном режиме утилитой pgbench. Объясните полученный результат.


```sql
postgres=# show synchronous_commit;
 synchronous_commit
--------------------
 on
(1 row)

postgres=# ALTER SYSTEM SET synchronous_commit = off;
ALTER SYSTEM
```

Перезагружаем кластер и проверяем установленный параметр:

```sql
postgres=# show synchronous_commit;
 synchronous_commit
--------------------
 off
(1 row)
```



```sql
postgres@otuspg:/home/steel$ pgbench -i postgres
dropping old tables...
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.09 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.24 s (drop tables 0.02 s, create tables 0.00 s, client-side generate 0.12 s, vacuum 0.03 s, primary keys 0.06 s).
```



```sql
postgres@otuspg:/home/steel$ pgbench -c10 -P 60 -T 600 -h localhost -p 5432 -U postgres postgres
Password:
pgbench (15.12 (Ubuntu 15.12-1.pgdg24.10+1))
starting vacuum...end.
progress: 60.0 s, 1538.8 tps, lat 6.479 ms stddev 4.167, 0 failed
progress: 120.0 s, 1519.5 tps, lat 6.572 ms stddev 4.162, 0 failed
progress: 180.0 s, 1581.9 tps, lat 6.313 ms stddev 3.975, 0 failed
progress: 240.0 s, 1602.6 tps, lat 6.232 ms stddev 3.890, 0 failed
progress: 300.0 s, 1571.9 tps, lat 6.353 ms stddev 3.983, 0 failed
progress: 360.0 s, 1565.1 tps, lat 6.380 ms stddev 3.991, 0 failed
progress: 420.0 s, 1559.2 tps, lat 6.405 ms stddev 4.022, 0 failed
progress: 480.0 s, 1560.1 tps, lat 6.401 ms stddev 3.969, 0 failed
progress: 540.0 s, 1584.9 tps, lat 6.300 ms stddev 3.931, 0 failed
progress: 600.0 s, 1549.1 tps, lat 6.447 ms stddev 4.072, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 10
number of threads: 1
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 937997
number of failed transactions: 0 (0.000%)
latency average = 6.387 ms
latency stddev = 4.017 ms
initial connection time = 88.090 ms
tps = 1563.504405 (without initial connection time)
```

**Производительность возросла, так как при асинхронном режиме мы получили 1563, а при синхронном 643. Вероятно связано с тем, что серверу не нужно ждать записи на диск, чтоб сообщить о результате транзакции (есть риск потери данных).**

#### 6.	Создайте новый кластер с включенной контрольной суммой страниц. Создайте таблицу. Вставьте несколько значений. Выключите кластер. Измените пару байт в таблице. Включите кластер и сделайте выборку из таблицы. Что и почему произошло? как проигнорировать ошибку и продолжить работу?6.	Создайте новый кластер с включенной контрольной суммой страниц. Создайте таблицу. Вставьте несколько значений. Выключите кластер. Измените пару байт в таблице. Включите кластер и сделайте выборку из таблицы. Что и почему произошло? как проигнорировать ошибку и продолжить работу?

```sql
steel@otuspg:~$ sudo pg_createcluster 15 chksms -- --data-checksums
Creating new PostgreSQL cluster 15/chksms ...
/usr/lib/postgresql/15/bin/initdb -D /var/lib/postgresql/15/chksms --auth-local peer --auth-host scram-sha-256 --no-instructions --data-checksums
The files belonging to this database system will be owned by user "postgres".
This user must also own the server process.

The database cluster will be initialized with locale "en_US.UTF-8".
The default database encoding has accordingly been set to "UTF8".
The default text search configuration will be set to "english".

Data page checksums are enabled.

fixing permissions on existing directory /var/lib/postgresql/15/chksms ... ok
creating subdirectories ... ok
selecting dynamic shared memory implementation ... posix
selecting default max_connections ... 100
selecting default shared_buffers ... 128MB
selecting default time zone ... Etc/UTC
creating configuration files ... ok
running bootstrap script ... ok
performing post-bootstrap initialization ... ok
syncing data to disk ... ok
Ver Cluster Port Status Owner    Data directory                Log file
15  chksms  5433 down   postgres /var/lib/postgresql/15/chksms /var/log/postgresql/postgresql-15-chksms.log


steel@otuspg:~$ pg_ctlcluster 15 chksms start
Warning: the cluster will not be running as a systemd service. Consider using systemctl:
  sudo systemctl start postgresql@15-chksms
Error: You must run this program as the cluster owner (postgres) or root
steel@otuspg:~$ sudo pg_ctlcluster 15 chksms start
steel@otuspg:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory                Log file
15  chksms  5433 online postgres /var/lib/postgresql/15/chksms /var/log/postgresql/postgresql-15-chksms.log
15  main    5432 online postgres /var/lib/postgresql/15/main   /var/log/postgresql/postgresql-15-main.log



postgres=# CREATE TABLE test(i int);
CREATE TABLE
postgres=# insert into test select s.id from generate_series(1, 500) as s(id);
INSERT 0 500
postgres=# select * from test limit 20;
 i
----
  1
  2
  3
  4
  5
  6
  7
  8
  9
 10
 11
 12
 13
 14
 15
 16
 17
 18
 19
 20
(20 rows)

steel@otuspg:~$ sudo systemctl stop postgresql@15-chksms
[sudo] password for steel:
steel@otuspg:~$ sudo pg_lsclusters
Ver Cluster Port Status Owner    Data directory                Log file
15  chksms  5433 down   postgres /var/lib/postgresql/15/chksms /var/log/postgresql/postgresql-15-chksms.log
15  main    5432 online postgres /var/lib/postgresql/15/main   /var/log/postgresql/postgresql-15-main.log



postgres=# SELECT pg_relation_filepath('test');
 pg_relation_filepath
----------------------
 base/5/16388
(1 row)

root@otuspg:/var/lib/postgresql/15/chksms/base/5# nano 16388

sudo systemctl start postgresql@15-chksms


postgres=# select * from test;
WARNING:  page verification failed, calculated checksum 22057 but expected 29697
ERROR:  invalid page in block 0 of relation base/5/16388


postgres=# set ignore_checksum_failure = on;
SET
postgres=# show ignore_checksum_failure;
 ignore_checksum_failure
-------------------------
 on
(1 row)

postgres=# select * from test;
WARNING:  page verification failed, calculated checksum 22057 but expected 29697
 i
---
(0 rows)
```


**Не смогли получить данные, ввиду того, что в контрольных суммах файла таблицы выявлены ошибки. Есть возможность включить параметр ignore_checksum_failure, в следствие чего будет выдано предупреждение, но запрос будет выполнен.
При большом объеме данных можем обнаружить повреждение страницы только при обращении к ней, при этом на длинной дистанции в бэкапах данная страница может быть так же повреждена.**

