
# 1. Текущий уровень изоляции: read committed

1) видите ли вы новую запись и если да то почему?
   
нет, потому что при текущем уровне изоляции в сессии 2 не видны не закоммиченные изменения транзакции сессии 1

2) видите ли вы новую запись и если да то почему?
   
Видно, потому что при текущем уровне изоляции в сессии 2 видны закоммиченные изменения транзакции сессии 1

&nbsp;

# 2. Переключаем на уровень  изоляции repeatable read

## Кейс 1

Сессия 1:	

```
begin
set transaction isolation level repeatable read
insert into persons(first_name, second_name) values('sveta', 'svetova');
```

Сессия 2:

```
begin
 set transaction isolation level repeatable read
 select * from persons
```
видите ли вы новую запись и если да то почему?

нет, потому что при текущем уровне изоляции в сессии 2 не видны любые изменения транзакции сессии 1 до завершения трензакции сессии 2

&nbsp;

## Кейс 2

Сессия 1:		

```
begin
set transaction isolation level repeatable read
insert into persons(first_name, second_name) values('sveta', 'svetova');
commit;
end
```

Сессия 2:

```
begin
 set transaction isolation level repeatable read
 select * from persons
```

видите ли вы новую запись и если да то почему?

нет, потому что при текущем уровне изоляции в сессии 2 не видны любые изменения транзакции сессии 1 до завершения трензакции сессии 2

&nbsp;

## Кейс 3

Сессия 1:	

```
begin
set transaction isolation level repeatable read
insert into persons(first_name, second_name) values('sveta', 'svetova');
commit;
end
```

Сессия 2:

```
begin
 set transaction isolation level repeatable read
 select * from persons
end
begin
set transaction isolation level repeatable read
end
```

видите ли вы новую запись и если да то почему?

Да, потому что при текущем уровне изоляции в сессии 2 видны все закоммиченные изменения других транзакций, выполненных в любых других сессиях
