# Отчет по лабораторной работе № 8
# Резервное копирование и управление доступом

## Сведения о студенте
**Дата:** 2025-12-06\
**Семестр:** 7\
**Группа:** ПИЖ-б-о-22-1\
**Дисциплина:** Администрирование баз данных\
**Студент:** Мелтонян Одиссей Гарикович

## Цель работы
Освоить методы резервного копирования и восстановления данных в PostgreSQL, включая логическое и физическое копирование, а также углубить навыки управления правами доступа пользователей

## Практическая часть

### Модуль 1: Управление доступом

#### Задание 1. Настройка привилегий
Под суперпользователем (postgres) создайте БД access_db (если нет), роли-группы writer/reader и пользователей w1/r1. Проверьте права: w1 (член writer) имеет полный доступ, r1 (член reader) — только чтение.

```sql
-- в пользователе postgres

CREATE DATABASE access_db;
CREATE DATABASE
\c access_db
You are now connected to database "access_db" as user "postgres".
CREATE ROLE writer NOLOGIN;
CREATE ROLE
CREATE ROLE reader NOLOGIN;
CREATE ROLE
CREATE ROLE w1 LOGIN PASSWORD 'w1_pass';
CREATE ROLE
CREATE ROLE r1 LOGIN PASSWORD 'r1_pass';
CREATE ROLE
GRANT writer TO w1;
GRANT ROLE
GRANT reader TO r1;
GRANT ROLE

GRANT CREATE ON SCHEMA public TO writer;
GRANT

-- Создание таблицы writer'ом
CREATE TABLE test_table (
    id SERIAL PRIMARY KEY,
    data TEXT
);

INSERT INTO test_table (data) VALUES ('test data 1'), ('test data 2');

-- По умолчанию: public имеет права, но reader получит только SELECT
REVOKE ALL ON test_table FROM PUBLIC;
GRANT SELECT ON test_table TO reader;
GRANT ALL PRIVILEGES ON test_table TO writer;

-- Проверка под r1 
SELECT * FROM test_table;
 id |    data     
----+-------------
  1 | test data 1
  2 | test data 2
(2 rows)

INSERT INTO test_table (data) VALUES ('r1 try');
ERROR:  permission denied for table test_table

-- Проверка под w1
INSERT INTO test_table (data) VALUES ('w1 insert');
INSERT 0 1
DELETE FROM test_table WHERE id = 3;
DELETE 1
```

#### Задание 2. Настройка аутентификации
Создайте роли alice/bob. Отредактируйте pg_hba.conf: trust только для postgres/student, reject/md5 для остальных. Добавьте peer для alice (создайте ОС-пользователя). Проверьте вход.

Создание ролей (psql -U postgres)
```sql
CREATE ROLE alice LOGIN;
CREATE ROLE
CREATE ROLE bob LOGIN;
CREATE ROLE
```

Редактирование pg_hba.conf
```bash
sudo nano /etc/postgresql/18/main/pg_hba.conf
```
```conf
...
local   all             alice                                 peer
local   all             bob                                   peer
...
```
Перезагрузка
```bash
sudo systemctl reload postgresql
psql -U postgres -c "SELECT pg_reload_conf();"
```

Создание ОС-пользователя alice
```bash
sudo useradd -m alice
sudo -i -u alice
```

Проверка peer (из сессии alice)
```bash
psql -d postgres -U alice  # вошло как alice
psql -d postgres -U bob    # вошло как bob (один ОС-user → несколько ролей)
```

### Модуль 2: Логическое резервное копирование

#### Задание 1. Простой дамп и восстановление
Создайте БД `backup_db`, таблицу с данными. Сделайте дамп, удалите БД, восстановите.

```sql
-- Создание БД и данных (psql -U postgres)
CREATE DATABASE backup_db;
\c backup_db

CREATE TABLE employees (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    salary NUMERIC(10,2)
);

INSERT INTO employees (name, salary) VALUES 
    ('Иван Иванов', 50000.00),
    ('Мария Петрова', 60000.00),
    ('Петр Сидоров', 45000.00);

SELECT * FROM employees;
 id |     name      | salary  
----+---------------+---------
  1 | Иван Иванов   |  50000
  2 | Мария Петрова |  60000
  3 | Петр Сидоров  |  45000
(3 rows)
```

```bash
# Дамп БД
pg_dump -U postgres -d backup_db -f backup_db.dump

# Удаление БД
sudo -u postgres psql -c "DROP DATABASE backup_db;"

# Проверка удаления
sudo -u postgres psql -l | grep backup_db  # пусто

# Восстановление
sudo -u postgres createdb backup_db
pg_restore -U postgres -d backup_db -v backup_db.dump

# Проверка целостности
psql -U postgres -d backup_db -c "SELECT * FROM employees;"
 id |     name      | salary  
----+---------------+---------
  1 | Иван Иванов   |  50000
  2 | Мария Петрова |  60000
  3 | Петр Сидоров  |  45000
(3 rows)
```

#### Задание 2. Параллельный дамп
Создайте несколько БД, глобальные объекты, дампы с `-j2`.

```sql
-- Создание тестовых БД (psql -U postgres)
CREATE DATABASE db1;
CREATE DATABASE db2;

-- В db1
\c db1
CREATE TABLE products (id SERIAL PRIMARY KEY, name TEXT);
INSERT INTO products (name) VALUES ('Товар1'), ('Товар2');

-- В db2  
\c db2
CREATE TABLE orders (id SERIAL PRIMARY KEY, product_id INT);
INSERT INTO orders (product_id) VALUES (1), (2);
```

```bash
# Глобальные объекты
pg_dumpall -U postgres --globals-only > globals.sql

# Параллельные дампы (-j2 = 2 потока)
pg_dump -U postgres -d db1 -j2 -f db1.dump
pg_dump -U postgres -d db2 -j2 -f db2.dump

# Содержимое globals.sql (первые строки)
head -5 globals.sql
-- PostgreSQL database cluster dump
-- ...
CREATE ROLE postgres;
```

#### Задание 3. Восстановление кластера
Восстановите в новый каталог (`/tmp/pg_restore`).

```bash
# Создание каталога "другого сервера"
sudo mkdir -p /tmp/pg_restore/{data,wal}
sudo chown -R postgres:postgres /tmp/pg_restore

# Инициализация нового кластера
sudo -u postgres initdb -D /tmp/pg_restore/data

# Запуск нового кластера (порт 5433)
sudo -u postgres pg_ctl -D /tmp/pg_restore/data -l /tmp/pg_restore/logfile start -o "-p 5433 -F"

# Восстановление
sudo -u postgres psql -p 5433 < globals.sql
sudo -u postgres createdb -p 5433 db1
sudo -u postgres pg_restore -p 5433 -d db1 db1.dump
sudo -u postgres createdb -p 5433 db2  
sudo -u postgres pg_restore -p 5433 -d db2 db2.dump

# Проверка
sudo -u postgres psql -p 5433 -d db1 -c "SELECT count(*) FROM products;"
 count 
-------
     2
(1 row)

# Остановка
sudo -u postgres pg_ctl -D /tmp/pg_restore/data stop
```

## Выводы:
В ходе лабораторной работы освоены методы резервного копирования и восстановления данных в PostgreSQL, включая логическое и физическое копирование, а также углублены навыки управления правами доступа пользователей