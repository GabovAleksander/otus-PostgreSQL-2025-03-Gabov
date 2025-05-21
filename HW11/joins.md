
##### 1.	Реализовать прямое соединение двух или более таблиц

```sql
select	*
from
	movie m
join genre g on
	m.genre = g.id
```
![image](https://github.com/user-attachments/assets/218bbd29-11a0-496b-9a05-4ecc407aef03)


*Реализована выборка записей где присутствуют записи и в movie и genre, т.е. inner join*

##### 2.	Реализовать левостороннее (или правостороннее) соединение двух или более таблиц

```sql
select	*
from
	genre g
left join movie m  on
	m.genre = g.id
```
![image](https://github.com/user-attachments/assets/51baabdd-35cf-4700-bf6a-97abd9d4efc6)

*Выбраны все жанры из первой таблицы и добавлены если есть фильмы для каждого. Если соответствующей записи нет, то ставится null 
*

##### 3.	Реализовать кросс соединение двух или более таблиц
```sql
select * from movie m cross join genre g
```
![image](https://github.com/user-attachments/assets/7e2de3aa-49c1-4bd7-95b4-7947613dd70a)

*Практического смысла в данном исполнении не имеет, но теперь у нас все фильмы есть в любом жанре. *

##### 4.	Реализовать полное соединение двух или более таблиц
```sql
select * from movie m full join genre g on m.genre = g.id
```
![image](https://github.com/user-attachments/assets/ed9bebc5-23d1-4b8f-a6ff-b1c833269e57)

*Выбирает все записи в левой и в правой таблицах и где есть значения в обеих таблица, там будут заполнены все колонки, а если нет соответствия, то заполнены будут значения только для одной таблицы из двух. В нашем случае для всех записей из таблицы movie есть записи в genre, но не для всех записей таблицы genre есть записи в movie.
*

##### 5.	Реализовать запрос, в котором будут использованы разные типы соединений

```sql
select *
from actor a 
left join actors_in_movie aim on a.id=aim.actor 
right join movie m on aim.movie =m.id 
left join genre g on m.genre = g.id 
order by a.id

```
![image](https://github.com/user-attachments/assets/5509b3f0-bcd6-42ad-bc73-61e0ecac57dc)



*Получили выборку всех фильмов и слева перечень актеров, если есть, а если нет то null, справа проставлены жанры и так же если бы не было жанра то был бы null*


##### 6.	Сделать комментарии на каждый запрос
*Комментарии для запросов сделаны в пунктах 1-5*

##### 7.	К работе приложить структуру таблиц, для которых

![image](https://github.com/user-attachments/assets/46a82448-537d-4258-873e-7efaaf6e062c)

```sql
create table genre (id serial primary key, name text);
create table actor (id serial primary key, lastname text, firstname text);
create table movie (id serial primary key, name text, year int, genre int, foreign key (genre) references genre(id));
create table actors_in_movie (actor int, movie int, foreign key(actor) references actor(id), foreign key(movie) references movie(id));


insert into genre values (1, 'Комедия'), (2, 'Триллер'), (3, 'Ужасы'), (4, 'Фантастика'), (5, 'Сериал'), (6, 'Драма'), (7, 'Документальный');
insert into movie values (1, 'Железный человек', 2008, 4), (2, 'Веном', 2018, 4), (3, 'Оппенгеймер', 2023, 6), (4, 'Острые козырьки', 2013, 5), (5, 'За спичками', 1979, 1);

--козырьки	
insert into actor values (1, 'Киллиан', 'Мерфи'),(	2,'Хелен','Маккрори'),(3,'Пол','Андерсон'), (4,'Том','Харди');

--Железный человек
insert into actor values (5,'Роберт','Дауни-мл.'), (6,'Терренс','Ховард'), (7,'Джефф','Бриджес'),(8,'Гвинет','Пэлтроу');

--веном
insert into actor values (9,'Мишель','Уильямс'), (10,'Риз','Ахмед'),(11,'Скотт','Хейз');

--оппенгеймер
insert into actor values (12,'Эмили','Блант'), (13,'Мэтт','Деймон'), (14,'Флоренс','Пью');

insert into actor values (15,'Чарли','Чаплин'), (16,'Олимпиада','Спартаковна');



insert into actors_in_movie values (5,1);
insert into actors_in_movie values (6,1);
insert into actors_in_movie values (7,1);
insert into actors_in_movie values (8,1);

insert into actors_in_movie values (4,2);
insert into actors_in_movie values (9,2);
insert into actors_in_movie values (10,2);
insert into actors_in_movie values (11,2);

insert into actors_in_movie values (1,3);
insert into actors_in_movie values (12,3);
insert into actors_in_movie values (13,3);
insert into actors_in_movie values (14,3);

insert into actors_in_movie values (1,4);
insert into actors_in_movie values (2,4);
insert into actors_in_movie values (3,4);
insert into actors_in_movie values (4,4);
insert into actors_in_movie values (5,4);
insert into actors_in_movie values (6,4);
```

