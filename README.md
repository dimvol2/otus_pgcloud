##### otus_pgcloud
#Курс `PostgreSQL Cloud Solutions`
###ДЗ #1 (Занятие "SQL и реляционные СУБД. PostgreSQL в облаках")

1. Создал и запустил VM в GCE otus-pgcloud2023-micro
2. Сгенерировал пару ssh ключей:
```bash
ssh-keygen -t ed25519
```
публичный ключ добавил в метаданные VM

3. Вошёл ssh по IP адресу VM, установил репозиторий pg и 15 postgres:
```bash
echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list
wget -qO- https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo tee /etc/apt/trusted.gpg.d/pgdg.asc &>/dev/null
apt install postgresql-15
```
Убедился, что кластер запущен:
```bash
root@otus-pgcloud2023-micro:~# pg_lsclusters 
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```

4. Зашёл второй сессией ssh, переключился в пользователя postgres (`sudo su
postgres`),
запустил консоли `psql` в обоих сессиях, выключил в них автокоммит:
```
\set AUTOCOMMIT off
```

5. Cоздал в БД набор тестовых данных из первой консоли:
```sql
postgres=# create table persons(id serial, first_name text, second_name text);
insert into persons(first_name, second_name) values('ivan', 'ivanov');
insert into persons(first_name, second_name) values('petr', 'petrov');
commit;
CREATE TABLE
INSERT 0 1
INSERT 0 1
COMMIT
```

6. Проверил уровень изоляции транзакций:

```sql
postgres=# show transaction isolation level;
commit;
 transaction_isolation 
-----------------------
 read committed
(1 row)

COMMIT
```

7. Начал в обоих консолях новую транзакцию с уровнем изоляции транзакций по
умолчанию (`read committed`):
```sql
postgres=# start transaction;
START TRANSACTION
postgres=*# 
```

8. Добавил в первой консоли запись в таблицу:
```sql
postgres=# insert into persons(first_name, second_name) values('sergey', 'sergeev');
INSERT 0 1
```
Проверил наличие этой записи во второй консоли:
```sql
postgres=*# select * from persons;

 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)
```
её нет, поскольку на уровне изоляции транзакций `read committed` в СУБД
PostgreSQL отсутствует аномалия `dirty read` ("грязное чтение")

Зафиксировал транзакцию в первой консоли:
```sql
postgres=*# commit;
COMMIT
```
и проверил во второй консоли наличие записи:
```sql
postgres=*# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```
Запись есть, поскольку в PostgreSQL на уровне изоляции транзакций `read
committed` присутствует аномалия `nonrepeatable read` ("неповторяющееся
чтение")

Зафиксировал транзакцию во второй консоли
```sql
postgres=*# commit;
COMMIT
```
10) Начал в обеих консолях транзакции с уровнем изоляции `repeatable read`:
```sql
postgres=# start transaction isolation level repeatable read;
START TRANSACTION
```
В первой консоли добавил запись в таблицу:
```sql
postgres=*# insert into persons(first_name, second_name) values('sveta', 'svetova');
INSERT 0 1
```
Во второй консоли проверил наличие этой записи:
```sql
postgres=*# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```
Записи нет, поскольку на уровне изоляции транзакций repeatable read в СУБД
PostgreSQL отсутствует аномалия `dirty read` ("грязное чтение")

Зафиксировал транзакцию в первой консоли:
```sql
postgres=*# commit;
COMMIT
```
и проверил во второй консоли наличие записи:
```sql
postgres=*# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```
Её нет, поскольку на уровне изоляции транзакций `repeatable read` в
PostgreSQL отсутствует аномалия `nonrepeatable read` ("неповторяющееся чтение")

Завершил вторую транзакцию и проверил наличие записи в таблице:
```sql
postgres=*# commit;
COMMIT
postgres=# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
  4 | sveta      | svetova
(4 rows)
```
запись есть

