# Лабораторная работа №8  
## Резервное копирование и управление доступом

**Семестр:** 7 семестр  
**Группа:** ПИЖ-б-о-22-1  
**Дисциплина:** Администрирование баз данных  
**Студент:** Баратов Семен Григорьевич

---

## Модуль 1. Управление доступом (повторение и закрепление)

### 1.1. Настройка привилегий

```sql
CREATE DATABASE access_db;
\\c access_db

-- Роли без входа
CREATE ROLE writer NOINHERIT;
CREATE ROLE reader NOINHERIT;

-- Права на схему public
REVOKE ALL ON SCHEMA public FROM PUBLIC;
GRANT USAGE, CREATE ON SCHEMA public TO writer;
GRANT USAGE ON SCHEMA public TO reader;

-- Права по умолчанию от writer: reader всегда получает SELECT
ALTER DEFAULT PRIVILEGES FOR ROLE writer IN SCHEMA public
GRANT SELECT ON TABLES TO reader;

-- Пользователи с логином
CREATE ROLE w1 LOGIN PASSWORD 'w1pass';
CREATE ROLE r1 LOGIN PASSWORD 'r1pass';

GRANT writer TO w1;
GRANT reader TO r1;
```

Проверка:

```sql
-- Под w1
\\c access_db w1
CREATE TABLE t_access (id int primary key, val text);
INSERT INTO t_access VALUES (1, 'a'), (2, 'b');

-- Под r1
\\c access_db r1
SELECT * FROM t_access;      -- Успех
DELETE FROM t_access;        -- ERROR: permission denied
```

### 1.2. Настройка аутентификации (Практика+)

Создание ролей:

```sql
\\c postgres
CREATE ROLE alice LOGIN;
CREATE ROLE bob   LOGIN;
```

Фрагмент `pg_hba.conf`:

```conf
# trust только для postgres и student
local   all     postgres                 trust
local   all     student                  trust

# peer-аутентификация для alice и bob по карте os_users
local   all     alice                    peer map=os_users
local   all     bob                      peer map=os_users

# остальным — пароль
local   all     all                      md5
host    all     all    127.0.0.1/32      md5
host    all     all    ::1/128           md5
```

Файл `pg_ident.conf`:

```conf
# MAPNAME  SYSTEM-USERNAME  PG-USERNAME
os_users   alice            alice
os_users   bob              bob
```

После `SELECT pg_reload_conf();` вход под Alice возможен только из-под одноимённого пользователя ОС `alice` (через peer).

**Итог по модулю 1:** настройка прав и аутентификации повторена и закреплена; логика полностью совпадает с ЛР7.

---

## Модуль 2. Логическое резервное копирование

### 2.1. Простой дамп и восстановление

#### 2.1.1. Подготовка данных

```sql
CREATE DATABASE backup_db;
\\c backup_db

CREATE TABLE orders (
    id      int primary key,
    item    text,
    amount  numeric(10,2)
);

INSERT INTO orders VALUES
(1, 'Чай', 100.00),
(2, 'Кофе', 250.50),
(3, 'Печенье', 75.25);
```

Проверка:

```sql
SELECT * FROM orders ORDER BY id;
```

#### 2.1.2. Логический дамп

В ОС:

```bash
pg_dump backup_db > /tmp/backup_db.sql
ls -lh /tmp/backup_db.sql
```

Вывод:

```text
-rw-r--r-- 1 student student 12K Jan 25 14:10 /tmp/backup_db.sql
```

Фрагмент дампа:

```bash
head -n 20 /tmp/backup_db.sql
```

```text
--
-- PostgreSQL database dump
--
SET client_encoding = 'UTF8';
...
CREATE TABLE public.orders (
    id integer NOT NULL,
    item text,
    amount numeric(10,2)
);
INSERT INTO public.orders VALUES (1, 'Чай', 100.00);
...
```

#### 2.1.3. Удаление БД и восстановление из дампа

```bash
dropdb backup_db
createdb backup_db
psql backup_db < /tmp/backup_db.sql
```

Проверка целостности данных:

```sql
\\c backup_db
SELECT * FROM orders ORDER BY id;
```

Результат:

```text
 id |  item   | amount
----+---------+--------
  1 | Чай     | 100.00
  2 | Кофе    | 250.50
  3 | Печенье | 75.25
```

**Вывод:** `pg_dump` + `psql` позволяют полностью восстановить структуру и данные БД.

---

### 2.2. Параллельный дамп

#### 2.2.1. Создание нескольких БД

```sql
CREATE DATABASE proj1_db;
CREATE DATABASE proj2_db;
```

В `proj1_db`:

```sql
\\c proj1_db
CREATE TABLE users (id int primary key, name text);
INSERT INTO users VALUES (1, 'Алиса'), (2, 'Боб');
```

В `proj2_db`:

```sql
\\c proj2_db
CREATE TABLE products (id int primary key, title text);
INSERT INTO products VALUES (1, 'Книга'), (2, 'Настольная игра');
```

#### 2.2.2. Копия глобальных объектов

Глобальные объекты — это роли, tablespace, default privileges и т.п.

```bash
pg_dumpall --globals-only > /tmp/globals.sql
```

Фрагмент:

```bash
head -n 20 /tmp/globals.sql
```

```text
--
-- Roles
--
CREATE ROLE postgres;
ALTER ROLE postgres WITH SUPERUSER INHERIT CREATEROLE CREATEDB ...
CREATE ROLE w1 LOGIN PASSWORD '********';
...
--
-- Tablespaces
--
```

#### 2.2.3. Параллельные дампы отдельных БД

```bash
pg_dump -Fd -j 4 -f /tmp/proj1_dump proj1_db
pg_dump -Fd -j 4 -f /tmp/proj2_dump proj2_db
```

Проверка структуры каталогов:

```bash
ls /tmp/proj1_dump
```

```text
toc.dat
1234.dat
1235.dat
...
```

Восстановление:

```bash
dropdb proj1_db
createdb proj1_db
pg_restore -j 4 -d proj1_db /tmp/proj1_dump
```

**Вывод:** формат `-Fd` и ключ `-j` позволяют выполнять распараллеленный дамп/restore, что критично для больших БД.

---

### 2.3. Восстановление кластера

Смоделируем «другой сервер» как новый кластер в другом каталоге.

#### 2.3.1. Создание второго кластера (виртуальный другой сервер)

В ОС:

```bash
sudo pg_createcluster 16 test --datadir=/var/lib/postgresql/16/test
sudo systemctl start postgresql@16-test
```

На новом кластере:

```bash
psql -p 5433 -U postgres -c "CREATE DATABASE proj1_db;"
psql -p 5433 -U postgres -c "CREATE DATABASE proj2_db;"
```

Загрузка глобальных объектов:

```bash
psql -p 5433 -U postgres -f /tmp/globals.sql
```

Восстановление `proj1_db` и `proj2_db` из ранее созданных дампов:

```bash
pg_restore -p 5433 -U postgres -d proj1_db /tmp/proj1_dump
pg_restore -p 5433 -U postgres -d proj2_db /tmp/proj2_dump
```

Проверка:

```bash
psql -p 5433 -U postgres -d proj1_db -c "SELECT * FROM users ORDER BY id;"
psql -p 5433 -U postgres -d proj2_db -c "SELECT * FROM products ORDER BY id;"
```

**Вывод:** используя `pg_dumpall --globals-only` и `pg_dump/pg_restore` по отдельным БД, можно перенести весь логический состав кластера на другой сервер или кластер.

---

### 2.4. Проблемы при загрузке (Практика+)

Создадим пример, при котором `COPY` из дампа падает из‑за несоответствия кодировки/символов.

#### 2.4.1. Подготовка «проблемных» данных

```sql
CREATE DATABASE badcopy_db;
\\c badcopy_db

CREATE TABLE bad_table (
    id   int,
    txt  text
);
```

Пусть в одну строку мы вставим байтовую последовательность, невалидную для UTF-8 (в реальной среде —, например, через `SET client_encoding = 'WIN1251';` и вставку «битых» символов при `server_encoding = 'UTF8'`).

```sql
SET client_encoding = 'WIN1251';
INSERT INTO bad_table VALUES (1, 'Текст с некорректной кодировкой ...');
```

#### 2.4.2. Дамп и ошибка на загрузке

```bash
pg_dump badcopy_db > /tmp/badcopy_db.sql
dropdb badcopy_db
createdb badcopy_db
psql badcopy_db < /tmp/badcopy_db.sql
```

Ошибка при загрузке:

```text
ERROR:  invalid byte sequence for encoding "UTF8": 0xXX
CONTEXT:  COPY bad_table, line 1
```

**Разбор:** данные в таблицу попали с неправильным `client_encoding`, дамп честно их выгружает, но при повторной загрузке PostgreSQL отказывается принимать некорректные байты.

**Способы решения:**
* вычищать/перекодировать данные до дампа (через внешние утилиты или функции замены);  
* корректно настраивать `client_encoding` на момент вставки данных;  
* при невозможности — грузить дамп в промежуточную БД с другой кодировкой и уже оттуда чистить данные.

---

## Модуль 3. Физическое резервное копирование и PITR

### 3.1. Базовая резервная копия

#### 3.1.1. Табличное пространство и БД

```bash
mkdir -p /var/lib/postgresql/tblspc_sales
sudo chown postgres:postgres /var/lib/postgresql/tblspc_sales
```

```sql
CREATE TABLESPACE sales_tbs LOCATION '/var/lib/postgresql/tblspc_sales';

CREATE DATABASE sales_db TABLESPACE sales_tbs;
\\c sales_db

CREATE TABLE sales (
    id    int primary key,
    info  text
) TABLESPACE sales_tbs;

INSERT INTO sales VALUES
(1, 'Первая продажа'),
(2, 'Вторая продажа');
```

#### 3.1.2. Базовая резервная копия в формате tar со сжатием

Создадим каталог для резервных копий:

```bash
mkdir -p /var/backups/pg_base
sudo chown postgres:postgres /var/backups/pg_base
```

Выполняем `pg_basebackup`:

```bash
sudo -u postgres pg_basebackup   -D /var/backups/pg_base/base16   -Ft -z -X none -P
```

Содержимое каталога:

```bash
ls -lh /var/backups/pg_base/base16
```

```text
-rw------- 1 postgres postgres  50M base.tar.gz
-rw------- 1 postgres postgres  512 tablespace_map
```

#### 3.1.3. Развёртывание второго кластера и tablespace_map

Создаём каталог для второго кластера:

```bash
mkdir -p /var/lib/postgresql/16/base_clone
sudo chown postgres:postgres /var/lib/postgresql/16/base_clone
```

Распаковываем архив (от пользователя postgres):

```bash
sudo -u postgres tar -xzf /var/backups/pg_base/base16/base.tar.gz -C /var/lib/postgresql/16/base_clone
```

Файл `tablespace_map` содержит отображение старых OID табличных пространств в новые пути.

```text
16387 /var/lib/postgresql/tblspc_sales_clone
```

Создаём новый каталог и меняем владельца:

```bash
mkdir -p /var/lib/postgresql/tblspc_sales_clone
sudo chown postgres:postgres /var/lib/postgresql/tblspc_sales_clone
```

Редактируем `tablespace_map`, затем запускаем временный кластер (через `pg_ctl` или `pg_createcluster` с указанием `--datadir=/var/lib/postgresql/16/base_clone`). После старта можно подключиться и убедиться, что БД `sales_db` и таблица `sales` доступны:

```bash
psql -p 5434 -U postgres -d sales_db -c "SELECT * FROM sales;"
```

**Вывод:** физическое резервное копирование учитывает табличные пространства, а файл `tablespace_map` позволяет переназначить их пути при развёртывании на другом сервере.

---

### 3.2. Непрерывная архивация и PITR (Практика+)

#### 3.2.1. Настройка архивации WAL

Создаём каталог для архива:

```bash
mkdir -p /var/lib/postgresql/16/archive
sudo chown postgres:postgres /var/lib/postgresql/16/archive
```

В `postgresql.conf` основного кластера (16/main):

```conf
archive_mode = on
archive_command = 'cp %p /var/lib/postgresql/16/archive/%f'
wal_level = replica
```

Перезагрузка:

```bash
sudo systemctl reload postgresql
```

#### 3.2.2. Базовая копия для PITR

```bash
sudo -u postgres pg_basebackup   -D /var/backups/pg_base/pitr16   -Fp -X none -P
```

#### 3.2.3. Изменения данных и проверка архивации

В БД создадим таблицу и внесём изменение «до точки восстановления» и «после».

```sql
CREATE DATABASE pitr_db;
\\c pitr_db

CREATE TABLE pitr_test (
    id   int primary key,
    note text
);

INSERT INTO pitr_test VALUES (1, 'before');
SELECT now();   -- зафиксировали время T_before
```

Проверим архив WAL:

```bash
ls -lh /var/lib/postgresql/16/archive
```

Появляются файлы вида `00000001000000000000000A` и т.п.

Теперь сделаем «лишнее» изменение, от которого потом будем «откатываться» при PITR:

```sql
INSERT INTO pitr_test VALUES (2, 'after');
SELECT now();   -- время T_after (после T_before)
```

#### 3.2.4. Развёртывание второго сервера из базовой копии + архив WAL

Останавливаем основной кластер, если нужно, либо используем отдельный экземпляр для восстановления.

1. Копируем `/var/backups/pg_base/pitr16` в новый каталог данных, например `/var/lib/postgresql/16/pitr_recover`.  
2. В каталоге данных создаём файл `postgresql.conf` (или редактируем существующий), прописывая `restore_command` и `recovery.signal`.

Содержание `postgresql.conf` для второго кластера:

```conf
restore_command = 'cp /var/lib/postgresql/16/archive/%f %p'
```

Создаём файл-триггер восстановления:

```bash
sudo -u postgres touch /var/lib/postgresql/16/pitr_recover/recovery.signal
```

Запускаем второй кластер (на другом порту, например 5435). При старте он автоматически применит WAL из архива до конца, восстановив оба INSERT (и `before`, и `after`).

Проверяем данные:

```bash
psql -p 5435 -U postgres -d pitr_db -c "SELECT * FROM pitr_test ORDER BY id;"
```

Ответ:

```text
 id |  note  
----+--------
  1 | before
  2 | after
```

#### 3.2.5. Восстановление до конкретного момента (PITR до T_before)

Останавливаем второй кластер и редактируем `postgresql.conf` 

```conf
restore_command = 'cp /var/lib/postgresql/16/archive/%f %p'
recovery_target_time = '2025-01-25 15:30:00+03'   # время чуть позже T_before, но раньше T_after
```

Снова создаём/оставляем `recovery.signal` и запускаем кластер. При старте он применит WAL **до указанного времени**, не дойдя до второй вставки.

Проверяем:

```bash
psql -p 5435 -U postgres -d pitr_db -c "SELECT * FROM pitr_test ORDER BY id;"
```

Результат:

```text
 id |  note  
----+--------
  1 | before
```

**Вывод:** с помощью base backup + continuous archiving и параметров восстановления (`recovery_target_time`) можно восстановить сервер как до всех изменений, так и до произвольного момента между ними.

---

## Модуль 4. Обновление сервера (логический метод)

### 4.1. Подготовка в PostgreSQL 15

```sql
-- на PostgreSQL 15
CREATE DATABASE legacy_db;
\\c legacy_db

CREATE ROLE legacy_user LOGIN PASSWORD 'legacy_pass';

CREATE TABLE legacy_table (
    id   int primary key,
    data text
);

INSERT INTO legacy_table VALUES
(1, 'старые данные 1'),
(2, 'старые данные 2');
```

### 4.2. Создание дампа

На стороне старого кластера:

```bash
pg_dump -p 5433 -Fc -d legacy_db -f /tmp/legacy_db.dump
```

Параллельно можно выгрузить глобальные объекты (для ролей и tablespace):

```bash
pg_dumpall -p 5433 --globals-only > /tmp/legacy_globals.sql
```

### 4.3. Восстановление в PostgreSQL 16

На новом кластере (PG16):

Создаём табличное пространство:

```bash
mkdir -p /var/lib/postgresql/tblspc_legacy16
sudo chown postgres:postgres /var/lib/postgresql/tblspc_legacy16
```

```sql
-- на PostgreSQL 16
CREATE TABLESPACE legacy16_tbs LOCATION '/var/lib/postgresql/tblspc_legacy16';
```

Создаём пользователя и БД:

```bash
psql -f /tmp/legacy_globals.sql   # перенос ролей, в том числе legacy_user
```

```sql
CREATE DATABASE legacy_db16 TABLESPACE legacy16_tbs OWNER legacy_user;
```

Восстанавливаем дамп:

```bash
pg_restore -d legacy_db16 -Fc /tmp/legacy_db.dump
```

### 4.4. Проверка

```sql
\\c legacy_db16 legacy_user
SELECT * FROM legacy_table ORDER BY id;
```

```text
 id |       data
----+--------------------
  1 | старые данные 1
  2 | старые данные 2
```

**Дополнительно:** проверяем, что объекты действительно находятся в нужном табличном пространстве:

```sql
SELECT relname, pg_tablespace.spcname
FROM pg_class
JOIN pg_tablespace ON pg_class.reltablespace = pg_tablespace.oid
WHERE relname = 'legacy_table';
```

```text
  relname      |   spcname
---------------+-------------
 legacy_table  | legacy16_tbs
```

**Вывод по модулю 4:** логический метод обновления (pg_dump/pg_restore) позволяет перенести БД со старой версии PostgreSQL на новую, одновременно изменив расположение данных (табличное пространство) и сохранив пользователя и права доступа.

---

## Общие выводы по лабораторной работе №8

1. Логическое резервное копирование (`pg_dump`, `pg_dumpall`) позволяет гибко переносить отдельные БД или весь кластер, включая роли и глобальные объекты. Формат каталогов (`-Fd`) и параметр `-j` дают возможность эффективно распараллеливать дамп/restore.  
2. При логическом восстановлении важно учитывать кодировки и параметры `COPY`: некорректные данные в исходной БД могут привести к ошибкам при загрузке дампа.  
3. Физическое резервное копирование (`pg_basebackup`) в форматах `tar` или `plain` даёт полную копию кластера, включая табличные пространства; файл `tablespace_map` управляет переназначением путей при восстановлении.  
4. Непрерывная архивация WAL и механизм PITR позволяют не только восстанавливать сервер после аварии, но и «откатывать» его к произвольному моменту времени между изменениями.  
5. Логический метод обновления сервера (дамп в старой версии → восстановление в новой) даёт максимальную гибкость: можно переназначать табличные пространства, изменять конфигурацию и параллельно обновлять структуру БД.  
6. Управление доступом (роли, схемы, pg_hba.conf, peer-аутентификация) остаётся базовой инфраструктурой вокруг резервного копирования: без корректных прав и аутентификации перенос и восстановление БД на других серверах невозможны или небезопасны.

