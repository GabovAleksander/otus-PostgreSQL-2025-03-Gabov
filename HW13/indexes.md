#### Сгенерируем данные

Создадим и заполним таблицу с товарами

```sql
create table product(id serial primary key, name varchar);
insert into product (name) values ('Куртка');
insert into product (name) values ('Джемпер');
insert into product (name) values ('Футболка');
insert into product (name) values ('Пальто');
insert into product (name) values ('Майка');
insert into product (name) values ('Рубашка');
```

Создадим и заполним таблицу с цветами товаров

```sql
create table color(id serial primary key, name varchar);
insert into color (name) values ('Красный');
insert into color (name) values ('Черный');
insert into color (name) values ('Белый');
insert into color (name) values ('Синий');
insert into color (name) values ('Желтый');
insert into color (name) values ('Зеленый');
```

Создадим таблицу с продажами

```sql
create table sales(id serial primary key, product_id int, color_id int, summa money);
```
Вставим 1 млн строк данных

```sql
insert into sales(id, product_id, color_id, summa) 
select
a.n,
(select id	from product order by random()*a.n limit 1),
(select	id  from color order by	random()*a.n limit 1),
(select (random() * 9999*a.n)::decimal(16, 2))
from generate_series(1,1000000) as a(n)
```

Изменим текущее значение последовательности

```sql
SELECT SETVAL('sales_id_seq', MAX(id)) FROM sales;
```


Проверим данные

```sql
select
	s.id,
	p."name",
	c."name",
	s.summa
from
	sales s,
	color c ,
	product p
where
	s.color_id = c.id
	and s.product_id = p.id
order by s.id
```

![image](https://github.com/user-attachments/assets/01685be0-2c9f-476b-8531-6fa2ea148a6b)

 

### 1.	Создать индекс к какой-либо из таблиц вашей БД

Проверим план без индекса, найдем запись по идентификатору

```sql
explain (costs, verbose, format text, analyze)
select * from sales where id=13495
```
![image](https://github.com/user-attachments/assets/4b846b0f-02c0-4748-8271-f3d650cae8c9)

Видно что поиск производится с помощью перебора всех строк

Создадим индекс на поле идентификатор

```sql
CREATE INDEX idx_sales_id ON sales(id);
```
![image](https://github.com/user-attachments/assets/0b787ff4-c3b0-4419-a2f4-d68daa1a8052)


 

### 2.	Прислать текстом результат команды explain,
в которой используется данный индекс

```sql
explain (costs, verbose, format text, analyze)
select * from sales where id=13495
```

![image](https://github.com/user-attachments/assets/65db217e-d56f-4c58-bcf5-08f32943023c)

```sql
Index Scan using idx_sales_id on public.sales  (cost=0.42..8.44 rows=1 width=211) (actual time=0.017..0.019 rows=1 loops=1)
  Output: id, product_id, color_id, summa, description, description_tsvector
  Index Cond: (sales.id = 13495)
Planning Time: 0.061 ms
Execution Time: 0.031 ms

```

Скорость поиска значительно выросла и вместо перебора Seq Scan до создания индекса, происходит поиск по индексу Index scan.


### 3.	Реализовать индекс для полнотекстового поиска

Для этого добавим поле описание в таблицу sales, чтоб его заполнить создадим таблицу Dictionary с перечнем слов, заполним ее из файла csv в dbeaver:

```sql
create table dictionary(id serial primary key, word varchar);
```
![image](https://github.com/user-attachments/assets/9386b6be-bb46-4971-92ac-29a231cdf4a3)


Добавим поле описание

```sql
alter table sales add description varchar;
```

Заполним поле описание, сделаем функцию, потому что без нее select выполнялся 1 раз и во все ячейки вставлялось одинаковое значение. 

```sql
create function getDescription() returns varchar 
as $$
declare var varchar:='';
begin
select string_agg(a.word,' ') into var from (select 1 as grp, word from dictionary order by random() limit random()*20) a group by grp;
return var;
end;
$$
language plpgsql;
```


```sql
update sales set description=getDescription()
```
![image](https://github.com/user-attachments/assets/019a4ed6-62fa-4aed-aa94-037e1ca00579)


Для полнотекстового поиска создадим столбец description_tsvector с автоматическим заполнением

```sql
alter table sales add column description_tsvector TSVECTOR generated always
as (to_tsvector('russian',description)) stored;
```

![image](https://github.com/user-attachments/assets/189c1f79-43b5-4fdb-ba06-6c5516b6864f)

 
Создадим индекс полнотекстового поиска

```sql
create index idx_sales_descroption_tsvector ON sales USING gin(description_tsvector);
```
![image](https://github.com/user-attachments/assets/86883b1e-5d85-4bb2-a3f3-5719d4a094b3)

 

```sql
explain select id, description from sales where description_tsvector @@ to_tsquery('russian', 'первый & человек')
```
![image](https://github.com/user-attachments/assets/f948d4c8-47e2-4f52-80be-b7e8147e5441)

 
![image](https://github.com/user-attachments/assets/59036a58-3401-4008-bb0f-d966bee44be5)

 Используется Bitmap Scans который всегда состоит, минимум, из двух узлов. Сначала (на нижнем уровне) идет Bitmap Index Scan, а затем – Bitmap Heap Scan. 


### 4.	Реализовать индекс на часть таблицы или индекс на поле с функцией

Посмотрим план запроса без создания индекса

```sql
explain analyze 
select * from dictionary where length(word)=3
```
![image](https://github.com/user-attachments/assets/3a97522a-515a-4020-bed2-910af8ec5522)

Видно что происходит прямой перебор записей, что долго.
 

Создадим индекс с функцией вычисления длины слова:

```sql
create index idx_cnt_symbols on dictionary (length(word))
```
![image](https://github.com/user-attachments/assets/dc3bf25e-6abd-4349-80c4-ac7c0b7cf029)

 

```sql
explain analyze select * from dictionary where length(word)=15
```
![image](https://github.com/user-attachments/assets/63c76553-4563-46af-94ca-11926c953db2)

Посмотрим план аналогичного запроса, в котором видно, что снова используется Bitmap Scans и опять скорость выполнения запроса прилично выросла. 
 

```sql
select * from dictionary where length(word)=15
```
![image](https://github.com/user-attachments/assets/a3e88ce2-3d7a-4a30-91dd-921a17be4be4)

 

### 5.	Создать индекс на несколько полей

Посмотрим как выглядит план запроса без создания индекса:

```sql
explain analyze select id from sales where product_id=4 and color_id=3
```
![image](https://github.com/user-attachments/assets/c261c0ab-401f-4c7c-8e31-cc6eec57eb1e)

Снова происходит перебор всех записей.

 
Создадим индекс на 2 поля product_id и color_id 

```sql
create index idx_productId_colorId on sales(product_id, color_id)
```

Снова посмотрим план запроса:

```sql
explain analyze select * from sales where product_id=4 and color_id=3
```

 ![image](https://github.com/user-attachments/assets/393b47d3-c7bd-4aba-a32e-227dcc273d14)

Посмотрим план запроса, в котором видно, что опять используется Bitmap Scans и скорость выполнения запроса выше. 

```sql
select * from sales where product_id=4 and color_id=3
```

![image](https://github.com/user-attachments/assets/d65bb3ba-fda0-4fab-a651-77db561df992)

