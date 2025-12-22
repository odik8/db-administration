# Отчет по лабораторной работе № 7
# Управление доступом, расширениями и локализацией

## Сведения о студенте
**Дата:** 2025-12-06\
**Семестр:** 7\
**Группа:** ПИЖ-б-о-22-1\
**Дисциплина:** Администрирование баз данных\
**Студент:** Мелтонян Одиссей Гарикович

## Цель работы
Освоить управление правами доступа пользователей, работу с расширениями PostgreSQL и настройку параметров локализации. Получить практические навыки настройки аутентификации, управления привилегиями, установки расширений и миграции данных между разными кодировками

## Практическая часть

### Модуль 1: Управление доступом

#### Задача 1: Базовые привилегии
- Создайте новую базу данных access_db и двух пользователей: writer (с правом создания объектов) и reader (только чтение).
- Отзовите у роли public все привилегии на схему public в новой БД. Выдайте роли writer права CREATE и USAGE на схему public, а роли reader — только USAGE.
- Настройте привилегии по умолчанию так, чтобы роль reader автоматически получала право SELECT на новые таблицы в схеме public, принадлежащие writer.
- Создайте пользователей w1 (входит в роль writer) и r1 (входит в роль reader).
- Подключитесь под writer, создайте таблицу test_table. Убедитесь, что r1 может только читать ее, а w1 — имеет полный доступ (включая DELETE).

Создание БД и ролей
```sql
CREATE DATABASE access_db;

CREATE ROLE writer;
CREATE ROLE reader;

CREATE ROLE w1 LOGIN PASSWORD 'w1_pass';
CREATE ROLE r1 LOGIN PASSWORD 'r1_pass';

GRANT writer TO w1;
GRANT reader TO r1;
```

Подключение к новой БД под суперпользователем
```bash
psql -d access_db -U postgres
```

Отбор привилегий у public на схему public и выдача нужных прав ролям:

```sql
REVOKE ALL PRIVILEGES ON SCHEMA public FROM PUBLIC;

GRANT USAGE, CREATE ON SCHEMA public TO writer;

GRANT USAGE ON SCHEMA public TO reader;
```

Привилегии по умолчанию для таблиц writer
```sql
ALTER DEFAULT PRIVILEGES FOR ROLE writer IN SCHEMA public GRANT SELECT ON TABLES TO reader;
```

Проверка через w1
```bash
psql -d access_db -U w1
```
```sql
CREATE TABLE public.test_table (
    id   serial PRIMARY KEY,
    data text
);

INSERT INTO public.test_table (data) VALUES
('row 1'),
('row 2');
```
Роль writer как владелец таблицы имеет полный доступ (SELECT/INSERT/UPDATE/DELETE/TRUNCATE/REFERENCES/TRIGGER).

Проверка, что r1 может только читать
```bash
psql -d access_db -U r1
```

Проверка разрешённых действий:
```sql
SELECT * FROM public.test_table;
ERROR:  permission denied for table test_table
INSERT INTO public.test_table (data) VALUES ('x');
ERROR:  permission denied for table test_table
UPDATE public.test_table SET data = 'y' WHERE id = 1;
ERROR:  permission denied for table test_table
DELETE FROM public.test_table WHERE id = 1;
ERROR:  permission denied for table test_table
```

#### Задача 2: Аутентификация (Практика+)
- Создайте пользовательские роли alice и bob.
- Отредактируйте pg_hba.conf, разрешив беспарольный вход (trust) только для postgres и student. Для всех остальных методов установите reject или md5. Перезагрузите конфигурацию. Убедитесь, что вход для alice и bob запрещен.
- Для alice и bob настройте аутентификацию по peer (сопоставление с пользователем ОС). Убедитесь, что войти нельзя без создания пользователя в ОС.
- Создайте в ОС пользователя alice. Настройте вход в PostgreSQL для роли alice с методом peer. Проверьте вход.
- Проверьте, можно ли использовать одно отображение peer для нескольких ролей.

Создание роли alice и bob

```sql
CREATE ROLE alice LOGIN;
CREATE ROLE bob   LOGIN;
```
Настройка pg_hba.conf: trust только для postgres и student

```bash
sudo nano /etc/postgresql/16/main/pg_hba.conf
```

```conf
# только postgres и student входят без пароля
local   all     postgres,student           trust

# всем остальным либо md5
local   all     all                        md5
```

Перезагрузка конфигурации и проверка запретов для alice/bob

```bash
psql -U postgres -c "SELECT pg_reload_conf();"
```

```bash
psql -U alice -d postgres
psql -U bob   -d postgres
```
Для обоих ролей требуется пароль

Настройка peer для alice и bob
```bash
sudo nano /etc/postgresql/16/main/pg_hba.conf
```

```text
# postgres и student по trust
local   all     postgres,student           trust

# alice и bob по peer
local   all     alice,bob                  peer

# local all all                            reject
```
Перезагружен конфиг:

```bash
psql -U postgres -c "SELECT pg_reload_conf();"
```
Теперь:

```bash
psql -U alice -d postgres
psql -U bob   -d postgres
```
Не пускает, пока нет одноимённых пользователей ОС, потому что peer требует совпадения имени системного пользователя и роли в БД.
​

Создание системного пользователя alice и проверка входа

```bash
sudo useradd -m alice
```

Заходим под ним и пробуем psql без указания имени

```bash
sudo -i -u alice
psql -d postgres
```
Пускает в базу как роль alice без пароля


### Модуль 2: Управление расширениями

#### Задание 1. Установка расширения:
- Установите расширение units_of_measure (или uom, или аналогичное, доступное в вашей сборке). Убедитесь, что оно появилось в pg_available_extensions.

```sql
CREATE EXTENSION pg_trgm;
CREATE EXTENSION
SELECT name, default_version, installed_version
FROM pg_available_extensions
WHERE name = 'pg_trgm';
  name   | default_version | installed_version 
---------+-----------------+-------------------
 pg_trgm | 1.6             | 1.6
(1 row)

SELECT extname, extversion
FROM pg_extension
WHERE extname = 'pg_trgm';
 extname | extversion 
---------+------------
 pg_trgm | 1.6
(1 row)
```

#### Задание 2. Создание и исследование:
- Создайте расширение в своей БД без указания версии. Определите, какая версия установилась и какие скрипты выполнились (можно посмотреть в системном каталоге).

```sql
SELECT objid::regclass AS obj, classid::regclass AS class
FROM pg_depend d
JOIN pg_extension e ON d.refobjid = e.oid
WHERE e.extname = 'pg_trgm' AND d.deptype = 'e';

--- вывод
```

#### Задание 3. Добавление данных:
- Добавьте в справочник расширения новые единицы измерения (например, футы и дюймы).

```sql
CREATE TABLE units_of_measure (
    id   serial PRIMARY KEY,
    code text NOT NULL UNIQUE,
    name text NOT NULL,
    factor numeric
);
CREATE TABLE
INSERT INTO units_of_measure (code, name, factor) VALUES
    ('ft',  'foot',  0.3048),
    ('in',  'inch',  0.0254),
    ('m',   'meter', 1.0),
    ('cm',  'centimeter', 0.01);
INSERT 0 4
SELECT * FROM units_of_measure;
 id | code |    name    | factor 
----+------+------------+--------
  1 | ft   | foot       | 0.3048
  2 | in   | inch       | 0.0254
  3 | m    | meter      |    1.0
  4 | cm   | centimeter |   0.01
(4 rows)
```

#### Задание 4. Управление доступом:
- Измените права доступа к таблицам расширения: отзовите SELECT у public, выдайте его новой специальной роли.

```sql
CREATE ROLE uom_reader;
CREATE ROLE
REVOKE SELECT ON TABLE units_of_measure FROM PUBLIC;
REVOKE
GRANT SELECT ON TABLE units_of_measure TO uom_reader;
GRANT
SET ROLE uom_reader;
SET
SELECT * FROM units_of_measure;
 id | code |    name    | factor 
----+------+------------+--------
  1 | ft   | foot       | 0.3048
  2 | in   | inch       | 0.0254
  3 | m    | meter      |    1.0
  4 | cm   | centimeter |   0.01
(4 rows)

RESET ROLE;
RESET
SELECT * FROM units_of_measure;
ERROR:  permission denied for table units_of_measure
```

#### Задание 5. Резервное копирование:
- Используя pg_dump, выгрузите объекты расширения. Изучите дамп. Убедитесь, что выгружаются метаданные (таблицы, типы, функции) и данные.

```bash
student~$ pg_dump -U postgres -d access_db \
  -t units_of_measure \
  -f units_of_measure_dump.sql
```
units_of_measure_dump.sql:
```
CREATE TABLE public.units_of_measure (...);
COPY public.units_of_measure (id, code, name, factor) FROM stdin;
1	ft	foot	0.3048
2	in	inch	0.0254
...
```


## Выводы:
В ходе лабораторной работы освоено управление правами доступа пользователей, работу с расширениями PostgreSQL и настройку параметров локализации. Получены практические навыки настройки аутентификации, управления привилегиями, установки расширений и миграции данных между разными кодировками