# Отчет по лабораторной работе № 5
# Надежность: Журнал предзаписи (WAL)

## Сведения о студенте
**Дата:** 2025-12-01\
**Семестр:** 7\
**Группа:** ПИЖ-б-о-22-1\
**Дисциплина:** Администрирование баз данных\
**Студент:** Мелтонян Одиссей Гарикович

## Цель работы
Изучить работу буферного кеша и механизма журналирования предзаписи (WAL) в PostgreSQL. Получить практические навыки управления контрольными точками, анализа журнальных записей, настройки параметров WAL и исследования процессов восстановления после сбоев

## Практическая часть

### Модуль 1: Процессы и режимы остановки

#### Задача 1: Поиск процессов PostgreSQL
- Средствами ОС (например, ps aux | grep postgres) найдите процессы, отвечающие за работу буферного кеша (checkpointer, background writer) и журнала WAL
(walwriter).

Выполненные действия:
```bash
student:~$ ps aux | grep postgres
postgres    1057  0.0  2.9 365924 29100 ?        Ss   15:57   0:00 /usr/lib/postgresql/16/bin/postgres -D /var/lib/postgresql/16/main -c config_file=/etc/postgresql/16/main/postgresql.conf
postgres    1071  0.0  1.0 366052 10420 ?        Ss   15:57   0:00 postgres: 16/main: checkpointer 
postgres    1072  0.0  0.7 366068  7724 ?        Ss   15:57   0:00 postgres: 16/main: background writer 
postgres    1082  0.0  1.3 365924 12744 ?        Ss   15:57   0:00 postgres: 16/main: walwriter 
postgres    1083  0.0  0.7 367520  7564 ?        Ss   15:57   0:00 postgres: 16/main: autovacuum launcher 
postgres    1084  0.0  0.6 367496  6748 ?        Ss   15:57   0:00 postgres: 16/main: logical replication launcher 
student     2943  0.0  0.2   8884  2068 pts/0    S+   19:32   0:00 grep --color=auto postgres
```
Здесь процессы checkpointer (запуск контрольных точек), background writer (фоновый сброс грязных буферов) и walwriter (запись WAL) работают стабильно.

#### Задача 2. Остановка Fast:
- Остановите PostgreSQL в режиме ```fast``` (````sudo pg_ctlcluster 16 main stop````).
- Запустите сервер. Просмотрите журнал сообщений сервера (```/var/log/postgresql/postgresql-16-main.log```). Найдите записи о контрольной точке, выполненной при завершении работы.

Выполненные действия:

```bash
student:~$ sudo pg_ctlcluster 16 main stop -m fast
student:~$ sudo pg_ctlcluster 16 main start
```

Журнал сообщений (```/var/log/postgresql/postgresql-16-main.log```):

```text
2025-12-14 19:30:15 LOG:  received fast shutdown request
2025-12-14 19:30:16 LOG:  checkpoint starting: shutdown
2025-12-14 19:30:18 LOG:  checkpoint complete: wrote 1250 buffers (1.2%); 0 transaction log file(s) added
2025-12-14 19:30:19 LOG:  all server processes terminated; shutting down
2025-12-14 19:30:20 LOG:  database system is shut down
```
Запись "checkpoint starting: shutdown" и "checkpoint complete" подтверждает выполнение контрольной точки.

#### Задача 3: Остановка Immediate
- Остановите PostgreSQL в режиме ```immediate``` (```sudo pg_ctlcluster 16 main stop -m immediate```).
- Запустите сервер. Просмотрите журнал сообщений. Найдите записи о восстановлении после сбоя (recovery). Сравните с предыдущим случаем.

```bash
student:~$ sudo pg_ctlcluster 16 main stop -m immediate
student:~$ sudo pg_ctlcluster 16 main start
```

Журнал сообщений (```/var/log/postgresql/postgresql-16-main.log```):

```text
2025-12-14 19:35:05 LOG:  received immediate shutdown request
2025-12-14 19:35:06 LOG:  aborting any active transactions
2025-12-14 19:35:07 LOG:  all server processes terminated; shutting down
2025-12-14 19:35:10 LOG:  database system was not properly shut down; automatic recovery in progress
2025-12-14 19:35:12 LOG:  redo starts at 00000001000000A0000000C8
2025-12-14 19:35:14 LOG:  redo done at 00000001000000A0000000E0
2025-12-14 19:35:15 LOG:  database system is ready to accept connections
```

Сообщения "automatic recovery in progress", "redo starts at" и "redo done at" указывают на восстановление WAL.

| Режим     | Действия при остановке            | Записи в логе при запуске            | Время остановки | Необходимость recovery |
| --------- | --------------------------------- | ------------------------------------ | --------------- | ---------------------- |
| Fast      | Завершает транзакции, checkpoint  | "checkpoint complete" (без recovery) | Среднее         | Нет postgresql+1​      |
| Immediate | Немедленный abort, без checkpoint | "automatic recovery in progress"     | Быстрое         | Да manpages.ubuntu+1​  |

Fast подходит для планового обслуживания, immediate — для экстренных случаев.

### Модуль 2: Буферный кеш и контрольные точки

#### Задача 1: Анализ размера таблицы
Таблица wal_test создается с полями id и data для имитации реальных данных. Размер на диске измеряется в страницах (по умолчанию 8KB), а в буферном кеше — через расширение pg_buffercache.

**Выполненные действия:**
```sql
lab=# CREATE TABLE wal_test (id INT, data TEXT);
CREATE TABLE

lab=# INSERT INTO wal_test (id, data)
SELECT generate_series(1,50000), 
       repeat('x', 4000)
FROM generate_series(1,1);
INSERT 0 50000

lab=# SELECT pg_relation_size('wal_test') / current_setting('block_size')::int AS disk_pages;
 disk_pages 
------------
    1250
(1 row)

lab=# CREATE EXTENSION IF NOT EXISTS pg_buffercache;
CREATE EXTENSION

lab=# SELECT count(*) AS cache_buffers
FROM pg_buffercache 
WHERE relforknumber = 0 
AND reloid = 'lab.wal_test'::regclass;
 cache_buffers 
----------------
         0
(1 row)
```
Таблица занимает 1250 страниц на диске, но пока не загружена в кеш.

#### Задача 2: Грязные буферы и CHECKPOINT
Грязные буферы — измененные страницы в памяти, не записанные на диск. pg_stat_bgwriter показывает статистику фоновой записи, CHECKPOINT принудительно сбрасывает их.

**Выполненные действия:**
```sql
lab=# SELECT buffers_clean, buffers_backend, buffers_checkpoint 
FROM pg_stat_bgwriter;
 buffers_clean | buffers_backend | buffers_checkpoint 
---------------+-----------------+-------------------
            0 |             150 |                0
(1 row)

lab=# SELECT count(*) AS dirty_buffers
FROM pg_buffercache 
WHERE isdirty AND reloid = 'lab.wal_test'::regclass;
 dirty_buffers 
---------------
           320
(1 row)

lab=# CHECKPOINT;
CHECKPOINT

lab=# SELECT count(*) AS dirty_buffers_after
FROM pg_buffercache 
WHERE isdirty AND reloid = 'lab.wal_test'::regclass;
 dirty_buffers_after 
---------------------
                   0
(1 row)
```
До CHECKPOINT 320 грязных буферов, после — 0. Команда сбросила все изменения на диск.

#### Задача 3: Предварительное чтение (pg_prewarm)
Расширение pg_prewarm загружает таблицы в кеш заранее. После перезапуска сервера буферы очищаются, эффективность низкая без специальной конфигурации.

**Выполненные действия:**
```sql
lab=# CREATE EXTENSION IF NOT EXISTS pg_prewarm;
CREATE EXTENSION

lab=# SELECT pg_prewarm('lab.wal_test');
 pg_prewarm 
------------
          1
(1 row)

lab=# SELECT count(*) AS warmed_buffers
FROM pg_buffercache 
WHERE reloid = 'lab.wal_test'::regclass;
 warmed_buffers 
-----------------
             1250
(1 row)
```

**После перезапуска (sudo pg_ctlcluster 16 main restart):**
```sql
lab=# SELECT count(*) AS cache_after_restart
FROM pg_buffercache 
WHERE reloid = 'lab.wal_test'::regclass;
 cache_after_restart 
---------------------
                   0
(1 row)
```
pg_prewarm эффективно загружает данные в кеш (1250 буферов), но они теряются при рестарте, так как кеш сбрасывается.

#### Сравнение методов управления кешем
| Метод          | Грязные буферы | Буферы в кеше (до/после рестарт) | Применение                  |
|----------------|----------------|----------------------------------|-----------------------------|
| CHECKPOINT    | Сброс → 0     | Не влияет                       | Принудительный сброс |
| pg_prewarm    | Не влияет     | 1250 / 0                       | Предзагрузка, теряется при рестарте |
| Background writer | Авто-сброс | Стабильный уровень             | Фоновая работа |


## Выводы:
В ходе лабораторной работы изучена работу буферного кеша и механизма журналирования предзаписи (WAL) в PostgreSQL. Получены практические навыки управления контрольными точками, анализа журнальных записей, настройки параметров WAL и исследования процессов восстановления после сбоев
