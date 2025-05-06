•	Создать инстанс ВМ с 2 ядрами и 4 Гб ОЗУ и SSD 10GB

•	Установить на него PostgreSQL 15 с дефолтными настройками

•	Создать БД для тестов: выполнить pgbench -i postgres

```sql
    pgbench -i -h localhost -p 5432 -U postgres postgres 
    steel@otuspg:~$ pgbench -i -h localhost -p 5432 -U postgres postgres
    Password:
    dropping old tables...
    NOTICE:  table "pgbench_accounts" does not exist, skipping
    NOTICE:  table "pgbench_branches" does not exist, skipping
    NOTICE:  table "pgbench_history" does not exist, skipping
    NOTICE:  table "pgbench_tellers" does not exist, skipping
    creating tables...
    generating data (client-side)...
    100000 of 100000 tuples (100%) done (elapsed 0.03 s, remaining 0.00 s)
    vacuuming...
    creating primary keys...
    done in 0.30 s (drop tables 0.00 s, create tables 0.02 s, client-side generate 0.19 s, vacuum 0.04 s, primary keys 0.06 s).

```

•	Запустить pgbench -c8 -P 6 -T 60 -U postgres postgres
pgbench -h localhost -p 5432 -c 8 -P 6 -T 60 -U postgres postgres

```sql
steel@otuspg:~$ pgbench -h localhost -p 5432 -c 8 -P 6 -T 60 -U postgres postgres
Password:
pgbench (15.12 (Ubuntu 15.12-1.pgdg24.10+1))
starting vacuum...end.
progress: 6.0 s, 703.8 tps, lat 11.195 ms stddev 7.667, 0 failed
progress: 12.0 s, 690.3 tps, lat 11.577 ms stddev 8.113, 0 failed
progress: 18.0 s, 688.3 tps, lat 11.616 ms stddev 8.209, 0 failed
progress: 24.0 s, 679.8 tps, lat 11.758 ms stddev 8.567, 0 failed
progress: 30.0 s, 703.5 tps, lat 11.349 ms stddev 7.742, 0 failed
progress: 36.0 s, 678.7 tps, lat 11.793 ms stddev 8.447, 0 failed
progress: 42.0 s, 690.8 tps, lat 11.571 ms stddev 8.141, 0 failed
progress: 48.0 s, 681.0 tps, lat 11.735 ms stddev 8.672, 0 failed
progress: 54.0 s, 702.2 tps, lat 11.383 ms stddev 7.718, 0 failed
progress: 60.0 s, 693.8 tps, lat 11.523 ms stddev 8.167, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 41482
number of failed transactions: 0 (0.000%)
latency average = 11.549 ms
latency stddev = 8.150 ms
initial connection time = 76.444 ms
tps = 692.013411 (without initial connection time)
```


•	Применить параметры настройки PostgreSQL из прикрепленного к материалам занятия файла


```sql
alter system set max_connections to "40";
alter system set shared_buffers to "1024MB";
alter system set effective_cache_size to "3GB";
alter system set maintenance_work_mem to "512MB";
alter system set checkpoint_completion_target  to "0.9";
alter system set wal_buffers  to "16MB";
alter system set default_statistics_target  to "500";
alter system set random_page_cost  to "4";
alter system set effective_io_concurrency  to "2";
alter system set work_mem  to "6553kB";
alter system set min_wal_size to "4GB";
alter system set max_wal_size to "16GB";
```



•	Протестировать заново
```sql
steel@otuspg:~$ pgbench -h localhost -p 5432 -c 8 -P 6 -T 60 -U postgres postgres
Password:
pgbench (15.12 (Ubuntu 15.12-1.pgdg24.10+1))
starting vacuum...end.
progress: 6.0 s, 678.0 tps, lat 11.631 ms stddev 8.153, 0 failed
progress: 12.0 s, 689.0 tps, lat 11.600 ms stddev 7.734, 0 failed
progress: 18.0 s, 698.3 tps, lat 11.446 ms stddev 7.650, 0 failed
progress: 24.0 s, 695.0 tps, lat 11.506 ms stddev 7.833, 0 failed
progress: 30.0 s, 704.3 tps, lat 11.348 ms stddev 7.779, 0 failed
progress: 36.0 s, 708.7 tps, lat 11.277 ms stddev 7.748, 0 failed
progress: 42.0 s, 697.2 tps, lat 11.458 ms stddev 7.820, 0 failed
progress: 48.0 s, 695.2 tps, lat 11.471 ms stddev 7.729, 0 failed
progress: 54.0 s, 704.0 tps, lat 11.391 ms stddev 7.855, 0 failed
progress: 60.0 s, 694.3 tps, lat 11.511 ms stddev 7.482, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 41792
number of failed transactions: 0 (0.000%)
latency average = 11.464 ms
latency stddev = 7.780 ms
initial connection time = 73.864 ms
tps = 697.148846 (without initial connection time)
```


•	Что изменилось и почему?

Увеличился tps, хотя как будто в рамках погрешности.  При нашем объеме данных и количестве подключений изменяемые параметры особо не должны влиять. Возможно повлияло увеличение work_mem, wal_buffers, shared_buffers

•	Создать таблицу с текстовым полем и заполнить случайными или сгенерированными данным в размере 1млн строк

```sql
create table test(str text);
INSERT INTO test(str) SELECT 'string' FROM generate_series(1,1000000);
```

•	Посмотреть размер файла с таблицей

```sql
SELECT pg_size_pretty(pg_TABLE_size('test'))
```
Размер = 35MB

•	5 раз обновить все строчки и добавить к каждой строчке любой символ
```sql
update test set str = CONCAT(str, 'o');
update test set str = CONCAT(str, 'o');
update test set str = CONCAT(str, 'o');
update test set str = CONCAT(str, 'o');
update test set str = CONCAT(str, 'o');
```


•	Посмотреть количество мертвых строчек в таблице и когда последний раз приходил автовакуум

```sql
SELECT relname, n_live_tup, n_dead_tup, last_autovacuum FROM pg_stat_user_TABLEs  WHERE relname = 'test'
```


| Таблица | Живых строк | Мертвых строк | Дата последнего автовакуума |
| ------------- | ------------- | ------------- | ------------- |
| Тест | 1000000 | 5000000 | 2025-05-05 20:54:31.306 +0300 |


•	Подождать некоторое время, проверяя, пришел ли автовакуум


| Таблица | Живых строк | Мертвых строк | Дата последнего автовакуума |
| ------------- | ------------- | ------------- | ------------- |
| test  | 1000000  | 0  |  2025-05-05 20:56:31.435 +0300 |

**Автовакуум прошел**

•	5 раз обновить все строчки и добавить к каждой строчке любой символ

```sql
update test set str = CONCAT(str, 'o');
update test set str = CONCAT(str, 'o');
update test set str = CONCAT(str, 'o');
update test set str = CONCAT(str, 'o');
update test set str = CONCAT(str, 'o');
```


•	Посмотреть размер файла с таблицей

```sql
SELECT pg_size_pretty(pg_TABLE_size('test'))
```
Размер = 261MB

•	Отключить Автовакуум на конкретной таблице

```sql
ALTER TABLE test SET (autovacuum_enabled = off);
```

•	10 раз обновить все строчки и добавить к каждой строчке любой символ

```sql
update test set str = CONCAT(str, 'o');
update test set str = CONCAT(str, 'o');
….
update test set str = CONCAT(str, 'o');
update test set str = CONCAT(str, 'o');
```



•	Посмотреть размер файла с таблицей

Размер = 571MB


•	Объясните полученный результат

**При каждом апдейте строки создается новая запись и при выключенном автовакууме удаления неактуальных записей не происходит.**

•	Не забудьте включить автовакуум)

```sql
ALTER TABLE test SET (autovacuum_enabled = on);
```
