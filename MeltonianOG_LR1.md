# Отчет по лабораторной работе № 1
# Архитектура СУБД и конфигурация

## Сведения о студенте
**Дата:** 2025-10-26\
**Семестр:** 7\
**Группа:** ПИЖ-б-о-22-1\
**Дисциплина:** Администрирование баз данных\
**Студент:** Мелтонян Одиссей Гарикович

## Цель работы
Изучить базовые компоненты архитектуры PostgreSQL (процессы, память) и получить практические навыки управления конфигурационными параметрами сервера на разных уровнях (экземпляр, сеанс). Освоить работу с основными и дополнительными файлами конфигурации, а также с представлениями pg_settings и pg_file_settings.

## Практическая часть

### Модуль 1: Исследование параметров и файлов конфигурации

#### Задача 1: Текущая конфигурация
**Цель:** Подключиться к серверу с помощью psql. Определить расположение основного файла конфигурации (postgresql.conf) с помощью команды SHOW config_file;

**Выполненные действия:**
```sql
SHOW config_file;
```

**Результаты:**
```bash
               config_file               
-----------------------------------------
 /etc/postgresql/16/main/postgresql.conf
(1 row)
```

Расположение файла конфигурации: /etc/postgresql/16/main/postgresql.conf

#### Задача 2: Анализ параметров

**Цель:** Изучить представление pg_settings. Найти параметры, для изменения которых требуется перезагрузка сервера (context 'postmaster'). Найти 2-3 параметра с контекстом sighup и user.

**Выполненные действия**
```sql
SELECT name, setting, context
FROM pg_settings
WHERE context = 'postmaster';
```

**Результат** - параметры, для изменения которых требуется перезагрузка сервера:
```bash
                name                 |                 setting                 |  context   
-------------------------------------+-----------------------------------------+------------
 archive_mode                        | off                                     | postmaster
 autovacuum_freeze_max_age           | 200000000                               | postmaster
 autovacuum_max_workers              | 3                                       | postmaster
 autovacuum_multixact_freeze_max_age | 400000000                               | postmaster
 bonjour                             | off                                     | postmaster
 bonjour_name                        |                                         | postmaster
 cluster_name                        | 16/main                                 | postmaster
 config_file                         | /etc/postgresql/16/main/postgresql.conf | postmaster
 data_directory                      | /var/lib/postgresql/16/main             | postmaster
 data_sync_retry                     | off                                     | postmaster
 debug_io_direct                     |                                         | postmaster
 dynamic_shared_memory_type          | posix                                   | postmaster
 event_source                        | PostgreSQL                              | postmaster
 external_pid_file                   | /var/run/postgresql/16-main.pid         | postmaster
 hba_file                            | /etc/postgresql/16/main/pg_hba.conf     | postmaster
 hot_standby                         | on                                      | postmaster
 huge_page_size                      | 0                                       | postmaster
 huge_pages                          | try                                     | postmaster
 ident_file                          | /etc/postgresql/16/main/pg_ident.conf   | postmaster
 ignore_invalid_pages                | off                                     | postmaster
 jit_provider                        | llvmjit                                 | postmaster
 listen_addresses                    | localhost                               | postmaster
 logging_collector                   | off                                     | postmaster
 max_connections                     | 100                                     | postmaster
 max_files_per_process               | 1000                                    | postmaster
 max_locks_per_transaction           | 64                                      | postmaster
 max_logical_replication_workers     | 4                                       | postmaster
 max_pred_locks_per_transaction      | 64                                      | postmaster
 max_prepared_transactions           | 0                                       | postmaster
 max_replication_slots               | 10                                      | postmaster
 max_wal_senders                     | 10                                      | postmaster
 max_worker_processes                | 8                                       | postmaster
 min_dynamic_shared_memory           | 0                                       | postmaster
 old_snapshot_threshold              | -1                                      | postmaster
 port                                | 5432                                    | postmaster
 recovery_target                     |                                         | postmaster
 recovery_target_action              | pause                                   | postmaster
 recovery_target_inclusive           | on                                      | postmaster
 recovery_target_lsn                 |                                         | postmaster
 recovery_target_name                |                                         | postmaster
 recovery_target_time                |                                         | postmaster
 recovery_target_timeline            | latest                                  | postmaster
 recovery_target_xid                 |                                         | postmaster
 reserved_connections                | 0                                       | postmaster
 shared_buffers                      | 16384                                   | postmaster
 shared_memory_type                  | mmap                                    | postmaster
 shared_preload_libraries            |                                         | postmaster
 superuser_reserved_connections      | 3                                       | postmaster
 track_activity_query_size           | 1024                                    | postmaster
 track_commit_timestamp              | off                                     | postmaster
 unix_socket_directories             | /var/run/postgresql                     | postmaster
 unix_socket_group                   |                                         | postmaster
 unix_socket_permissions             | 0777                                    | postmaster
 wal_buffers                         | 512                                     | postmaster
 wal_decode_buffer_size              | 524288                                  | postmaster
 wal_level                           | replica                                 | postmaster
 wal_log_hints                       | off                                     | postmaster
(57 rows)

```

**Выполненные действия**

```sql
SELECT name, setting, context
FROM pg_settings
WHERE context = 'sighup'
LIMIT 3;
```

**Результаты:** - параметры с контекстом sighup

```bash
              name               |  setting   | context 
---------------------------------+------------+---------
 archive_cleanup_command         |            | sighup
 archive_command                 | (disabled) | sighup
 archive_library                 |            | sighup
(3 rows)
```

**Выполненные действия**

```sql
SELECT name, setting, context
FROM pg_settings
WHERE context = 'user'
LIMIT 3;
```

**Результаты:** - параметры с контекстом user

```bash
        name         | setting | context 
---------------------+---------+---------
 application_name    | psql    | user
 array_nulls         | on      | user
 backend_flush_after | 0       | user
(3 rows)
```

#### Задача 3: Анализ файлов
**Цель:** Изучить представление pg_file_settings. Определить, из каких файлов и с какими значениями были считаны текущие настройки параметров shared_buffers и work_mem

```pg_file_settings``` — это системное представление PostgreSQL,
в котором показывается вся информация о параметрах,
считанных из конфигурационных файлов (```postgresql.conf```, ```postgresql.auto.conf```, ```*.conf```)

Структура ```pg_file_settings```:

```sql
\d pg_file_settings
```

```bash
          View "pg_catalog.pg_file_settings"
   Column   |  Type   | Collation | Nullable | Default 
------------+---------+-----------+----------+---------
 sourcefile | text    |           |          | 
 sourceline | integer |           |          | 
 seqno      | integer |           |          | 
 name       | text    |           |          | 
 setting    | text    |           |          | 
 applied    | boolean |           |          | 
 error      | text    |           |          | 
```

| Колонка        | Что значит                                                   |
| -------------- | ------------------------------------------------------------ |
| **sourcefile** | путь к файлу, где найден параметр                            |
| **sourceline** | номер строки в файле                                         |
| **name**       | имя параметра                                                |
| **setting**    | значение параметра                                           |
| **applied**    | применилось ли значение (true/false)                         |
| **error**      | сообщение об ошибке, если была (например, неверное значение) |


**Выполненные действия**

```sql
SELECT sourcefile, sourceline, name, setting
FROM pg_file_settings
WHERE name IN ('shared_buffers', 'work_mem')
```

**Результаты:**

```bash
               sourcefile                | sourceline |      name      | setting 
-----------------------------------------+------------+----------------+---------
 /etc/postgresql/16/main/postgresql.conf |        130 | shared_buffers | 128MB
(1 row)
```

Параметр ```work_mem``` также находится в файле ```/etc/postgresql/16/main/postgresql.conf``` со значением 4MB в закомментированном состоянии

```conf
# - Memory -

...
#work_mem = 4MB
...
```

### Модуль 2: Исследование параметров и файлов конфигурации

#### Задача 1: Изменение через ALTER SYSTEM
**Цель:** Используя команду ALTER SYSTEM, установить для параметра work_mem новое значение. Убедиться, что изменение записалось в файл postgresql.auto.conf (используя функцию pg_read_file). Применить изменение, перечитав конфигурацию (SELECT pg_reload_conf();). Проверить новое значение параметра и его источник pg_settings

**Выполненные действия**
```sql
student=# ALTER SYSTEM SET work_mem = '32MB';
```

**Результат**
```sql
ALTER SYSTEM
```

**Применение изменений**
```sql
SELECT pg_reload_conf();
```

**Проверка изменений**
```sql
student=# SELECT name, setting, source, sourcefile
FROM pg_settings
WHERE name = 'work_mem';
   name   | setting |       source       |                    sourcefile                    
----------+---------+--------------------+--------------------------------------------------
 work_mem | 32768   | configuration file | /var/lib/postgresql/16/main/postgresql.auto.conf
(1 row)
```

#### Задача 2: Изменение через дополнительный файл
**Цель:** Создайте файл в каталоге, указанном в директиве
include_dir основного конфигурационного файла. Установите в этом файле значение для параметра log_min_duration_statement. Примените изменение и проверьте его

**Выполненные действия**

Создание файла
```sql
student:~$ sudo nano /etc/postgresql/16/main/conf.d/logging.conf
```
Применение изменений
```sql
student=# SELECT pg_reload_conf();

 pg_reload_conf 
----------------
 t
(1 row)
```

Проверка результата
```sql
student=# SHOW log_min_duration_statement;

 log_min_duration_statement 
----------------------------
 1s
(1 row)
```

#### Задача 3. Ошибка в конфигурации: 
**Цель:** Намеренно внесите синтаксическую ошибку в один из
конфигурационных файлов (например, invalid_value вместо числового значения). Попытайтесь
перечитать конфигурацию. Изучите представление pg_file_settings, чтобы найти запись об
ошибке. Исправьте ошибку и перечитайте конфигурацию.

**Выполненные действия**
```bash
student:~$ sudo nano /etc/postgresql/16/main/conf.d/shared_buffers.conf
student:~$ cat /etc/postgresql/16/main/conf.d/shared_buffers.conf
shared_buffers = invalid_value
```

```sql
student=# SELECT pg_reload_conf();

 pg_reload_conf 
----------------
 t
(1 row)

student=# SELECT name, error
FROM pg_file_settings
WHERE error IS NOT NULL;

      name      |            error             
----------------+------------------------------
 shared_buffers | setting could not be applied
(1 row)
```

При анализе параметра shared_buffers в представлении pg_file_settings было обнаружено, что значение из дополнительного файла shared_buffers.conf не применилось (поле applied = f, сообщение “setting could not be applied”).
Это связано с тем, что параметр имеет контекст postmaster и требует перезапуска сервера для вступления изменений в силу.

При попытке перезапуска PostgreSQL не может запустить сервер, потому что значение параметра shared_buffers некорректно.

```bash
student:~$ sudo systemctl status postgresql@16-main.service
× postgresql@16-main.service - PostgreSQL Cluster 16-main
     Loaded: loaded (/usr/lib/systemd/system/postgresql@.service; enabled-runtime; preset: enabled)
     Active: failed (Result: protocol) since Sun 2025-11-09 22:14:51 MSK; 6s ago
   Duration: 8min 1.522s
    Process: 2708 ExecStart=/usr/bin/pg_ctlcluster --skip-systemctl-redirect 16-main start (code=exited, status=1/FAILURE)
        CPU: 20ms

ноя 09 22:14:51 course systemd[1]: Starting postgresql@16-main.service - PostgreSQL Cluster 16-main...
ноя 09 22:14:51 course postgresql@16-main[2708]: Error: /usr/lib/postgresql/16/bin/pg_ctl /usr/lib/postgresql/16/bin/pg_ctl>
ноя 09 22:14:51 course postgresql@16-main[2708]: 2025-11-09 22:14:51.786 MSK [2713] LOG:  invalid value for parameter "shar>
ноя 09 22:14:51 course postgresql@16-main[2708]: 2025-11-09 22:14:51.786 MSK [2713] FATAL:  configuration file "/etc/postgr>
ноя 09 22:14:51 course postgresql@16-main[2708]: pg_ctl: could not start server
ноя 09 22:14:51 course postgresql@16-main[2708]: Examine the log output.
ноя 09 22:14:51 course systemd[1]: postgresql@16-main.service: Can't open PID file /run/postgresql/16-main.pid (yet?) after>
ноя 09 22:14:51 course systemd[1]: postgresql@16-main.service: Failed with result 'protocol'.
ноя 09 22:14:51 course systemd[1]: Failed to start postgresql@16-main.service - PostgreSQL Cluster 16-main.
```

Исправление ошибки
```bash
student:~$ sudo nano /etc/postgresql/16/main/conf.d/shared_buffers.conf
student:~$ cat /etc/postgresql/16/main/conf.d/shared_buffers.conf
shared_buffers = 256MB
```

```sql
student=# SHOW shared_buffers;

 shared_buffers 
----------------
 256MB
(1 row)
```

### Модуль 3: Управление параметрами на уровне сеанса

#### Задача 1. Команда SET
**Цель**: В рамках сеанса измените значение параметра work_mem с помощью SET.
Проверьте новое значение. Завершите транзакцию с помощью ROLLBACK и проверьте значение
параметра again. Объясните результат.

**Выполненные действия**

```sql
student=# 
student=# SHOW work_mem;
 work_mem 
----------
 32MB
(1 row)

student=# SET work_mem = '64MB';
SET
student=# SHOW work_mem;
 work_mem 
----------
 64MB
(1 row)

student=# ROLLBACK;
WARNING:  there is no transaction in progress
ROLLBACK
student=# SHOW work_mem;
 work_mem 
----------
 64MB
(1 row)
```

```SET``` действует до конца сеанса.
```ROLLBACK``` не отменяет ```SET```, если он был вне транзакции.

#### Задача 2. Команда SET LOCAL
**Цель:** Откройте транзакцию (BEGIN). Внутри транзакции используйте SET LOCAL для изменения параметра work_mem. Проверьте изменение. После завершения транзакции (COMMIT) проверьте значение параметра ещё раз. Объясните результат.

**Выполненные действия**
```sql
student=# BEGIN;
BEGIN
student=*# SET LOCAL work_mem = '128MB';
SET
student=*# SHOW work_mem;
 work_mem 
----------
 128MB
(1 row)

student=*# COMMIT;
COMMIT
student=# SHOW work_mem;
 work_mem 
----------
 64MB
(1 row)
```

```SET LOCAL``` действует только внутри текущей транзакции. После COMMIT возвращается прежнее значение.

#### Задача 3. Пользовательский параметр
**Цель:** Создайте и установите значение для пользовательского
параметра (имя должно содержать точку, например, app.my_setting). Прочитайте его значение
с помощью current_setting.

**Выполненные действия**
```sql
student=# SET app.my_setting = 'lab01';
SET
student=# SELECT current_setting('app.my_setting');
 current_setting 
-----------------
 lab01
(1 row)
```

## Вывод
В ходе лабораторной работы были изучены базовые компоненты архитектуры PostgreSQL (процессы, память) и получены практические навыки управления конфигурационными параметрами сервера на разных уровнях (экземпляр, сеанс). Освоена работа с основными и дополнительными файлами конфигурации, а также с представлениями pg_settings и pg_file_settings.
