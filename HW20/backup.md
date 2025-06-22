##### 1.	Создаем ВМ/докер c ПГ.1.	Создаем ВМ/докер c ПГ.
##### 2.	Создаем БД, схему и в ней таблицу.
```sql
create database backuptest;
create schema backuphw;
set search_path to backuphw, public;
```

##### 3.	Заполним таблицы автосгенерированными 100 записями.
```sql
create table testtable1 as
select
	generate_series(1,100) as id,
	md5(random()::text)::char(100) as text;
```
 
![image](https://github.com/user-attachments/assets/76202f32-6e09-450e-8555-41e1f8855f4d)

##### 4.	Под линукс пользователем Postgres создадим каталог для бэкапов
```sql
steel@otuspg:/var/lib/postgresql/15$ mkdir backup
```

##### 5.	Сделаем логический бэкап используя утилиту COPY
```sql
COPY testtable1 TO '/var/lib/postgresql/15/backup/logic_backup.sql';
```
 ![image](https://github.com/user-attachments/assets/a0d1d008-617a-461f-b543-22cc8cb1f0c2)


##### 6.	Восстановим в 2 таблицу данные из бэкапа.
```sql
create table testtable2(id integer, varchar text);
```

```sql
COPY testtable2 FROM '/var/lib/postgresql/15/backup/logic_backup.sql';
```

 ![image](https://github.com/user-attachments/assets/522d225c-741a-41a5-b005-7c194f747e7c)


Проверим наличие данных в обеих таблицах:


```sql
select * from testtable1 t1 left join testtable2 t2 on t1.id=t2.id
```

![image](https://github.com/user-attachments/assets/530a87d9-f017-4469-ae2d-8713e694cd15)

 
##### 7.	Используя утилиту pg_dump создадим бэкап в кастомном сжатом формате двух таблиц

```sql
steel@otuspg:/var/lib/postgresql/15/backup$ sudo -u postgres -p 5432 pg_dump -d backuptest -C -U postgres -Fc > /var/lib/postgresql/15/backup/backup_pgdump.gz
```


##### 8.	Используя утилиту pg_restore восстановим в новую БД только вторую таблицу!

```sql
drop table testtable2
```

 ![image](https://github.com/user-attachments/assets/91eff24f-d265-44f7-83a2-dc792e04b0da)


```sql
steel@otuspg:/var/lib/postgresql/15/backup$ sudo -u postgres -p 5432 pg_restore -n backuphw -t testtable2 -d backuptest < /var/lib/postgresql/15/backup/backup_pgdump.gz
```

Проверим наличие данных в востановленной таблице:

```sql
select * from testtable2
```
 
![image](https://github.com/user-attachments/assets/5296eded-230d-4db7-a07c-51d9ba926342)

