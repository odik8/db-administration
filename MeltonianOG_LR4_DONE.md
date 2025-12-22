# Отчет по лабораторной работе № 4
# Техобслуживание: Очистка (VACUUM)

## Сведения о студенте
**Дата:** 2025-12-01\
**Семестр:** 7\
**Группа:** ПИЖ-б-о-22-1\
**Дисциплина:** Администрирование баз данных\
**Студент:** Мелтонян Одиссей Гарикович

## Цель работы
Всестороннее изучение механизмов очистки (VACUUM) в PostgreSQL. Получение практических навыков управления ручной и автоматической очисткой, анализа работы HOTобновлений, исследования влияния очистки на размер таблиц и индексов, а также работы с заморозкой версий строк

## Практическая часть

### Модуль 1:  Ручная очистка и ее влияние

#### Задача 1: Отключение автоочистки
Глобально отключите процесс автоочистки (autovacuum = off в конфиге, требует перезагрузки, или ALTER SYSTEM + pg_reload_conf()). Убедитесь, что он не работает

**Выполненные действия:**
```sql
postgres=# ALTER SYSTEM SET autovacuum = off;
ALTER SYSTEM
postgres=# SELECT pg_reload_conf();
 pg_reload_conf 
----------------
 t
(1 row)

postgres=# SHOW autovacuum;
 autovacuum 
------------
 off
(1 row)
```

#### Задача 2: Подготовка данных
В новой базе данных создайте таблицу vacuum_test (id INT) и индекс по полю id. Вставьте в таблицу 100 000 случайных чисел.

**Выполненные действия**

```sql
postgres=# CREATE DATABASE lab4;
CREATE DATABASE
postgres=# \c lab4
psql (18.0 (Ubuntu 18.0-1.pgdg24.04+3), server 16.10 (Ubuntu 16.10-1.pgdg24.04+1))
You are now connected to database "lab4" as user "postgres".

lab4=# CREATE TABLE vacuum_test (id INT);
CREATE TABLE

lab4=# CREATE INDEX idx_id ON vacuum_test(id);
CREATE INDEX

lab4=# INSERT INTO vacuum_test (id)
SELECT (random() * 1000000)::int
FROM generate_series(1, 100000);
INSERT 0 100000
```

#### Задача 3. Наблюдение без очистки
- Несколько раз (3-5) обновите половину строк в таблице (UPDATE vacuum_test SET id = id + 1 WHERE random() < 0.5;).
- После каждого обновления контролируйте размер таблицы и индекса с помощью
pg_total_relation_size.
- Зафиксируйте рост размеров

```sql
lab4=# UPDATE vacuum_test SET id = id + 1 WHERE random() < 0.5;
UPDATE 49757
lab4=# SELECT pg_total_relation_size('vacuum_test') AS table_size, pg_total_relation_size('idx_id') AS index_size;
 table_size | index_size 
------------+------------
   10100736 |    4644864
(1 row)

lab4=# UPDATE vacuum_test SET id = id + 1 WHERE random() < 0.5;
UPDATE 50023
lab4=# SELECT pg_total_relation_size('vacuum_test') AS table_size, pg_total_relation_size('idx_id') AS index_size;
 table_size | index_size 
------------+------------
   11517952 |    5152768
(1 row)

lab4=# UPDATE vacuum_test SET id = id + 1 WHERE random() < 0.5;
UPDATE 50185
lab4=# SELECT pg_total_relation_size('vacuum_test') AS table_size, pg_total_relation_size('idx_id') AS index_size;
 table_size | index_size 
------------+------------
   15212544 |    7618560
(1 row)
```

#### Задача 4. Полная очистка
- Выполните VACUUM FULL vacuum_test;.
- Сравните размеры таблицы и индекса до и после.

```sql
lab4=# VACUUM FULL vacuum_test;
VACUUM
lab4=# SELECT pg_total_relation_size('vacuum_test') AS table_size, pg_total_relation_size('idx_id') AS index_size;
 table_size | index_size 
------------+------------
    5865472 |    2236416
(1 row)
```
размеры таблицы и индекса 
до
```sql
 table_size | index_size 
------------+------------
   15212544 |    7618560
(1 row)
```

после
```sql
 table_size | index_size 
------------+------------
    5865472 |    2236416
(1 row)
```

#### Задача 5. Обычная очистка
- Повторите цикл обновлений из пункта 3, но после каждого обновления вызывайте обычную очистку (VACUUM vacuum_test;).
- Сравните динамику размеров с результатами из пункта 3.

```sql
lab4=# UPDATE vacuum_test SET id = id + 1 WHERE random() < 0.5;
UPDATE 50145
lab4=# SELECT pg_total_relation_size('vacuum_test') AS table_size, pg_total_relation_size('idx_id') AS index_size;
 table_size | index_size 
------------+------------
    9945088 |    4464640
(1 row)

lab4=# VACUUM vacuum_test;
VACUUM
lab4=# SELECT pg_total_relation_size('vacuum_test') AS table_size, pg_total_relation_size('idx_id') AS index_size;
 table_size | index_size 
------------+------------
    9945088 |    4464640
(1 row)
```

```sql
lab4=# UPDATE vacuum_test SET id = id + 1 WHERE random() < 0.5;
UPDATE 49954
lab4=# SELECT pg_total_relation_size('vacuum_test') AS table_size, pg_total_relation_size('idx_id') AS index_size;
 table_size | index_size 
------------+------------
    9945088 |    4464640
(1 row)

lab4=# VACUUM vacuum_test;
VACUUM
lab4=# SELECT pg_total_relation_size('vacuum_test') AS table_size, pg_total_relation_size('idx_id') AS index_size;
 table_size | index_size 
------------+------------
   10305536 |    4464640
(1 row)
```

```sql
lab4=# UPDATE vacuum_test SET id = id + 1 WHERE random() < 0.5;
UPDATE 49925
lab4=# SELECT pg_total_relation_size('vacuum_test') AS table_size, pg_total_relation_size('idx_id') AS index_size;
 table_size | index_size 
------------+------------
   16424960 |    8904704
(1 row)

lab4=# VACUUM vacuum_test;
VACUUM
lab4=# SELECT pg_total_relation_size('vacuum_test') AS table_size, pg_total_relation_size('idx_id') AS index_size;
 table_size | index_size 
------------+------------
   16424960 |    8904704
(1 row)
```

Обычный VACUUM не обязан уменьшать файл таблицы или индекса на диске, а лишь помечает «мертвые» кортежи как свободное место для повторного использования. Поэтому pg_total_relation_size до и после VACUUM vacuum_test; вполне может совпадать.

#### Задача 6. Включение автоочистки
- Включите автоочистку обратно

```sql
lab4=# ALTER SYSTEM SET autovacuum = on;
ALTER SYSTEM
lab4=# SELECT pg_reload_conf();
 pg_reload_conf 
----------------
 t
(1 row)

lab4=# SHOW autovacuum;
 autovacuum 
------------
 on
(1 row)

```

### Модуль 2: HOT-обновления и самоочистка

#### Задача 1. Самоочистка без HOT
- Создайте таблицу без индексов. Вставьте данные.
- Выполните несколько обновлений, не удовлетворяющих условиям HOT (например, изменяя поле, по которому потом создадите индекс).
- С помощью pageinspect проанализируйте табличную страницу до и после обновлений и последующей самоочистки. Следите за появлением и исчезновением мертвых кортежей.

Создание таблицы
```sql
CREATE TABLE no_hot (
    id  integer,
    v   text
);
```

Заполнение данных
```sql
INSERT INTO no_hot(id, v)
SELECT g, 'value-'||g
FROM generate_series(1,5) AS g;

SELECT lp, t_xmin, t_xmax, t_ctid FROM heap_page_items(get_raw_page('no_hot', 0));
 lp | t_xmin | t_xmax | t_ctid 
----+--------+--------+--------
  1 |    813 |      0 | (0,1)
  2 |    813 |      0 | (0,2)
  3 |    813 |      0 | (0,3)
  4 |    813 |      0 | (0,4)
  5 |    813 |      0 | (0,5)
(5 rows)
```

До обновлений таблица содержит 5 живых кортежей, все с одинаковым идентификатором вставившей транзакции, без признаков удаления (xmax = 0).

Внесение изменений
```sql
UPDATE no_hot SET id = id + 100 WHERE id = 1;
UPDATE 1

UPDATE no_hot SET id = id + 100 WHERE id = 2;
UPDATE 1

UPDATE no_hot SET id = id + 100 WHERE id = 3;
UPDATE 1

SELECT lp, t_xmin, t_xmax, t_ctid FROM heap_page_items(get_raw_page('no_hot', 0));
 lp | t_xmin | t_xmax | t_ctid 
----+--------+--------+--------
  1 |    813 |    814 | (0,6)
  2 |    813 |    815 | (0,7)
  3 |    813 |    816 | (0,8)
  4 |    813 |      0 | (0,4)
  5 |    813 |      0 | (0,5)
  6 |    814 |      0 | (0,6)
  7 |    815 |      0 | (0,7)
  8 |    816 |      0 | (0,8)
(8 rows)
```

Каждый UPDATE не перезаписывает строку на месте, а создаёт новую версию на свободном месте страницы, помечая старый кортеж как “мертвый” (xmax ≠ 0). В результате число строк выросло с 5 до 8

Самоочистка
```sql
VACUUM no_hot;
VACUUM

SELECT lp, t_xmin, t_xmax, t_ctid FROM heap_page_items(get_raw_page('no_hot', 0));
 lp | t_xmin | t_xmax | t_ctid 
----+--------+--------+--------
  1 |        |        | 
  2 |        |        | 
  3 |        |        | 
  4 |    813 |      0 | (0,4)
  5 |    813 |      0 | (0,5)
  6 |    814 |      0 | (0,6)
  7 |    815 |      0 | (0,7)
  8 |    816 |      0 | (0,8)
(8 rows)
```
Команда VACUUM прошла по таблице, обнаружила, что старые версии c xmax, соответствующим завершённым транзакциям, больше не видимы ни одной активной транзакции, и физически очистила их из страницы.

#### Задача 2. HOT-обновление:
- Создайте таблицу hot_test и индекс по одному из полей.
- Вставьте строку. Выполните обновление, которое удовлетворяет условиям HOT (изменение поля, не входящего в индекс, и наличие свободного места на странице).
- С помощью pageinspect убедитесь, что новая версия строки находится на той же странице, а запись в индексе не изменилась.

Создание таблицы и индекса
```sql
CREATE TABLE hot_test (
    id  integer,
    v   text
);
CREATE TABLE

CREATE INDEX hot_test_id_idx ON hot_test(id);
CREATE INDEX
```

Вставка строки
```sql
INSERT INTO hot_test(id, v) VALUES (1, 'short');
INSERT 0 1

SELECT lp, t_xmin, t_xmax, t_ctid
FROM heap_page_items(get_raw_page('hot_test', 0));
 lp | t_xmin | t_xmax | t_ctid 
----+--------+--------+--------
  1 |    819 |      0 | (0,1)
(1 row)
```
До обновлений на странице 0 находится один живой кортеж без признаков удаления (xmax = 0).

Вставка и обновление
```sql
UPDATE hot_test
SET v = 'a little bit longer text'
WHERE id = 1;
UPDATE 1

SELECT lp, t_xmin, t_xmax, t_ctid
FROM heap_page_items(get_raw_page('hot_test', 0));
 lp | t_xmin | t_xmax | t_ctid 
----+--------+--------+--------
  1 |    819 |    820 | (0,2)
  2 |    820 |      0 | (0,2)
(2 rows)
```
UPDATE изменил неиндексируемое поле v, поэтому сработало HOT-обновление: новая версия создана на той же странице 0

Проверка индекса
```sql
SELECT itemoffset,
       ctid
FROM bt_page_items('hot_test_id_idx', 1);
 itemoffset | ctid  
------------+-------
          1 | (0,1)
(1 row)
```
Индекс по полю id (которое не менялось) содержит одну запись, указывающую на голову HOT-цепочки. Новая версия кортежа не создала дополнительную запись в индексе

#### Задача 3. HOT-обновление с переносом:
- Воспроизведите ситуацию, когда на странице недостаточно места для нового HOT-обновления.
- Выполните обновление. Убедитесь с помощью pageinspect, что новая версия создалась на другой странице.
- Проверьте, сколько записей теперь ссылается на этот кортеж в индексе. 
- Объясните результат.

Создание таблицы и индекса
```sql
CREATE TABLE hot_move (
    id integer,
    v  text
);

CREATE INDEX hot_move_id_idx ON hot_move(id);
```

Вставка начальных данных
```sql
INSERT INTO hot_move(id, v)
VALUES (1, repeat('a', 1000));
```

Проверка содержимого первой страницы:
```sql
SELECT lp, t_xmin, t_xmax, t_ctid
FROM heap_page_items(get_raw_page('hot_move', 0));
 lp | t_xmin | t_xmax | t_ctid 
----+--------+--------+--------
  1 |    830 |      0 | (0,1)
(1 row)
```

Заполнение страницы данными

Для создания ситуации нехватки свободного места страница была заполнена дополнительными строками большого размера
```sql
INSERT INTO hot_move(id, v)
SELECT 2, repeat('b', 2000)
FROM generate_series(1, 100);
```

Обновление строки

Выполним обновление, изменяющее неиндексируемое поле v:
```sql
UPDATE hot_move
SET v = repeat('c', 3000)
WHERE id = 1;
```

Анализ содержимого страниц

Содержимое страницы 0 после обновления:
```sql
SELECT lp, t_xmin, t_xmax, t_ctid
FROM heap_page_items(get_raw_page('hot_move', 0));

 lp | t_xmin | t_xmax | t_ctid 
----+--------+--------+--------
  1 |    830 |    832 | (0,5)
  2 |    831 |      0 | (0,2)
  3 |    831 |      0 | (0,3)
  4 |    831 |      0 | (0,4)
  5 |    832 |      0 | (0,5)
```
Старая версия строки с id = 1 получила значение xmax, указывающее на транзакцию обновления, и стала мёртвой.

Содержимое страницы 1:
```sql
SELECT lp, t_xmin, t_xmax, t_ctid
FROM heap_page_items(get_raw_page('hot_move', 1));

 lp | t_xmin | t_xmax | t_ctid 
----+--------+--------+--------
  1 |    831 |      0 | (1,1)
  2 |    831 |      0 | (1,2)
  3 |    831 |      0 | (1,3)
  4 |    831 |      0 | (1,4)
```

Новая версия обновлённой строки была размещена на другой странице (страница 1), так как на странице 0 не осталось достаточного свободного места.

Проверка индекса
```sql
SELECT itemoffset, ctid
FROM bt_page_items('hot_move_id_idx', 1);
```

В индексе присутствуют две записи, указывающие на разные версии строки.

Несмотря на то, что обновление затрагивало только неиндексируемое поле ```v```, HOT-обновление не произошло, так как на странице, содержащей исходную версию строки, не хватило свободного места для размещения новой версии.

## Выводы:
В ходе лабораторной работы всесторонне изучены механизмы очистки (VACUUM) в PostgreSQL. Получены практические навыки управления ручной и автоматической очисткой, анализа работы HOTобновлений, исследования влияния очистки на размер таблиц и индексов, а также работы с заморозкой версий строк
