# Отчет по лабораторной работе № 2
# Организация данных и системный каталог

## Сведения о студенте
**Дата:** 2025-11-24\
**Семестр:** 7\
**Группа:** ПИЖ-б-о-22-1\
**Дисциплина:** Администрирование баз данных\
**Студент:** Мелтонян Одиссей Гарикович

## Цель работы
Всестороннее изучение логической и физической структуры хранения данных в
PostgreSQL. Получение практических навыков управления базами данных, схемами, табличными
пространствами. Глубокое освоение работы с системным каталогом для извлечения метаинформации.
Исследование низкоуровневых аспектов хранения, включая TOAST.

## Практическая часть

### Модуль 1: Базы данных и схемы

#### Задача 1: Создание и проверка БД
**Цель:** Создать новую базу данных lab02_db. Проверить ее начальный размер с помощью ```pg_database_size('lab02_db')```

**Выполненные действия:**
```sql
student=# CREATE DATABASE lab02_db;
student=# SELECT pg_database_size('lab02_db');
```

**Результаты:**
```bash
 pg_database_size 
------------------
          7602703
(1 row)
```

Начальный размер - 7602703 байт (≈ 8 МБ — минимальная структура БД PostgreSQL)

Расположение файла конфигурации: /etc/postgresql/16/main/postgresql.conf

#### Задача 2: Работа со схемами

**Цель:** Подключиться к lab02_db. Создать две схемы: app и схему с именем пользователя. В каждой схеме создайте по одной таблице и вставить в них данные

**Выполненные действия:**

Подключение к базе данных
```sql
student=# \c lab02_db;
psql (18.0 (Ubuntu 18.0-1.pgdg24.04+3), server 16.10 (Ubuntu 16.10-1.pgdg24.04+1))
You are now connected to database "lab02_db" as user "student".
```

Создание схем и таблиц
```sql
lab02_db=# CREATE SCHEMA app;
CREATE SCHEMA
lab02_db=# CREATE SCHEMA student;
CREATE SCHEMA
```

Создание и заполнение таблиц
```sql
lab02_db=# CREATE TABLE app.users(id int, name text);
CREATE TABLE
lab02_db=# INSERT INTO app.users VALUES (1, 'Odysseys');
INSERT 0 1
lab02_db=# CREATE TABLE student.items(id int, value text);
CREATE TABLE
lab02_db=# INSERT INTO student.items VALUES (10, 'hello');
INSERT 0 1
```

**Результат**
```sql
lab02_db=# SELECT * FROM app.users;
 id |   name   
----+----------
  1 | Odysseys

lab02_db=# SELECT * FROM student.items;
 id | value 
----+-------
 10 | hello
```

#### Задача 3: Контроль размера

**Цель:** Снова проверить размер базы данных. Объяснить его изменение

**Выполненные действия:**
```sql
lab02_db=# SELECT pg_database_size('lab02_db');
 pg_database_size 
------------------
          7803363
```
Размер базы данных вырос за счёт появления пользовательских таблиц и вставленных данных

#### Задача 4: Управление путем поиска

**Цель:** Настроить параметр search_path для текущего сеанса так, чтобы при обращении по неполному имени приоритет имела пользовательская схема, а затем схема app. Продемонстрировать работу, обратившись к таблицам без указания схемы

**Выполненные действия:**
```sql
lab02_db=# SET search_path TO student, app;
SET
lab02_db=# SELECT * FROM items;
 id | value 
----+-------
 10 | hello
(1 row)

lab02_db=# SELECT * FROM users;
 id |   name   
----+----------
  1 | Odysseys
(1 row)
```
Запросы работают без указания схемы

#### Задача 5: Практика+ (Настройка параметра БД)

**Цель:** Для базы lab02_db установить значение параметра temp_buffers так, чтобы в каждом новом сеансе, подключенном этой БД, оно было в 4 раза больше значения по умолчанию

**Выполненные действия:**
```sql
lab02_db=# ALTER DATABASE lab02_db SET temp_buffers = '32MB';
ALTER DATABASE
lab02_db=# \c lab02_db
psql (18.0 (Ubuntu 18.0-1.pgdg24.04+3), server 16.10 (Ubuntu 16.10-1.pgdg24.04+1))
You are now connected to database "lab02_db" as user "student".
lab02_db=# SHOW temp_buffers;
 temp_buffers 
--------------
 32MB
(1 row)
```

### Модуль 2: Системный каталог

#### Задача 1: Исследование ```pg_class```
**Цель:** Получить описание системной таблицы pg_class.

**Выполненные действия:**

```sql
lab02_db=# \d pg_class

 reltablespace       | oid          |           | not null | 
 relpages            | integer      |           | not null | 
 reltuples           | real         |           | not null | 
 relallvisible       | integer      |           | not null | 
 reltoastrelid       | oid          |           | not null | 
 relhasindex         | boolean      |           | not null | 
 relisshared         | boolean      |           | not null | 
 relpersistence      | "char"       |           | not null | 
 relkind             | "char"       |           | not null | 
 relnatts            | smallint     |           | not null | 
 relchecks           | smallint     |           | not null | 
 relhasrules         | boolean      |           | not null | 
 relhastriggers      | boolean      |           | not null | 
 relhassubclass      | boolean      |           | not null | 
 relrowsecurity      | boolean      |           | not null | 
 relforcerowsecurity | boolean      |           | not null | 
 relispopulated      | boolean      |           | not null | 
 relreplident        | "char"       |           | not null | 
 relispartition      | boolean      |           | not null | 
 relrewrite          | oid          |           | not null | 
 relfrozenxid        | xid          |           | not null | 
 relminmxid          | xid          |           | not null | 
 relacl              | aclitem[]    |           |          | 
 reloptions          | text[]       | C         |          | 
 relpartbound        | pg_node_tree | C         |          | 
Indexes:
    "pg_class_oid_index" PRIMARY KEY, btree (oid)
    "pg_class_relname_nsp_index" UNIQUE CONSTRAINT, btree (relname, relnamespace)
    "pg_class_tblspc_relfilenode_index" btree (reltablespace, relfilenode)

(END)
```

#### Задача 2: Исследование ```pg_tables```
**Цель:** Получите подробное описание представления pg_tables (команда \d+ pg_tables). Объясните разницу между таблицей и представлением

**Выполненные действия:**
```sql
lab02_db=# \d+ pg_tables
                          View "pg_catalog.pg_tables"
   Column    |  Type   | Collation | Nullable | Default | Storage | Description 
-------------+---------+-----------+----------+---------+---------+-------------
 schemaname  | name    |           |          |         | plain   | 
 tablename   | name    |           |          |         | plain   | 
 tableowner  | name    |           |          |         | plain   | 
 tablespace  | name    |           |          |         | plain   | 
 hasindexes  | boolean |           |          |         | plain   | 
 hasrules    | boolean |           |          |         | plain   | 
 hastriggers | boolean |           |          |         | plain   | 
 rowsecurity | boolean |           |          |         | plain   | 

View definition:
SELECT n.nspname AS schemaname,
  c.relname AS tablename,
  pg_get_userbyid(c.relowner) AS tableowner,
  t.spcname AS tablespace,
  c.relhasindex AS hasindexes,
  c.relhasrules AS hasrules,
  c.relhastriggers AS hastriggers,
  c.relrowsecurity AS rowsecurity
  FROM pg_class c
    LEFT JOIN pg_namespace n ON n.oid = c.relnamespace
    LEFT JOIN pg_tablespace t ON t.oid = c.reltablespace
WHERE c.relkind = ANY (ARRAY['r'::"char", 'p'::"char"]);
```

Разница:
Таблица — физически хранит данные
Представление — хранит только SQL-запрос.

#### Задача 3: Исследование ```pg_tables```
**Цель:** В базе lab02_db создать временную таблицу. Получить полный список всех схем в этой БД, включая системные (pg_catalog, information_schema). Объяснить наличие временной схемы

**Выполненные действия:**
```sql
lab02_db=# CREATE TEMP TABLE t_temp(x int);
CREATE TABLE
lab02_db=# SELECT nspname FROM pg_namespace ORDER BY 1;
      nspname       
--------------------
 app
 information_schema
 pg_catalog
 pg_temp_3
 pg_toast
 pg_toast_temp_3
 public
 student
(8 rows)
```
Схема pg_temp_3 — автоматически созданная временная схема.

#### Задача 4: Представления ```information_schema```
**Цель:** Получить список всех представлений в схеме ```information_schema```

**Выполненные действия:**
```sql
              table_name               
---------------------------------------
 pg_shadow
 pg_roles
 pg_hba_file_rules
 pg_settings
 pg_file_settings
 pg_backend_memory_contexts
 pg_ident_file_mappings
 pg_config
 pg_shmem_allocations
 pg_tables
 pg_replication_origin_status
 pg_statio_all_sequences
 pg_group
 pg_user
...
```

#### Задача 5: Анализ метакоманды
**Цель:** Выполнить в psql команду \d+ pg_views. Изучить вывод и объяснить, какие запросы к системному каталогу скрыты за этой командой

**Выполненные действия:**
```sql
lab02_db=# \d+ pg_views
                         View "pg_catalog.pg_views"
   Column   | Type | Collation | Nullable | Default | Storage  | Description 
------------+------+-----------+----------+---------+----------+-------------
 schemaname | name |           |          |         | plain    | 
 viewname   | name |           |          |         | plain    | 
 viewowner  | name |           |          |         | plain    | 
 definition | text |           |          |         | extended | 
View definition:
 SELECT n.nspname AS schemaname,
    c.relname AS viewname,
    pg_get_userbyid(c.relowner) AS viewowner,
    pg_get_viewdef(c.oid) AS definition
   FROM pg_class c
     LEFT JOIN pg_namespace n ON n.oid = c.relnamespace
  WHERE c.relkind = 'v'::"char";
```
Команда \d+ pg_views не просто читает описание объекта — она выводит SQL-код, который лежит в системном представлении pg_catalog.pg_views.

То есть psql фактически показывает, какой запрос PostgreSQL использует, чтобы сформировать список всех представлений в БД.

| Системный каталог           | Для чего используется                               |
| --------------------------- | --------------------------------------------------- |
| **pg_class**                | поиск всех объектов (в том числе представлений)     |
| **pg_namespace**            | получение имён схем                                 |
| **pg_authid** (не напрямую) | получение имени владельца через `pg_get_userbyid()` |
| **pg_get_viewdef()**        | получение SQL-кода каждого представления            |

### Модуль 3: Табличные пространства

#### Задача 1: Создание Tablespace
**Цель:** Создать каталог в файловой системе. Создать новое табличное пространство lab02_ts, указывающее на этот каталог

**Выполненные действия:**
```bash
student:~$ sudo -iu postgres
postgres@course:~$ mkdir mytablespace
postgres@course:~$ ls
16  mytablespace
```

```sql
postgres=# CREATE TABLESPACE lab02_ts LOCATION '/var/lib/postgresql/mytablespace';
CREATE TABLESPACE
```

#### Задача 2: Tablespace по умолчанию
**Цель:** Изменить табличное пространство по умолчанию для базы данных template1 на lab02_ts. Объяснить цель этого действия

**Выполненные действия:**
```sql
postgres=# ALTER DATABASE template1 SET default_tablespace = 'lab02_ts';
ALTER DATABASE
```
Назначено новое tablespace для всех новых БД, созданных на основе template1.

#### Задача 3: Наследование свойства
**Цель:** Создать новую базу данных lab02_db_new. Проверить ее табличное пространство по умолчанию. Объяснить результат

**Выполненные действия:**
```sql
postgres=# CREATE DATABASE lab02_db_new;
CREATE DATABASE
postgres=# SELECT dattablespace FROM pg_database WHERE datname='lab02_db_new';
 dattablespace 
---------------
          1663
(1 row)
```
База наследует tablespace template1.

#### Задача 4: Символическая ссылка
**Цель:** Найти в каталоге PGDATA/pg_tblspc/ символьную ссылку, соответствующую lab02_ts

**Выполненные действия:**
```sql
postgres=# SELECT * 
FROM pg_tablespace
WHERE spcname = 'lab02_ts';
  oid  | spcname  | spcowner | spcacl | spcoptions 
-------+----------+----------+--------+------------
 16415 | lab02_ts |       10 |        | 
(1 row)

postgres=# \q
```

```bash
postgres@course:~$ ls -l /var/lib/postgresql/16/main/pg_tblspc/16415
lrwxrwxrwx 1 postgres postgres 32 ноя 24 15:46 /var/lib/postgresql/16/main/pg_tblspc/16415 -> /var/lib/postgresql/mytablespace
```

#### Задача 5: Удаление Tablespace
**Цель:** Удалить табличное пространство lab02_ts с опцией CASCADE. Объяснить необходимость использования CASCADE

**Выполненные действия:**
```sql
postgres=# DROP TABLESPACE lab02_ts CASCADE;
ERROR:  syntax error at or near "CASCADE"
LINE 1: DROP TABLESPACE lab02_ts CASCADE;
```

В PostgreSQL 16 и младше CASCADE у DROP TABLESPACE не поддерживается

```sql
postgres=# DROP TABLESPACE lab02_ts;
DROP TABLESPACE
```

#### Задача 6: Практика+ (Параметр Tablespace)
**Цель:** Установить параметр ```random_page_cost``` в значение 1.1 для табличного пространства pg_default

**Выполненные действия:**
```sql
postgres=# ALTER TABLESPACE pg_default SET (random_page_cost = 1.1);
ALTER TABLESPACE
postgres=# SELECT spcname, spcoptions
FROM pg_tablespace
WHERE spcname = 'pg_default';
  spcname   |       spcoptions       
------------+------------------------
 pg_default | {random_page_cost=1.1}
(1 row)
```

## Выводы:
в ходе лабораторной работы было проведено всесторонное изучение логической и физической структуры хранения данных в
PostgreSQL. Получены практических навыков управления базами данных, схемами, табличными
пространствами. Глубокое освоение работы с системным каталогом для извлечения метаинформации.
Исследованы низкоуровневые аспекты хранения, включая TOAST.