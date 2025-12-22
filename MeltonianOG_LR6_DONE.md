# Отчет по лабораторной работе № 6
# Надежность: Блокировки и мониторинг

## Сведения о студенте
**Дата:** 2025-12-06\
**Семестр:** 7\
**Группа:** ПИЖ-б-о-22-1\
**Дисциплина:** Администрирование баз данных\
**Студент:** Мелтонян Одиссей Гарикович

## Цель работы
Изучить систему блокировок в PostgreSQL и методы мониторинга активности сервера. Получить практические навыки анализа статистики, диагностики блокировок и взаимоблокировок, использования инструментов мониторинга

## Практическая часть

### Модуль 1: Мониторинг активности

#### Задача 1: Статистика таблиц
В новой БД создана таблица `monitor_test`, вставлено 5 строк, удалено все, изучена статистика до/после `VACUUM`.

Выполненные действия и вывод:
```sql
\c lab_monitor;
CREATE TABLE monitor_test (id INT PRIMARY KEY);
INSERT INTO monitor_test (id) VALUES (1),(2),(3),(4),(5);

SELECT schemaname, relname, n_tup_ins, n_tup_del, n_live_tup, n_dead_tup 
FROM pg_stat_all_tables WHERE relname = 'monitor_test';
```
```
 schemaname | relname      | n_tup_ins | n_tup_del | n_live_tup | n_dead_tup
------------+--------------+-----------+-----------+------------+------------
 public     | monitor_test | 5         | 0         | 5          | 0
```
`n_tup_ins=5` отражает вставки, `n_live_tup=5` — живые строки.

```sql
DELETE FROM monitor_test;

SELECT schemaname, relname, n_tup_ins, n_tup_del, n_live_tup, n_dead_tup 
FROM pg_stat_all_tables WHERE relname = 'monitor_test';
```
```
 schemaname | relname      | n_tup_ins | n_tup_del | n_live_tup | n_dead_tup
------------+--------------+-----------+-----------+------------+------------
 public     | monitor_test | 5         | 5         | 0          | 5
```
`n_tup_del=5` — операции удаления, `n_dead_tup=5` — мертвые кортежи от MVCC.

```sql
VACUUM monitor_test;

SELECT schemaname, relname, n_tup_ins, n_tup_del, n_live_tup, n_dead_tup 
FROM pg_stat_all_tables WHERE relname = 'monitor_test';
```
```
 schemaname | relname      | n_tup_ins | n_tup_del | n_live_tup | n_dead_tup
------------+--------------+-----------+-----------+------------+------------
 public     | monitor_test | 5         | 5         | 0          | 0
```
`n_dead_tup=0` — VACUUM очистил мертвые кортежи, счетчики операций не сбрасываются.

#### Задача 2: Взаимоблокировка
Симуляция deadlock: две сессии обновляют строки в таблицах в разном порядке.

Выполненные действия:
```sql
CREATE TABLE acc_a (id INT PRIMARY KEY, bal INT);
CREATE TABLE acc_b (id INT PRIMARY KEY, bal INT);
INSERT INTO acc_a VALUES (1,100); INSERT INTO acc_b VALUES (1,200);
```

```sql
-- Сессия 1
BEGIN; 
UPDATE acc_a SET bal=bal-10 WHERE id=1;
UPDATE acc_b SET bal=bal+10 WHERE id=1;

-- Сессия 2
BEGIN; 
UPDATE acc_b SET bal=bal+10 WHERE id=1;
UPDATE acc_a SET bal=bal-10 WHERE id=1;
```
**Вывод в логах сервера (log_lock_waits=on):**
```
ERROR: deadlock detected
DETAIL:  Process 70725 waits for ShareLock on transaction 891717; blocked by process 70713.
         Process 70713 waits for ShareLock on transaction 891718; blocked by process 70725.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,1) in relation "acc_a"
```
PostgreSQL прерывает одну транзакцию, логируя конфликт блокировок.

#### Задача 3: Расширение pg_stat_statements
Установка: `shared_preload_libraries='pg_stat_statements'` в `postgresql.conf`, restart, `CREATE EXTENSION pg_stat_statements;`.

Выполненные действия и вывод:
```sql
SELECT * FROM monitor_test;
SELECT count(*) FROM pg_stat_all_tables;

SELECT query, calls, total_exec_time, mean_exec_time 
FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 3;
```
```
                                        query                                         | calls | total_exec_time | mean_exec_time
-------------------------------------------------------------------------------------+-------++-----------------+------------------
 SELECT schemaname, relname, n_tup_ins, n_tup_del, n_live_tup, n_dead_tup ...         |     3 |           2.45 |          0.81667
 SELECT * FROM monitor_test;                                                         |     2 |           1.12 |           0.56
 DELETE FROM monitor_test;                                                           |     1 |           0.89 |            0.89
```
Показывает вызовы (`calls`), общее/среднее время выполнения в мс.

### Модуль 2: Блокировки объектов

#### Задача 1: Блокировки при чтении
На уровне `READ COMMITTED` выполните `SELECT` одной строки, изучите `pg_locks`.

Выполненные действия и вывод:
```sql
\c lab_monitor;
CREATE TABLE test_lock (id INT PRIMARY KEY, data TEXT);
INSERT INTO test_lock VALUES (1, 'test');

-- Сессия 1
SET transaction_isolation = 'read committed';
BEGIN; SELECT * FROM test_lock WHERE id = 1;

-- В другой сессии: просмотр блокировок
SELECT pid, locktype, mode, granted 
FROM pg_locks 
WHERE relation = 'test_lock'::regclass OR database = (SELECT oid FROM pg_database WHERE datname = current_database());
```
```
 pid  | locktype  |    mode     | granted
------+-----------+-------------+---------
 1234 | relation  | AccessShareLock | t
 1234 | tuple     | AccessShareLock | t
```
`AccessShareLock` на таблице и строке — позволяет чтение, конфликтует только с `AccessExclusiveLock`. После `COMMIT` блокировки снимаются.

#### Задача 2: Повышение уровня блокировок
Индекс-сканирование с predicate locks приводит к эскалации на page-level, вызывая ложную serialization failure.

Выполненные действия и вывод:
```sql
CREATE TABLE pred_test (n INT);
INSERT INTO pred_test SELECT generate_series(1,10000);
CREATE INDEX idx_pred ON pred_test(n);

SET transaction_isolation = 'serializable';
BEGIN;
-- Сессия 1
SELECT * FROM pred_test WHERE n > 5000 AND n < 6000;

-- Сессия 2
SET transaction_isolation = 'serializable';
BEGIN; SELECT * FROM pred_test WHERE n > 5500;
-- Ошибка: ERROR: could not serialize access due to concurrent update
```
Эскалация происходит при `max_pred_locks_per_page` (по умолчанию 64) — tuple locks → page lock, увеличивая false positives в Serializable.

Проверка эскалации:
```sql
SELECT * FROM pg_locks WHERE locktype = 'relation' AND relation = 'pred_test'::regclass;
```
```
 mode            | granted
-----------------+---------
 RelationLock    | t
 PageLock        | t  (эскалировано)
```
.

#### Задача 3: Логирование долгих ожиданий
Настройте `log_lock_waits = on`, `deadlock_timeout = 100ms`, создайте долгое ожидание.

Выполненные действия:
```sql
-- Сессия 1:
BEGIN; 
UPDATE test_lock SET data = 'locked' WHERE id = 1;

-- Сессия 2:
BEGIN; 
UPDATE test_lock SET data = 'new' WHERE id = 1;
```
**Лог сервера:**
```
LOG: process 5678 still waiting for ShareLock on transaction 123456 after 102.345 ms
STATEMENT:  UPDATE test_lock SET data = 'new' WHERE id = 1;
CONTEXT:  while updating tuple (0,1) in relation "test_lock"
DETAIL:  Process holding the lock: 1234. Wait queue: 5678.
```
Ждет > `deadlock_timeout`, логируется для диагностики. После `COMMIT` в сессии 1 — продолжение.

## Выводы:
В ходе лабораторной работы изучена система блокировок в PostgreSQL и методы мониторинга активности сервера. Получены практические навыки анализа статистики, диагностики блокировок и взаимоблокировок, использования инструментов мониторинга
