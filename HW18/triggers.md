Функция для триггера вставки записей
```sql
CREATE OR REPLACE FUNCTION trg_goods_insert_to_sales ()
RETURNS trigger
AS
$trg$
DECLARE 
	goods_row goods%ROWTYPE;
BEGIN

		SELECT goods_id, good_name, good_price INTO goods_row FROM goods WHERE goods_id = NEW.good_id;
		UPDATE good_sum_mart SET sum_sale = sum_sale + goods_row.good_price * NEW.sales_qty WHERE good_sum_mart.good_name = goods_row.good_name;
		IF found THEN
			RETURN NEW;
		END IF;
 
       INSERT INTO good_sum_mart(good_name,sum_sale) VALUES(goods_row.good_name, goods_row.good_price * NEW.sales_qty);
       RETURN NEW;
END;
$trg$
	LANGUAGE plpgsql
    VOLATILE
    SET search_path = pract_functions, public
	SECURITY DEFINER;
```


Триггер вставки
```sql
CREATE TRIGGER trg_insert_sales 
AFTER INSERT
on sales
FOR EACH ROW
EXECUTE PROCEDURE trg_goods_insert_to_sales();
```


Функция для триггера изменения записей
```sql
CREATE OR REPLACE FUNCTION trg_goods_update_to_sales ()
RETURNS trigger
AS
$trg$
DECLARE 
	goods_row goods%ROWTYPE;
BEGIN
    SELECT goods_id, good_name, good_price INTO goods_row FROM goods WHERE goods_id = NEW.good_id;
	UPDATE good_sum_mart SET sum_sale = sum_sale + goods_row.good_price * (NEW.sales_qty - OLD.sales_qty)  
		WHERE good_sum_mart.good_name = goods_row.good_name;
	IF found THEN
		RETURN NEW;
	END IF;
	return null;
END;
$trg$
	LANGUAGE plpgsql
    VOLATILE
    SET search_path = pract_functions, public
	SECURITY DEFINER;
```

Триггер обновления
```sql
CREATE TRIGGER trg_update_sales 
AFTER UPDATE
on sales
FOR EACH ROW
EXECUTE PROCEDURE trg_goods_update_to_sales ();
```



Функция для триггера удаления записей
```sql
CREATE OR REPLACE FUNCTION trg_goods_delete_to_sales ()
RETURNS trigger
AS
$trg$
DECLARE 
	goods_row goods%ROWTYPE;
BEGIN
    SELECT * INTO goods_row FROM goods WHERE goods_id = OLD.good_id;
	UPDATE good_sum_mart SET sum_sale = sum_sale - goods_row.good_price * OLD.sales_qty  
		WHERE good_sum_mart.good_name = goods_row.good_name;
	IF found THEN
		RETURN OLD;
	END IF;
END;
$trg$
	LANGUAGE plpgsql
    VOLATILE
    SET search_path = pract_functions, public
	SECURITY DEFINER;
```


Триггер удаления
```sql
CREATE TRIGGER trg_delete_sales
AFTER DELETE
on sales
FOR EACH ROW
EXECUTE PROCEDURE trg_goods_delete_to_sales ();
```




```sql
create table temp1 as select * from sales;

truncate table sales;

insert into sales OVERRIDING SYSTEM VALUE select sales_id, good_id, sales_time, sales_qty from temp1

SELECT SETVAL('sales_sales_id_seq', MAX(sales_id)) FROM sales;

select * from sales
```
 ![image](https://github.com/user-attachments/assets/e6814f59-2543-406b-b0fb-5a42d0f1e013)



```sql
select * from good_sum_mart
```
![image](https://github.com/user-attachments/assets/2279ef4e-4061-44c0-83ff-ba740dd6a61a)

 

```sql
INSERT INTO goods (goods_id, good_name, good_price)
VALUES 	(3, 'Шоколад', 100.00),
(4, 'Процессор', 17000.00)


INSERT INTO sales (good_id, sales_qty) VALUES (3, 2), (4, 4), (3, 3), (4, 1);

select * from good_sum_mart
```
 ![image](https://github.com/user-attachments/assets/0d993168-f50e-4f88-9f2e-df4ef5dc79ad)



```sql
delete from sales where sales_id=5
select * from good_sum_mart
```
![image](https://github.com/user-attachments/assets/f6eed702-301d-4449-abf7-4aebc2337bda)

 

```sql
update sales set sales_qty=5 where sales_id=7
select * from good_sum_mart
```
![image](https://github.com/user-attachments/assets/437e4cc6-719e-45da-a566-f6660565630c)


 Чем такая схема (витрина+триггер) предпочтительнее отчета, создаваемого "по требованию" (кроме производительности)?
Подсказка: В реальной жизни возможны изменения цен.

После изменения цены триггер будет создавать записи уже с учетом актуальной цены, а в случае отчета по требованию, нам придется вести отдельно цены (доп таблица или история записей) в разрезе дат и усложнять запрос. 
