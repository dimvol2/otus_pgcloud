# sandbox

---
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

