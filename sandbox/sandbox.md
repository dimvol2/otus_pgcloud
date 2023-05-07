# sandbox


## heading 2

1. раз
2. два
3. установил репозиторий pg и 15 postgres

```bash
echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list
wget -qO- https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo tee /etc/apt/trusted.gpg.d/pgdg.asc &>/dev/null
apt install postgresql-15
```

🚱


```sql
create table persons(id serial, first_name text, second_name text);
```

```sql
postgres=*# commit;
COMMIT
```

```sql
postgres=*# select * from persons;

 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)
```

```sql
postgres=# start transaction;
START TRANSACTION
postgres=*# 
```

