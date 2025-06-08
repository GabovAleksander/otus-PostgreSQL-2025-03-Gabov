### Ознакомьтесь с таблицами базы данных, особенно с таблицами bookings, tickets, ticket_flights, flights, boarding_passes, seats, airports, aircrafts.

Развернем дамп 

```sql
wget https://edu.postgrespro.ru/demo-big.zip
unzup demo-big.zip
psql -U postgres -h localhost -d postgres -f demo-big-20170815.sql
```

Посмотрим на размер таблиц и количество строк в них

```sql
SELECT schemaname,
       C.relname AS "relation",
       pg_size_pretty (pg_relation_size(C.oid)) as table,
       pg_size_pretty (pg_total_relation_size (C.oid)-pg_relation_size(C.oid)) as index,
       pg_size_pretty (pg_total_relation_size (C.oid)) as table_index,
       n_live_tup
FROM pg_class C
LEFT JOIN pg_namespace N ON (N.oid = C .relnamespace)
LEFT JOIN pg_stat_user_tables A ON C.relname = A.relname
WHERE nspname NOT IN ('pg_catalog', 'information_schema')
AND C.relkind <> 'i'
AND nspname !~ '^pg_toast'
ORDER BY pg_total_relation_size (C.oid) desc
```

![image](https://github.com/user-attachments/assets/908ed2a4-11f9-4f3c-978f-6c97fb70a85d)


###	Определите, какие данные в таблице bookings или других таблицах имеют логическую привязку к диапазонам, по которым можно провести секционирование (например, дата бронирования, рейсы).

Самая большая таблица по количеству записей ticket_flights, по схеме похоже, что основное поле для связи flight_id. Разобьем таблицу по хэшу для равномерного распределения данных по секциям.

![image](https://github.com/user-attachments/assets/adf38f2c-7aa1-4a5c-9e4d-3978692ada60)

```sql
explain analyze select * from ticket_flights where flight_id=14567
```
![image](https://github.com/user-attachments/assets/76e20fbc-e861-4300-afe3-3d63929100df)

производится перебор записей

Разобъем таблицу на секции по хэшу 

```sql
create table ticket_flights_replace (like ticket_flights including all) partition by hash(flight_id);
```

![image](https://github.com/user-attachments/assets/8a207df9-8635-4ca0-97c1-8eac1ac555a1)

Создадим 10 секций 


```sql
CREATE TABLE ticket_flights_replace_01 PARTITION OF ticket_flights_replace FOR VALUES WITH (MODULUS 10, REMAINDER 0);
CREATE TABLE ticket_flights_replace_02 PARTITION OF ticket_flights_replace FOR VALUES WITH (MODULUS 10, REMAINDER 1);
CREATE TABLE ticket_flights_replace_03 PARTITION OF ticket_flights_replace FOR VALUES WITH (MODULUS 10, REMAINDER 2);
CREATE TABLE ticket_flights_replace_04 PARTITION OF ticket_flights_replace FOR VALUES WITH (MODULUS 10, REMAINDER 3);
CREATE TABLE ticket_flights_replace_05 PARTITION OF ticket_flights_replace FOR VALUES WITH (MODULUS 10, REMAINDER 4);
CREATE TABLE ticket_flights_replace_06 PARTITION OF ticket_flights_replace FOR VALUES WITH (MODULUS 10, REMAINDER 5);
CREATE TABLE ticket_flights_replace_07 PARTITION OF ticket_flights_replace FOR VALUES WITH (MODULUS 10, REMAINDER 6);
CREATE TABLE ticket_flights_replace_08 PARTITION OF ticket_flights_replace FOR VALUES WITH (MODULUS 10, REMAINDER 7);
CREATE TABLE ticket_flights_replace_09 PARTITION OF ticket_flights_replace FOR VALUES WITH (MODULUS 10, REMAINDER 8);
CREATE TABLE ticket_flights_replace_10 PARTITION OF ticket_flights_replace FOR VALUES WITH (MODULUS 10, REMAINDER 9);
```

![image](https://github.com/user-attachments/assets/f5523243-08af-46da-ac55-750f80f50b34)

Перенесем данные из разбиваемой таблицы

```sql
insert into ticket_flights_replace select * from ticket_flights;
```

![image](https://github.com/user-attachments/assets/03524a29-19d3-45a9-bb6d-6f658ed72279)

Занимает довольно продолжительное время

Проверим как распределились данные:

```sql
SELECT schemaname,
       C.relname AS "relation",
       pg_size_pretty (pg_relation_size(C.oid)) as table,
       pg_size_pretty (pg_total_relation_size (C.oid)-pg_relation_size(C.oid)) as index,
       pg_size_pretty (pg_total_relation_size (C.oid)) as table_index,
       n_live_tup
FROM pg_class C
LEFT JOIN pg_namespace N ON (N.oid = C .relnamespace)
LEFT JOIN pg_stat_user_tables A ON C.relname = A.relname
WHERE nspname NOT IN ('pg_catalog', 'information_schema')
AND C.relkind <> 'i'
AND nspname !~ '^pg_toast'
ORDER BY pg_total_relation_size (C.oid) desc
```

![image](https://github.com/user-attachments/assets/59f04170-bf9f-4c6f-89ad-f86dfef59f42)

Данные по секциям распределились равномерно.

Проверим эффективность данного решения:

```sql
explain analyze select * from ticket_flights_replace where flight_id=14567
```

![image](https://github.com/user-attachments/assets/30880216-a841-4e7b-88ec-de34ab8fdc81)

Поиск производился только по секции ticket_flights_replace_07 и занял ~83 миллискунды против ~482 до разбиения.


Очевидно, что при большом приросте данных это не очень хорошее решение. Объем данных будет расти, а количество секций останется таким же. Возможно лучше бить таблицы по flight_id, учитывая, что он идет по порядку и таким образом мы можем отсечь в будущем исторические данные или перенести секции на более медленный носитель.


