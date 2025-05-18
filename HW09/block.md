##### 1.	Настройте сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд. Воспроизведите ситуацию, при которой в журнале появятся такие сообщения.

Проверим установленные настройки:

```sql
postgres=# show log_lock_waits;
 log_lock_waits
----------------
 off
(1 row)

postgres=# show deadlock_timeout;
 deadlock_timeout
------------------
 1s
(1 row)
```

Установим новые параметры:

```sql
postgres=# alter system set log_lock_waits = on;
ALTER SYSTEM
postgres=# alter system set deadlock_timeout='200ms';
ALTER SYSTEM
postgres=# SELECT pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)

postgres=# show deadlock_timeout;
 deadlock_timeout
------------------
 200ms
(1 row)

postgres=# show log_lock_waits;
 log_lock_waits
----------------
 on
(1 row)
```

Создадим таблицу для опытов и заполним ее:

```sql
postgres=# create table test(id integer not null, name varchar(100), cnt integer);
CREATE TABLE
postgres=# insert into test(id, name, cnt) values (generate_series(1,100), 'User'||trunc(random()*1000), trunc(random()*100));
INSERT 0 100INSERT 0 100
```

1 Сессия

```sql
postgres=# select pg_backend_pid();
 pg_backend_pid
----------------
           6952
(1 row)

postgres=# begin;
BEGIN
postgres=*# UPDATE test set cnt=cnt+1 where id=1;
UPDATE
```

2 Сессия

```sql
postgres=# select pg_backend_pid();
 pg_backend_pid
----------------
           6957
(1 row)

postgres=# begin;
BEGIN
postgres=*# CREATE INDEX ON test(id);
```

Проверим блокировки:

```sql
postgres=*# SELECT locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted FROM pg_locks;
```

| locktype  | relation  | virtxid  | xid  | mode  | granted  |
| :------------ | :------------ | :------------ | :------------ | :------------ | :------------ |
| relation  | test_id_idx  |   |   | RowExclusiveLock  | t  |
| relation  | pg_locks |   |   | AccessShareLock  | t  |
| virtualxid  |   | 7/262  |   | ExclusiveLock  | t  |
| virtualxid  |   | 6/59 |   | ExclusiveLock  | t  |
| transactionid  |   |   | 744 | ExclusiveLock  | t  |
| relation  | test  |   |   | ShareLock  | f  |
| relation  | test  |   |   | RowExclusiveLock  | t  |


```sql
postgres=*# SELECT * FROM pg_locks;
```


| locktype    | database | relation | page | tuple | virtualxid | transactionid | classid | objid | objsubid | virtualtransaction | pid  |       mode       | granted | fastpath |           waitstart |
|:---------------|:----------|:----------|:------|:-------|:------------|:---------------|:---------|:-------|:----------|:--------------------|:------|:------------------|:---------|:----------|:-------------------------------|
 |relation      |        5 |    16394 |      |       |            |               |         |       |          | 7/262              | 6957 | RowExclusiveLock | t       | t        |
 |relation      |        5 |    12073 |      |       |            |               |         |       |          | 7/262              | 6957 | AccessShareLock  | t       | t        |
 |virtualxid    |          |          |      |       | 7/262      |               |         |       |          | 7/262              | 6957 | ExclusiveLock    | t       | t        |
 |virtualxid    |          |          |      |       | 6/59       |               |         |       |          | 6/59               | 6952 | ExclusiveLock    | t       | t        |
 |transactionid |          |          |      |       |            |           744 |         |       |          | 7/262              | 6957 | ExclusiveLock    | t       | f        |
 |relation      |        5 |    16391 |      |       |            |               |         |       |          | 6/59               | 6952 | ShareLock        | f       | f        | 2025-05-17 13:57:08.240601+00
 |relation      |        5 |    16391 |      |       |            |               |         |       |          | 7/262              | 6957 | RowExclusiveLock | t       | f        |


Из журнала:

```sql
sudo cat /var/log/postgresql/postgresql-15-main.log

2025-05-17 13:57:08.440 UTC [6952] postgres@postgres LOG:  process 6952 still waiting for ShareLock on relation 16391 of database 5 after 200.072 ms
2025-05-17 13:57:08.440 UTC [6952] postgres@postgres DETAIL:  Process holding the lock: 6957. Wait queue: 6952.
2025-05-17 13:57:08.440 UTC [6952] postgres@postgres STATEMENT:  CREATE INDEX ON test(id);
```

##### 2.	Смоделируйте ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах. Изучите возникшие блокировки в представлении pg_locks и убедитесь, что все они понятны. Пришлите список блокировок и объясните, что значит каждая.

В 3-х сессиях выполняю UPDATE:
```sql
postgres=# begin;
BEGIN
postgres=*# UPDATE test set cnt=cnt+100 where id=1;
```
Проверим блокировки
```sql
postgres=*# SELECT locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted FROM pg_locks;
```

|   locktype    |   relation   | virtxid | xid |       mode       | granted |
|:---------------|:--------------|:---------|:-----|:------------------|:---------|
 |relation      | test_id_idx1 |         |     | RowExclusiveLock | t
 |relation      | test_id_idx  |         |     | RowExclusiveLock | t
 |relation      | test         |         |     | RowExclusiveLock | t
 |virtualxid    |              | 8/140   |     | ExclusiveLock    | t
 |relation      | pg_locks     |         |     | AccessShareLock  | t
 |relation      | test_id_idx1 |         |     | RowExclusiveLock | t
 |relation      | test_id_idx  |         |     | RowExclusiveLock | t
 |relation      | test         |         |     | RowExclusiveLock | t
 |virtualxid    |              | 7/264   |     | ExclusiveLock    | t
 |relation      | test_id_idx1 |         |     | RowExclusiveLock | t
 |relation      | test_id_idx  |         |     | RowExclusiveLock | t
 |relation      | test         |         |     | RowExclusiveLock | t
 |virtualxid    |              | 6/61    |     | ExclusiveLock    | t
 |transactionid |              |         | 747 | ExclusiveLock    | t
 |tuple         | test         |         |     | ExclusiveLock    | t
 |transactionid |              |         | 748 | ExclusiveLock    | t
 |transactionid |              |         | 746 | ShareLock        | f
 |tuple         | test         |         |     | ExclusiveLock    | f
 |transactionid |              |         | 746 | ExclusiveLock    | t

Видно, что транзакция 746 не может получить запрошенную блокировку, статус f для режима ShareLock. 
ExclusiveLock - блокировка номера транзакции.
RowExclusiveLock - получение эксклюзивного доступа к строке, обычно для изменения. Получают UPDATE, DELETE, INSERT.
```sql
sudo cat /var/log/postgresql/postgresql-15-main.log

2025-05-17 14:10:35.427 UTC [989] LOG:  checkpoint starting: time
2025-05-17 14:10:37.156 UTC [989] LOG:  checkpoint complete: wrote 18 buffers (0.1%); 0 WAL file(s) added, 0 removed, 0 recycled; write=1.708 s,                       sync=0.008 s, total=1.729 s; sync files=18, longest=0.004 s, average=0.001 s; distance=77 kB, estimate=88 kB
2025-05-17 14:12:05.070 UTC [6952] postgres@postgres LOG:  process 6952 still waiting for ShareLock on transaction 746 after 200.074 ms
2025-05-17 14:12:05.070 UTC [6952] postgres@postgres DETAIL:  Process holding the lock: 6957. Wait queue: 6952.
2025-05-17 14:12:05.070 UTC [6952] postgres@postgres CONTEXT:  while updating tuple (0,101) in relation "test"
2025-05-17 14:12:05.070 UTC [6952] postgres@postgres STATEMENT:  UPDATE test set cnt=cnt+100 where id=1;
2025-05-17 14:12:31.100 UTC [7247] postgres@postgres LOG:  process 7247 still waiting for ExclusiveLock on tuple (0,101) of relation 16391 of database 5 after 200.094 ms
2025-05-17 14:12:31.100 UTC [7247] postgres@postgres DETAIL:  Process holding the lock: 6952. Wait queue: 7247.
2025-05-17 14:12:31.100 UTC [7247] postgres@postgres STATEMENT:  UPDATE test set cnt=cnt+100 where id=1;
2025-05-17 14:15:35.199 UTC [989] LOG:  checkpoint starting: time
2025-05-17 14:15:35.819 UTC [989] LOG:  checkpoint complete: wrote 6 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.602 s, sync=0.006 s, total=0.621 s; sync files=5, longest=0.005 s, average=0.002 s; distance=12 kB, estimate=80 kB
```


##### 3.	Воспроизведите взаимоблокировку трех транзакций. Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?3.	Воспроизведите взаимоблокировку трех транзакций. Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?

В трех сессиях выполняю:
```sql
create index on test(id);
```
Проверим блокировки

```sql
sudo cat /var/log/postgresql/postgresql-15-main.log

2025-05-17 14:32:06.616 UTC [6952] postgres@postgres LOG:  process 6952 still waiting for ShareLock on transaction 749 after 200.061 ms
2025-05-17 14:32:06.616 UTC [6952] postgres@postgres DETAIL:  Process holding the lock: 7247. Wait queue: 6952.
2025-05-17 14:32:06.616 UTC [6952] postgres@postgres CONTEXT:  while inserting index tuple (1,4) in relation "pg_class_relname_nsp_index"
2025-05-17 14:32:06.616 UTC [6952] postgres@postgres STATEMENT:  create index on test(id);
2025-05-17 14:32:24.087 UTC [6957] postgres@postgres LOG:  process 6957 still waiting for ShareLock on transaction 749 after 200.103 ms
2025-05-17 14:32:24.087 UTC [6957] postgres@postgres DETAIL:  Process holding the lock: 7247. Wait queue: 6952, 6957.
2025-05-17 14:32:24.087 UTC [6957] postgres@postgres CONTEXT:  while inserting index tuple (6,1) in relation "pg_class_relname_nsp_index"
2025-05-17 14:32:24.087 UTC [6957] postgres@postgres STATEMENT:  create index on test(id);
```

*В логах указывается полная информация о блокировке и операции/сессии которая ее вызвала. Вероятно ответ - Да.*

##### 4.	Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга?
*В данном случае не должно быть взаимоблокировки, т.к. пока не отработает первая транзакция, вторая не выполнится и будет ожидать завершения первой. Так как будет заблокирована все записи таблицы. Хотя и смущает задание со звездочкой, как будто при каких то настройках/условиях такое можно воспроизвести. *
