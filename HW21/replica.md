##### 0. Создание виртуальных машин

Создал 3 ВМ 192.168.100.51-53
Разрешил подключение (и для репликации) из сети 192.168.100.*

 ![image](https://github.com/user-attachments/assets/72c33ae9-4688-4a64-9abf-33e0c28cceaa)

Создаем пользователя для репликации на каждом сервере

```sql
CREATE ROLE repic WITH LOGIN NOSUPERUSER NOCREATEDB NOCREATEROLE INHERIT REPLICATION NOBYPASSRLS CONNECTION LIMIT -1 PASSWORD 'Qwerty1';
```

Редактируем настройки на каждом сервере

```shell
sudo nano /var/lib/pgsql/16/data/postgresql.conf
```

```shell
listen_addresses = '*'
wal_level = logical
wal_log_hints = on

```

Перезапускаем сервер и проверяем жив ли:

![image](https://github.com/user-attachments/assets/e8bf4a85-8495-44b9-88bb-b5cdbe815f96)

 

##### 1.	На 1 ВМ создаем таблицы test для записи, test2 для запросов на чтение.

```sql
create TABLE test (id integer, nm text);
create TABLE test2 (id integer, nm text);
```

![image](https://github.com/user-attachments/assets/ba60e0e8-31ca-44da-8b1b-4a4af3df40b8)

Дадим права:
 
```sql
GRANT ALL ON TABLE public.test TO repic;
GRANT ALL ON TABLE public.test2 TO repic;
```

##### 2.	Создаем публикацию таблицы test и подписываемся на публикацию таблицы test2 с ВМ №2.

```sql
CREATE PUBLICATION public_test FOR TABLE test;
 ```

![image](https://github.com/user-attachments/assets/e414a5e1-8170-4b36-b657-5abd7055d48a)

```sql
CREATE SUBSCRIPTION subscribe_test2 CONNECTION 'hostaddr=192.168.100.52 port=5432 dbname=postgres user=repic password=Qwerty1 connect_timeout=10' PUBLICATION public_test2;
 ```

![image](https://github.com/user-attachments/assets/448aa9fc-b1e0-4ede-bd7b-9792ddc9d469)


##### 3.	На 2 ВМ создаем таблицы test2 для записи, test для запросов на чтение.


```sql
create TABLE test (id integer, nm text);
create TABLE test2 (id integer, nm text);
```


```sql
GRANT ALL ON TABLE public.test TO repic;
GRANT ALL ON TABLE public.test2 TO repic;
```


##### 4.	Создаем публикацию таблицы test2 и подписываемся на публикацию таблицы test1 с ВМ №1.

```sql
CREATE PUBLICATION public_test2 FOR TABLE test2;
```

![image](https://github.com/user-attachments/assets/b4bddc24-75d9-4800-a428-1349dc9c1963)


```sql
CREATE SUBSCRIPTION subscribe_test CONNECTION 'hostaddr=192.168.100.51 port=5432 dbname=postgres user=repic password=Qwerty1 connect_timeout=10' PUBLICATION public_test;
 ```

![image](https://github.com/user-attachments/assets/9f537090-c21e-470a-a5c2-3e94beb8e94c)


##### 5.	3 ВМ использовать как реплику для чтения и бэкапов (подписаться на таблицы из ВМ №1 и №2 ).


```sql
create TABLE test (id integer, nm text);
create TABLE test2 (id integer, nm text);
```


```sql
GRANT ALL ON TABLE public.test TO repic;
GRANT ALL ON TABLE public.test2 TO repic;
```

 ![image](https://github.com/user-attachments/assets/d10251c4-06b7-4c1b-b501-dadc5c5eb1e2)



```sql
CREATE SUBSCRIPTION subscribe_test_rep CONNECTION 'hostaddr=192.168.100.51 port=5432 dbname=postgres user=repic password=Qwerty1 connect_timeout=10' PUBLICATION public_test;
```

 ![image](https://github.com/user-attachments/assets/1c0d16f8-21a2-4328-ab9b-711f910fdd87)


```sql
CREATE SUBSCRIPTION subscribe_test2_rep CONNECTION 'hostaddr=192.168.100.52 port=5432 dbname=postgres user=repic password=Qwerty1 connect_timeout=10' PUBLICATION public_test2;
```

 ![image](https://github.com/user-attachments/assets/2a294376-da15-4c00-a639-3c9f8687a9e2)


```sql
GRANT ALL ON TABLE public.test TO repic;
GRANT ALL ON TABLE public.test2 TO repic;
```


Проверяем:


Сервер 1

```sql
insert into public.test values (1, 'Запись 1 сервер 1'), (2, 'Запись 2 сервер 1')
```

 ![image](https://github.com/user-attachments/assets/f663af19-ed67-4872-ba15-dba0d968b109)


Сервер 2 

```sql
insert into public.test2 values (1, 'Запись 1 сервер 2'), (2, 'Запись 2 сервер 2')
```
 
![image](https://github.com/user-attachments/assets/6491219a-f07e-4963-b74c-6bf576153230)


Содержимое таблиц:

Сервер1


```sql
select * from public.test2
```

 ![image](https://github.com/user-attachments/assets/89b78361-4a72-4310-a227-3c0da6510851)


Сервер 2


```sql
select * from public.test
```

![image](https://github.com/user-attachments/assets/9c427677-63f6-41e3-b469-d25bfce7ba7a)

 

Сервер 3


```sql
select * from public.test
```

![image](https://github.com/user-attachments/assets/b0d43228-75e1-40e4-869f-a7c58cdcb4b7)

 

```sql
select * from public.test2
```

![image](https://github.com/user-attachments/assets/7273da4f-41d2-44c5-a0ba-2fb2cf7c5a54)



Данные появились на всех серверах.

 
