# Отчет по лабораторной работе № 3
# Модель многопользовательского доступа: MVCC

## Сведения о студенте
**Дата:** 2025-12-01\
**Семестр:** 7\
**Группа:** ПИЖ-б-о-22-1\
**Дисциплина:** Администрирование баз данных\
**Студент:** Мелтонян Одиссей Гарикович

## Цель работы
Изучить принципы многоверсионного управления конкурентным доступом (MVCC) в
PostgreSQL. Получить практические навыки наблюдения за работой MVCC, анализа версий строк,
снимков данных и уровней изоляции транзакций. Освоить использование расширений и системных
представлений для исследования внутренней структуры данных.

## Практическая часть

### Модуль 1: Уровни изоляции и аномалии

#### Задача 1: Read Committed vs Удаление
- Создайте таблицу iso_test (id INT, data TEXT) и вставьте одну строку.
- В сеансе 1 начните транзакцию с уровнем READ COMMITTED и выполните SELECT * FROM iso_test;.
- В сеансе 2 удалите строку и зафиксируйте изменения (DELETE ...; COMMIT;).
- В сеансе 1 выполните тот же SELECT повторно. Сколько строк увидите? Завершите транзакцию в сеансе 1.

**Выполненные действия:**
```sql
student=# CREATE TABLE iso_test (id INT, data TEXT);
CREATE TABLE
student=# INSERT INTO iso_test VALUES (1, 'row1');
INSERT 0 1

-- Сеанс 1
student=# BEGIN ISOLATION LEVEL READ COMMITTED;
BEGIN
student=# SELECT * FROM iso_test;
 id | data 
----+------
  1 | row1
(1 row)

-- Сеанс 2
student=# DELETE FROM iso_test;
student=# COMMIT;

-- Сеанс 1
student=# SELECT * FROM iso_test;
 id | data 
----+------
(0 rows)

COMMIT;
```

**Результаты:**
в сеансе 1 видно 0 строк, потому что READ COMMITTED берёт новый снимок перед КАЖДОЙ командой и видит уже зафиксированное удаление

#### Задача 2: Repeatable Read vs Удаление
- Повторите предыдущий эксперимент, но в сеансе 1 начните транзакцию с BEGIN ISOLATION LEVEL REPEATABLE READ;.
- Объясните разницу в результатах между двумя уровнями изоляции.

**Выполненные действия:**
```sql
-- Сеанс 1
student=# BEGIN ISOLATION LEVEL REPEATABLE READ;
BEGIN
SELECT * FROM iso_test;
 id | data 
----+------
  1 | row1
(1 row)

-- Сеанс 2
DELETE FROM iso_test;
WARNING:  there is no transaction in progress

-- Сеанс 1
SELECT * FROM iso_test;
 id | data 
----+------
  1 | row1
(1 row)
```

Результат: и в первом, и во втором SELECT в сеансе 1 видно одну и ту же строку, хотя во втором сеансе она уже удалена и COMMIT выполнен.​
Причина: в REPEATABLE READ используется один и тот же снимок данных на всю транзакцию; изменения, закоммиченные после её начала, «невидимы» до её завершения

#### Задача 3: Создание таблицы в транзакции
- В сеансе 1 начните транзакцию и создайте новую таблицу new_table, вставьте в нее строку.
Не фиксируйте.
- В сеансе 2 выполните SELECT * FROM new_table;. Что произойдет?
- Зафиксируйте транзакцию в сеансе 1. Повторите запрос в сеансе 2.
- Повторите процесс, но вместо фиксации откатите транзакцию в сеансе 1. Что изменилось?

**Выполненные действия:**
```sql
-- Сеанс 1
student=# BEGIN;
BEGIN
student=*# CREATE TABLE new_table(id INT);
CREATE TABLE
student=*# INSERT INTO new_table VALUES (1);
INSERT 0 1

-- Сеанс 2
student=# SELECT * FROM new_table;
ERROR:  relation "new_table" does not exist

-- Сеанс 1
COMMIT;
COMMIT

-- Сеанс 2
student=# SELECT * FROM new_table;
 id 
----
  1
(1 row)
```

До COMMIT в сеансе 2 таблица new_table недоступна: запрос завершился ошибкой «relation "new_table" does not exist», потому что команда, выполненная в незавершённой транзакции, не видно другим сеансам.​
После COMMIT таблица появляется, и SELECT в сеансе 2 успешно вернёт строку с данными.​

Вариант с ROLLBACK:

```sql
-- Сеанс 1
student=# BEGIN;
BEGIN
student=*# CREATE TABLE new_table(id INT);
CREATE TABLE
student=*# INSERT INTO new_table VALUES (1);
INSERT 0 1

-- Сеанс 2
student=# SELECT * FROM new_table;
ERROR:  relation "new_table" does not exist

-- Сеанс 1
student=# ROLLBACK;
ROLLBACK

-- Сеанс 2
student=# SELECT * FROM new_table;
ERROR:  relation "new_table" does not exist
```

После ROLLBACK эффекты отменяются, поэтому для сеанса 2 таблица так и не существовала.​

#### Задача 4: Создание таблицы в транзакции
- В сеансе 1 начните транзакцию и выполните SELECT * FROM iso_test; (даже если таблица пуста).
- Попытайтесь в сеансе 2 выполнить DROP TABLE iso_test;. Получится ли? Объясните, почему.

**Выполненные действия:**
```sql
-- Сеанс 1
student=# BEGIN;
BEGIN
student=*# SELECT * FROM iso_test;
 id | data 
----+------
(0 rows)
```

```sql
-- Сеанс 2
student=# DROP TABLE iso_test;

```

Результат: DROP TABLE в сеансе 2 будет ждет и не выполняется, пока транзакция в сеансе 1 не завершится (COMMIT или ROLLBACK).​
Причина: SELECT берёт на таблицу блокировку типа ACCESS SHARE, а DROP TABLE требует ACCESS EXCLUSIVE, которая конфликтует с любыми другими блокировками на таблицу, поэтому DDL блокируется до окончания читающей транзакции.

### Модуль 2: Фантомное чтение и снимки

#### Задача 1. Фантомное чтение (Read Committed)
- Создайте пустую таблицу phantom_test (id INT).
- Продемонстрируйте на уровне Read Committed, что аномалия "фантомное чтение" не предотвращается (вставка новых строк в другом сеансе становится видимой).

**Выполненные действия:**
```sql
-- Сеанс 1
student=# CREATE TABLE phantom_test (id INT);
CREATE TABLE
student=# BEGIN ISOLATION LEVEL READ COMMITTED;
BEGIN
student=*# SELECT * FROM phantom_test;
 id 
----
(0 rows)

-- Сеанс 2
student=# INSERT INTO phantom_test VALUES (1);
INSERT 0 1
student=# COMMIT;
WARNING:  there is no transaction in progress
COMMIT

-- Сеанс 1
student=*# SELECT * FROM phantom_test;
 id 
----
  1
(1 row)
```

Во втором SELECT в сеансе 1 появляется «фантомная» строка, потому что при READ COMMITTED снимок берётся заново для КАЖДОГО запроса и включает уже закоммиченные вставки из других транзакций.​

#### Задача 2. Невидимость удалений (Repeatable Read)
- В сеансе 1 начните транзакцию с уровнем Repeatable Read (пока без запросов).
- В сеансе 2 удалите все строки из phantom_test и зафиксируйте.
- В сеансе 1 выполните SELECT * FROM phantom_test;. Увидятся ли удаленные строки?
- Выполните в сеансе 1 запрос SELECT * FROM pg_database; (не касаясь phantom_test).
- Повлияет ли это на видимость строк в phantom_test при последующем запросе?

**Выполненные действия:**
```sql
-- Сеанс 1
student=# BEGIN ISOLATION LEVEL REPEATABLE READ;
BEGIN
student=*# SELECT * FROM phantom_test;
 id 
----
  1
(1 row)

-- Сеанс 2
student=# DELETE FROM phantom_test;
DELETE 1

-- Сеанс 1
student=*# SELECT * FROM phantom_test;
 id 
----
(0 rows)
```

Снимок для REPEATABLE READ в PostgreSQL создаётся не на BEGIN, а на первом SQL‑запросе в транзакции.​
Поэтому первый SELECT * FROM phantom_test; в сеансе 1 увидит уже пустую таблицу: все строки были удалены и удаление закоммичено до момента взятия снимка.

```sql
-- Сеанс 1
student=*# SELECT * FROM pg_database;
...
student=*# SELECT * FROM phantom_test;
 id 
----
(0 rows)
```

Ни запрос к pg_database, ни что‑либо ещё внутри транзакции не меняют уже взятый снимок: повторный SELECT * FROM phantom_test; снова увидит те же пустые результаты.​
Поведение такое, потому что на уровне REPEATABLE READ один и тот же снимок используется для всех запросов транзакции и не обновляется.

#### Задача 3. Транзакционность DDL
- Убедитесь, что DROP TABLE является транзакционной операцией (можно
откатить).

**Выполненные действия:**
```sql
student=# CREATE TABLE ddl_test (id INT);
CREATE TABLE
student=# INSERT INTO ddl_test VALUES (1);
INSERT 0 1
student=# SELECT * FROM ddl_test;
 id 
----
  1
(1 row)

student=# BEGIN;
BEGIN
student=*# DROP TABLE ddl_test;
DROP TABLE
student=*# SELECT * FROM ddl_test;
ERROR:  relation "ddl_test" does not exist
LINE 1: SELECT * FROM ddl_test;
                      ^
student=!# ROLLBACK;
ROLLBACK
student=# SELECT * FROM ddl_test;
 id 
----
  1
(1 row)
```

После ROLLBACK таблица ddl_test существует и содержит прежние данные, потому что DROP TABLE был частью транзакции и был отменён вместе с ней.​
Если вместо ROLLBACK выполнить COMMIT, таблица будет окончательно удалена, как обычно.​


## Выводы:
В ходе лабораторной работы изучены принципы многоверсионного управления конкурентным доступом (MVCC) в PostgreSQL. Получены практические навыки наблюдения за работой MVCC, анализа версий строк, снимков данных и уровней изоляции транзакций. Освоено использование расширений и системных представлений для исследования внутренней структуры данных.