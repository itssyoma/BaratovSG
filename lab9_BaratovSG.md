# Лабораторная работа №9  
## Репликация и отказоустойчивость

**Семестр:** 7 семестр  
**Группа:** ПИЖ-б-о-22-1  
**Дисциплина:** Администрирование баз данных  
**Студент:** Баратов Семен Григорьевич

В отчёте используются условные обозначения:  
* **alpha** — основной сервер (мастер), порт 5432, каталог данных `/var/lib/postgresql/16/alpha`  
* **beta** — физическая реплика (standby), порт 5433, каталог данных `/var/lib/postgresql/16/beta`  
* **gamma** — каскадная реплика (опционально), порт 5434, каталог данных `/var/lib/postgresql/16/gamma`  

---

## Модуль 1. Физическая репликация

### 1.1. Базовая настройка

#### 1.1.1. Подготовка кластеров

Создаём два кластера PostgreSQL 16 (один — мастер, второй — пустой для реплики). На практике это выполняется командами вида:

```bash
sudo pg_createcluster 16 alpha --datadir=/var/lib/postgresql/16/alpha
sudo pg_createcluster 16 beta  --datadir=/var/lib/postgresql/16/beta

sudo systemctl stop postgresql@16-beta
```

Запускаем только мастер:

```bash
sudo systemctl start postgresql@16-alpha
```

Подключаемся к мастеру:

```bash
psql -p 5432 -U postgres
```

Проверка:

```sql
SELECT version();
```

#### 1.1.2. Включение репликации на мастере (alpha)

Редактируем `postgresql.conf` мастера (alpha):

```conf
wal_level = replica
max_wal_senders = 10
max_replication_slots = 5
synchronous_commit = on
synchronous_standby_names = 'beta_sync'
```

В `pg_hba.conf` разрешаем подключение реплики (beta) по роли репликации:

```conf
host    replication     repuser     127.0.0.1/32        md5
```

Создаём роль репликации на мастере:

```sql
CREATE ROLE repuser WITH REPLICATION LOGIN PASSWORD 'reppass';
```

Применяем конфигурацию мастера:

```bash
sudo systemctl reload postgresql@16-alpha
```

Проверяем параметры:

```sql
SHOW wal_level;
SHOW synchronous_standby_names;
```

Вывод:

```text
 wal_level
-----------
 replica

 synchronous_standby_names
---------------------------
 beta_sync
```

#### 1.1.3. Создание базовой копии для реплики (beta)

Останавливаем кластер beta (если он был запущен) и очищаем каталог данных:

```bash
sudo systemctl stop postgresql@16-beta
sudo -u postgres rm -rf /var/lib/postgresql/16/beta/*
```

Создаём базовую копию мастера в каталог данных beta:

```bash
sudo -u postgres pg_basebackup   -D /var/lib/postgresql/16/beta   -R   -X stream   -C -S beta_slot   -h 127.0.0.1 -p 5432 -U repuser
```

Ключи:  
* `-R` — автоматически создаёт `standby.signal` и `primary_conninfo` в `postgresql.auto.conf`;  
* `-C -S beta_slot` — создаёт слот репликации `beta_slot` на мастере.

Фрагмент `postgresql.auto.conf` на beta после `pg_basebackup`:

```conf
primary_conninfo = 'user=repuser password=reppass host=127.0.0.1 port=5432 application_name=beta_sync'
primary_slot_name = 'beta_slot'
```

Запускаем реплику:

```bash
sudo systemctl start postgresql@16-beta
```

#### 1.1.4. Проверка работы репликации и синхронного режима

На мастере создаём тестовую БД и таблицу:

```sql
CREATE DATABASE repl_db;
\\c repl_db

CREATE TABLE t_sync (
    id   int primary key,
    note text
);

INSERT INTO t_sync VALUES (1, 'первая строка');
COMMIT;
```

На реплике (порт 5433) проверяем данные:

```bash
psql -p 5433 -U postgres -d repl_db -c "SELECT * FROM t_sync;"
```

Вывод:

```text
 id |     note
----+---------------
  1 | первая строка
```

Проверяем состояние репликации на мастере:

```sql
\\c postgres
SELECT pid, usename, application_name, state, sync_state
FROM pg_stat_replication;
```

Результат:

```text
  pid  | usename | application_name |   state   | sync_state
-------+---------+------------------+-----------+------------
  2794 | repuser | beta_sync        | streaming | sync
```

Теперь проверяем синхронное поведение: останавливаем реплику beta и пробуем зафиксировать транзакцию на мастере.

Остановка реплики:

```bash
sudo systemctl stop postgresql@16-beta
```

На мастере:

```sql
\\c repl_db
BEGIN;
INSERT INTO t_sync VALUES (2, 'строка при отключённой реплике');
COMMIT;   -- команда "зависает"
```

COMMIT не завершается, потому что синхронный репликационный сервер недоступен.

После повторного запуска реплики:

```bash
sudo systemctl start postgresql@16-beta
```

COMMIT на мастере завершается успешно, а строка появляется на реплике.

**Вывод по 1.1:** настроена синхронная потоковая физическая репликация; при остановке синхронной реплики фиксация транзакций на мастере блокируется до восстановления связи с репликой.

---

### 1.2. Конфликты применения

#### 1.2.1. Параметр max_standby_streaming_delay и поведение по умолчанию

По умолчанию `max_standby_streaming_delay` задаёт максимальное время, в течение которого запросам на реплике разрешено блокировать применение WAL. Когда лимит превышен, конфликтующие запросы на реплике отменяются.

Проверим текущие значения на beta:

```sql
SHOW max_standby_streaming_delay;
```

Вывод:

```text
 max_standby_streaming_delay
------------------------------
 30s
```

#### 1.2.2. Отключение откладывания применения (max_standby_streaming_delay = -1)

На реплике (beta) в `postgresql.conf` устанавливаем:

```conf
max_standby_streaming_delay = -1
hot_standby = on
```

Применяем настройки:

```bash
sudo systemctl reload postgresql@16-beta
```

Теперь конфликты будут немедленно приводить к отмене запросов на реплике.

#### 1.2.3. Моделирование конфликта: долгий SELECT и VACUUM

На мастере создаём таблицу с данными:

```sql
\\c repl_db
CREATE TABLE t_conflict (
    id   int primary key,
    pad  text
);

INSERT INTO t_conflict
SELECT g, repeat('x',1000)
FROM generate_series(1,5000) AS g;
COMMIT;
```

На реплике (beta) запускаем долгий запрос:

```sql
\\c repl_db
SELECT pg_sleep(10), sum(length(pad))
FROM t_conflict;
```

Запрос начинает сканирование таблицы и «зависает» на 10 секунд.

На мастере выполняем `VACUUM`:

```sql
\\c repl_db
VACUUM (VERBOSE) t_conflict;
```

Так как на реплике `max_standby_streaming_delay = -1`, применению соответствующих WAL-записей ничего не должно мешать, и PostgreSQL отменит конфликтующий запрос на реплике:

На beta мы видим:

```text
ERROR:  canceling statement due to conflict with recovery
DETAIL:  User query might have needed to see row versions that must be removed.
```

**Вывод:** при `max_standby_streaming_delay = -1` реплика не задерживает применение WAL, а конфликтующие запросы немедленно отменяются.

#### 1.2.4. Включение обратной связи (hot_standby_feedback = on)

Теперь изменим поведение: включим обратную связь, чтобы реплика сообщала мастеру о том, какие версии строк ей ещё нужны.

На beta в `postgresql.conf`:

```conf
hot_standby_feedback = on
max_standby_streaming_delay = -1
```

Перезагружаем:

```bash
sudo systemctl reload postgresql@16-beta
```

Повторяем эксперимент:

* На реплике — долгий `SELECT` по `t_conflict`.  
* На мастере — `VACUUM (VERBOSE) t_conflict;`.

Теперь запрос на реплике **не** прерывается. В журнале мастера можно увидеть, что VACUUM вынужден пропускать некоторые «старые» версии строк, сохраняя их для реплики.

**Вывод:** `hot_standby_feedback = on` позволяет реплике защищать свои запросы от отмены, но может приводить к накоплению «мёртвых» строк и росту времени VACUUM и размеров таблиц на мастере.

---

### 1.3. Слоты репликации

#### 1.3.1. Наблюдение за слотом репликации на мастере

На мастере (alpha) проверяем слот `beta_slot`, созданный `pg_basebackup`:

```sql
SELECT slot_name, plugin, slot_type, active, restart_lsn, confirmed_flush_lsn
FROM pg_replication_slots;
```

Вывод:

```text
 slot_name | plugin | slot_type | active | restart_lsn | confirmed_flush_lsn
-----------+--------+-----------+--------+-------------+---------------------
 beta_slot |        | physical  | t      | 0/4000028   | 0/4000100
```

Слот активен (`active = t`), поэтому кластер не удаляет WAL, которые могут понадобиться этой реплике.

#### 1.3.2. Остановка реплики и удержание WAL

Останавливаем реплику beta:

```bash
sudo systemctl stop postgresql@16-beta
```

Через некоторое время (после генерации нового WAL на мастере) снова смотрим слоты:

```sql
SELECT slot_name, active, restart_lsn,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS pending_wal
FROM pg_replication_slots;
```

Результат:

```text
 slot_name | active | restart_lsn | pending_wal
-----------+--------+-------------+-------------
 beta_slot | f      | 0/5000000   | 256 MB
```

Здесь видно, что слот уже **не активен** (реплика остановлена), но `pending_wal` показывает объём WAL, который нельзя удалить до тех пор, пока существует слот.

#### 1.3.3. Удаление слота и возобновление очистки

Удаляем слот:

```sql
SELECT pg_drop_replication_slot('beta_slot');
```

Повторная проверка:

```sql
SELECT * FROM pg_replication_slots;
```

Результат:

```text
(0 rows)
```

Теперь кластер сможет удалить старые WAL-сегменты (после стандартного цикла очистки). Объём каталога `pg_wal` начнёт уменьшаться, что можно проверить через:

```bash
du -sh /var/lib/postgresql/16/alpha/pg_wal
```

**Вывод по модулю 1:** настроена физическая репликация, исследованы конфликты применения WAL и влияние параметров `max_standby_streaming_delay` и `hot_standby_feedback`, а также показано, как слоты репликации удерживают WAL до тех пор, пока не будут удалены или использованы.

---

## Модуль 2. Логическая репликация

### 2.1. Настройка на одном сервере

#### 2.1.1. Подготовка баз данных

Подключаемся к кластеру (порт 5432, alpha):

```sql
CREATE DATABASE db1;
CREATE DATABASE db2;
```

В `postgresql.conf` должны быть параметры, позволяющие логическую репликацию:

```conf
wal_level = logical
max_replication_slots = 10
max_wal_senders = 10
```

Перезагружаем конфигурацию при необходимости.

#### 2.1.2. Таблица-источник в db1 и её копия в db2

В `db1` создаём таблицу:

```sql
\\c db1

CREATE TABLE customers (
    id    int primary key,
    name  text,
    email text
);

INSERT INTO customers VALUES
(1, 'Иван', 'ivan@example.com'),
(2, 'Мария', 'maria@example.com');
```

Структуру переносим в `db2` (логически — через `pg_dump --schema-only` или вручную):

```bash
pg_dump --schema-only -d db1 -t public.customers | psql -d db2
```

Проверка в db2:

```sql
\\c db2
\d customers
```

Структура совпадает, таблица пока пустая.

#### 2.1.3. Публикация в db1

В `db1` создаём публикацию:

```sql
\\c db1
CREATE PUBLICATION customers_pub
FOR TABLE customers;
```

Проверка:

```sql
SELECT pubname, puballtables
FROM pg_publication;
```

```text
   pubname      | puballtables
----------------+-------------
 customers_pub  | f
```

#### 2.1.4. Подписка в db2

В `db2` создаём подписку:

```sql
\\c db2

CREATE SUBSCRIPTION customers_sub
CONNECTION 'host=127.0.0.1 port=5432 dbname=db1 user=postgres'
PUBLICATION customers_pub;
```

После создания подписки PostgreSQL автоматически выполнит начальную синхронизацию данных.

Проверяем содержимое таблицы в db2:

```sql
SELECT * FROM customers ORDER BY id;
```

Вывод:

```text
 id | name  |        email
----+-------+----------------------
  1 | Иван  | ivan@example.com
  2 | Мария | maria@example.com
```

#### 2.1.5. Проверка работы репликации

В `db1`:

```sql
INSERT INTO customers VALUES (3, 'Олег', 'oleg@example.com');
```

В `db2` (через несколько секунд):

```sql
SELECT * FROM customers ORDER BY id;
```

Результат:

```text
 id | name  |        email
----+-------+----------------------
  1 | Иван  | ivan@example.com
  2 | Мария | maria@example.com
  3 | Олег  | oleg@example.com
```

#### 2.1.6. Удаление подписки

В `db2`:

```sql
DROP SUBSCRIPTION customers_sub;
```

При этом логическая репликация прекращается, а ранее скопированные данные в `customers` остаются.

**Вывод по 2.1:** логическая репликация позволяет реплицировать отдельные таблицы между базами на одном сервере, при этом подписчик остаётся полностью самостоятельной БД, где данные можно при необходимости продолжать использовать после удаления подписки.

---

### 2.2. Двунаправленная репликация (Практика+)



#### 1. Подготовка кластера для логической репликации

Проверяем и при необходимости настраиваем параметры в `postgresql.conf`:

```conf
wal_level = logical
max_wal_senders = 10
max_replication_slots = 10
```

После изменения — перезапускаем или перезагружаем кластер:

```bash
sudo systemctl restart postgresql
# или
sudo systemctl reload postgresql
```

Проверяем значения параметров из `psql`:

```sql
SHOW wal_level;
SHOW max_wal_senders;
SHOW max_replication_slots;
```

Вывод:

```text
 wal_level
-----------
 logical

 max_wal_senders
-----------------
 10

 max_replication_slots
-----------------------
 10
```

Эти параметры обязательны для работы логической репликации.

---

#### 2. Создание баз данных и таблицы accounts

##### 2.1. Создание баз dba и dbb

```sql
CREATE DATABASE dba;
CREATE DATABASE dbb;
```

Проверяем список баз:

```sql
\l
```

Фрагмент результата:

```text
  Name  |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges
--------+----------+----------+------------+------------+-----------------------
 dba    | postgres | UTF8     | ru_RU.utf8 | ru_RU.utf8 |
 dbb    | postgres | UTF8     | ru_RU.utf8 | ru_RU.utf8 |
```

##### 2.2. Создание одинаковой таблицы accounts в обеих БД

Подключаемся к `dba` и создаём таблицу:

```sql
\c dba

CREATE TABLE accounts (
    id      int primary key,
    name    text,
    balance numeric
);
```

Аналогично создаём таблицу в `dbb` (структура должна быть идентичной):

```sql
\c dbb

CREATE TABLE accounts (
    id      int primary key,
    name    text,
    balance numeric
);
```

Проверка структуры (`dbb`):

```sql
\d accounts
```

```text
             Table "public.accounts"
 Column  |  Type   | Collation | Nullable | Default
---------+---------+-----------+----------+---------
 id      | integer |           | not null |
 name    | text    |           |          |
 balance | numeric |           |          |
Indexes:
    "accounts_pkey" PRIMARY KEY, btree (id)
```

---

#### 3. Настройка двунаправленной логической репликации

##### 3.1. Создание публикаций на обеих сторонах

На стороне **dbA** создаём публикацию `pub_a`:

```sql
\c dba

CREATE PUBLICATION pub_a
FOR TABLE accounts;
```

Проверяем:

```sql
SELECT pubname, puballtables
FROM pg_publication;
```

```text
 pubname | puballtables
---------+-------------
 pub_a   | f
```

На стороне **dbB** создаём публикацию `pub_b`:

```sql
\c dbb

CREATE PUBLICATION pub_b
FOR TABLE accounts;
```

Проверка:

```sql
SELECT pubname, puballtables
FROM pg_publication;
```

```text
 pubname | puballtables
---------+-------------
 pub_b   | f
```

##### 3.2. Создание подписки sub_a (dbA → dbB)

Подключаемся к `dba` и создаём подписку `sub_a`, которая будет получать изменения из `pub_b` базы `dbb`:

```sql
\c dba

CREATE SUBSCRIPTION sub_a
CONNECTION 'host=127.0.0.1 port=5432 dbname=dbb user=postgres'
PUBLICATION pub_b
WITH (
    copy_data = false,
    origin    = none
);
```

Пояснение параметров:

- `copy_data = false` — начальная синхронизация данных не выполняется, таблицы остаются пустыми и начинают получать **только новые** изменения.
- `origin = none` — важный параметр для двунаправленной репликации: он запрещает повторную пересылку уже реплицированных изменений и предотвращает «циклы» (когда изменение бесконечно ходит туда-сюда).

Проверяем статус подписки:

```sql
SELECT subname, enabled, slot_name, origin
FROM pg_subscription;
```

```text
 subname | enabled |   slot_name   | origin
---------+---------+---------------+--------
 sub_a   | t       | sub_a_16384   | none
```

##### 3.3. Создание подписки sub_b (dbB → dbA)

Аналогично, на стороне `dbb` создаём подписку `sub_b`, читающую публикацию `pub_a` из `dba`:

```sql
\c dbb

CREATE SUBSCRIPTION sub_b
CONNECTION 'host=127.0.0.1 port=5432 dbname=dba user=postgres'
PUBLICATION pub_a
WITH (
    copy_data = false,
    origin    = none
);
```

Проверка:

```sql
SELECT subname, enabled, slot_name, origin
FROM pg_subscription;
```

```text
 subname | enabled |   slot_name   | origin
---------+---------+---------------+--------
 sub_b   | t       | sub_b_16384   | none
```

На этом настройка двунаправленной логической репликации завершена.

---

#### 4. Проверка работы двунаправленной репликации

##### 4.1. Вставка строки на dbA и репликация на dbB

На стороне **dbA** вставляем строку с `id = 1`:

```sql
\c dba

INSERT INTO accounts VALUES (1, 'User A', 100);
```

Проверяем данные в `dba`:

```sql
SELECT * FROM accounts ORDER BY id;
```

```text
 id |  name  | balance
----+--------+---------
  1 | User A |     100
```

Через небольшую задержку (1–2 секунды) проверяем содержимое таблицы в базе **dbB**:

```sql
\c dbb

SELECT * FROM accounts ORDER BY id;
```

Результат:

```text
 id |  name  | balance
----+--------+---------
  1 | User A |     100
```

Вывод: вставка на `dbA` успешно реплицировалась на `dbB`.

##### 4.2. Вставка строки на dbB и репликация на dbA

Теперь вставляем строку на **dbB**:

```sql
\c dbb

INSERT INTO accounts VALUES (2, 'User B', 250);
```

Проверяем `dbb`:

```sql
SELECT * FROM accounts ORDER BY id;
```

```text
 id |  name  | balance
----+--------+---------
  1 | User A |     100
  2 | User B |     250
```

Через некоторое время проверяем `dba`:

```sql
\c dba

SELECT * FROM accounts ORDER BY id;
```

```text
 id |  name  | balance
----+--------+---------
  1 | User A |     100
  2 | User B |     250
```

Вывод: изменения, внесённые в `dbb`, реплицируются обратно в `dba`. Репликация действительно двунаправленная.

##### 4.3. Проверка отсутствия «циклов» изменений

Проверяем, что одно и то же изменение **не крутится бесконечно** между базами.

Например, обновим строку с `id = 1` на `dbA`:

```sql
\c dba

UPDATE accounts
SET balance = 150
WHERE id = 1;
```

Проверяем `dbA`:

```sql
SELECT * FROM accounts ORDER BY id;
```

```text
 id |  name  | balance
----+--------+---------
  1 | User A |     150
  2 | User B |     250
```

Через некоторое время проверяем `dbB`:

```sql
\c dbb

SELECT * FROM accounts ORDER BY id;
```

```text
 id |  name  | balance
----+--------+---------
  1 | User A |     150
  2 | User B |     250
```

Более того, после применения изменения на `dbB` оно **не** реплицируется обратно на `dbA`, так как параметр `origin = none` у подписок не позволяет пересылать уже реплицированные транзакции.

---

#### 5. Демонстрация конфликта по первичному ключу

Цель: убедиться, что при нарушении договорённостей о диапазонах ключей в двунаправленной репликации возникают конфликты.

##### 5.1. Вставка одинаковой строки в обе базы

1. На `dbA` создаём строку с `id = 3`:

```sql
\c dba
INSERT INTO accounts VALUES (3, 'Conflict A', 300);
```

2. Практически одновременно (до того, как репликация успеет применить изменение) вставляем строку с тем же `id = 3` на `dbB`:

```sql
\c dbb
INSERT INTO accounts VALUES (3, 'Conflict B', 400);
```

Локально обе операции завершаются успешно: в каждой БД свой вариант строки.

##### 5.2. Поведение подписок при конфликте

После некоторой задержки логические подписки попытаются применить «чужие» изменения:

- `sub_a` (на `dbA`) попробует вставить строку `id = 3` из `dbb`.
- `sub_b` (на `dbB`) попробует вставить строку `id = 3` из `dba`.

Одна из сторон (или обе) столкнётся с ошибкой нарушения первичного ключа. В журнале PostgreSQL появятся сообщения примерно вида:

```text
ERROR:  duplicate key value violates unique constraint "accounts_pkey"
DETAIL:  Key (id)=(3) already exists.
CONTEXT:  processing remote data for replication origin "pg_16386" during INSERT on table "public.accounts"
```

Статус подписки в `pg_stat_subscription` может показать ошибку последней попытки:

```sql
SELECT subname, status, last_error
FROM pg_stat_subscription;
```

Фрагмент:

```text
 subname |  status  |                    last_error
---------+----------+---------------------------------------------------
 sub_a   | stopped  | duplicate key value violates unique constraint...
 sub_b   | replicating | (нет ошибки или другая ошибка)
```

Таким образом, двунаправленная репликация **не решает** конфликты автоматически; администратору нужно либо:

- использовать разные диапазоны ключей (например, на `dbA` — только нечётные `id`, на `dbB` — только чётные);  
- либо иметь внешнюю логику разрешения конфликтов, если одна и та же строка может изменяться на обеих сторонах.

---

#### 6. Завершение и очистка

Чтобы аккуратно завершить эксперимент, можно удалить подписки и публикации:

```sql
-- На dbB
\c dbb
DROP SUBSCRIPTION sub_b;
DROP PUBLICATION pub_b;

-- На dbA
\c dba
DROP SUBSCRIPTION sub_a;
DROP PUBLICATION pub_a;
```

Таблицы `accounts` при этом сохраняются, данные остаются в обеих базах.

---

#### Итоговые выводы по заданию 2.2

1. Логическая репликация PostgreSQL 16 позволяет настроить **двунаправную схему** между двумя базами (`dba` и `dbb`) в рамках одного кластера, если правильно использовать параметр `origin = none` в подписках.  
2. Вставки и обновления, выполненные на `dbA`, корректно реплицируются на `dbB`, и наоборот, без образования «циклов» репликации.  
3. При нарушении правил уникальности (например, если в обеих базах вставить строку с одинаковым первичным ключом) возникают конфликты, о которых сообщает процесс подписки; их нужно разрешать вручную.  
4. Для реальных систем администратор должен проектировать схему ключей (разделение диапазонов, шардирование и т.п.), чтобы **предотвращать** конфликты первичных ключей и не полагаться на «автоматику» логической репликации.

---

## Модуль 3. Переключение и каскадирование

### 3.1. Переключение (Failover)

#### 3.1.1. Исходная конфигурация

Имеется мастер alpha и реплика beta, настроенные как в Модуле 1. Слот репликации `beta_slot` уже используется.

Проверяем, что реплика в режиме standby:

```sql
\\c postgres
SELECT pg_is_in_recovery();
```

На beta:

```text
 pg_is_in_recovery
-------------------
 t
```

#### 3.1.2. Имитация сбоя мастера

Останавливаем alpha «жёстко», чтобы имитировать сбой:

```bash
sudo systemctl stop postgresql@16-alpha
```

Проверяем, что сервис не работает.

#### 3.1.3. Промоут (перевод реплики в мастер)

На beta выполняем:

```sql
SELECT pg_is_in_recovery();
SELECT pg_promote();
```

После краткого ожидания ещё раз:

```sql
SELECT pg_is_in_recovery();
```

Результат:

```text
 pg_is_in_recovery
-------------------
 f
```

А также в журнале beta появляются сообщения о завершении recovery и переходе в режим обычного мастера.

Теперь beta стала новым мастером. Проверяем состояние WAL:

```sql
SELECT timeline_id FROM pg_control_checkpoint();
```

Timeline будет увеличен (например, 2 вместо 1).

#### 3.1.4. Перенастройка старого мастера как реплики

Когда alpha восстановлен (железо/ОС), мы можем переподключить его как новую реплику от beta.

1. Останавливаем alpha (если он случайно запустился).  
2. Очищаем его каталог данных.  
3. На beta создаём новый слот репликации (например, `alpha_slot`).

```sql
CREATE_REPLICATION_SLOT alpha_slot PHYSICAL;
```

4. Выполняем `pg_basebackup` с beta на alpha:

```bash
sudo -u postgres pg_basebackup   -D /var/lib/postgresql/16/alpha   -R   -X stream   -C -S alpha_slot   -h 127.0.0.1 -p 5433 -U repuser
```

5. Запускаем alpha как реплику (на другом порту, например 5435).

```bash
sudo systemctl start postgresql@16-alpha
```

Проверяем на alpha:

```sql
SELECT pg_is_in_recovery();
```

```text
 pg_is_in_recovery
-------------------
 t
```

На новом мастере beta:

```sql
SELECT client_addr, state, sync_state
FROM pg_stat_replication;
```

**Вывод по 3.1:** после отказа мастера alpha реплика beta успешно стала новым мастером (`pg_promote()`), а старый мастер переинициализирован как реплика от нового мастера.

---

### 3.2. Каскадная репликация (Практика+)

#### 3.2.1. Настройка gamma как каскадной реплики от beta

1. На beta включён `wal_level = replica`, `max_wal_senders` и `max_replication_slots`.  
2. Создаём слот для gamma:

```sql
CREATE_REPLICATION_SLOT gamma_slot PHYSICAL;
```

3. На уровне ОС создаём каталог для gamma и выполняем `pg_basebackup` с beta:

```bash
sudo -u postgres pg_basebackup   -D /var/lib/postgresql/16/gamma   -R   -X stream   -C -S gamma_slot   -h 127.0.0.1 -p 5433 -U repuser
```

В `postgresql.auto.conf` gamma укажет `primary_conninfo` на beta.

В `postgresql.conf` gamma настраиваем задержку применения WAL:

```conf
recovery_min_apply_delay = '10s'
```

Запускаем gamma на порту 5434 и проверяем `pg_is_in_recovery()` — должна быть `t`.

#### 3.2.2. Демонстрация задержки и независимости от мастера

На beta (мастер для gamma) выполняем:

```sql
\\c repl_db
INSERT INTO t_sync VALUES (100, 'gamma delay test');
COMMIT;
```

На beta запись видна сразу, на gamma — только спустя ~10 секунд, что демонстрирует действие `recovery_min_apply_delay`.

Теперь останавливаем исходный мастер alpha (он уже не участвует в цепочке; база beta — актуальный мастер). При необходимости можно симулировать его отказ — gamma продолжает получать WAL с beta и тем самым не зависит от исходного alpha.

**Вывод по 3.2:** каскадная репликация позволяет строить цепочки реплик (alpha → beta → gamma), gamma получает WAL от beta и поддерживает заданную задержку применения изменений, что полезно для «защиты от человеческих ошибок» (можно успеть «отмотаться» назад).

---

## Модуль 4. Обзор кластерных технологий

### 4.1. Построение отказоустойчивого кластера на основе репликации

Классический подход:  
* один мастер, несколько реплик (standby), расположенных на разных серверах/узлах;  
* синхронные реплики — для гарантии, что данные записаны минимум на двух узлах;  
* асинхронные реплики — для географического резервирования и разгрузки чтения;  
* мониторинг состояния через `pg_stat_replication`, внешние системы (Prometheus, Zabbix).

Основные элементы:  
* физическая потоковая репликация (на уровне WAL);  
* слоты репликации для контроля удержания WAL;  
* механизм `pg_promote()` и `pg_rewind` для переключения и восстановления старого мастера.

### 4.2. Программы-менеджеры для автоматического переключения (Patroni и др.)

Ручной failover (как в Модуле 3) на практике неудобен; его автоматизируют с помощью менеджеров кластера:

* **Patroni** — использует распределённое хранилище (Etcd, Consul) для хранения информации о лидере и конфигурации, поддерживает автоматический failover/ switchover и интеграцию с HAProxy.  
* **repmgr** — специализированный менеджер для PostgreSQL с поддержкой регистрации реплик, мониторинга и failover.  
* **pg_auto_failover** — решение от Citus Data для автоматизации высокой доступности.

Менеджер:  
* следит за здоровьем мастера;  
* при недоступности мастера выбирает лучшую реплику для промоут;  
* перезапускает сервисы и обновляет конфигурацию прокси/балансировщиков.

### 4.3. Балансировка нагрузки между мастером и репликами

Типичный сценарий:  
* мастер обслуживает операции записи и критичные чтения;  
* реплики используются для отчётности, аналитики, тяжёлых SELECT;  
* между приложением и кластером ставится балансировщик (HAProxy, PgBouncer, Pgpool-II), который умеет направлять запросы на чтение к репликам, а запросы на запись — к мастеру.

Важно учитывать:  
* задержку репликации (replication lag);  
* возможную неконсистентность при чтении с реплик (особенно асинхронных);  
* необходимость корректной маршрутизации транзакций (не отправить запись на read-only-реплику).

### 4.4. Ограничения репликации в PostgreSQL

Ключевые ограничения:

* Физическая репликация целиком повторяет кластер: нельзя выбирать отдельные схемы/таблицы.  
* Логическая репликация не реплицирует: DDL (по умолчанию), последовательности (без дополнительных механизмов), многие системные объекты.  
* Нет «из коробки» полного решения для конфликт‑фри двунаправленной репликации — нужно вручную проектировать схему ключей/шардирования.  
* Реплика в режиме hot standby — только для чтения; выполнение некоторых команд (например, `VACUUM FULL`, `CLUSTER`) невозможно.  
* Для полной отказоустойчивости должны быть продуманы внешние компоненты: мониторинг, автоматический failover, резервное копирование, тестирование сценариев восстановления.

**Вывод по модулю 4:** репликация — лишь один слой в построении отказоустойчивости. Полноценный кластер требует менеджера кластера, балансировщиков нагрузки, разработанных процедур резервного копирования и восстановления, а также учёта ограничений физической и логической репликации.

---

## Итоговые выводы по лабораторной работе №9

1. **Физическая репликация** в синхронном режиме обеспечивает высокую надёжность данных, но при недоступности синхронной реплики может блокировать фиксацию транзакций на мастере.  
2. Параметры `max_standby_streaming_delay` и `hot_standby_feedback` управляют балансом между свежестью данных на репликах и стабильностью запросов: либо прерываются запросы на реплике, либо откладываются операции очистки на мастере.  
3. **Слоты репликации** предотвращают потерю WAL, необходимого репликам, но при длительной остановке реплики могут приводить к переполнению диска из-за накопления WAL.  
4. **Логическая репликация** позволяет избирательно реплицировать таблицы между базами и версиями PostgreSQL, но требует явной настройки публикаций и подписок и не покрывает все типы объектов.  
5. Двунаправленная логическая репликация возможна, но чревата конфликтами; она требует строгой дисциплины по разделению диапазонов ключей или дополнительных соглашений.  
6. Процедура **переключения (failover)** включает промоут реплики, переинициализацию старого мастера и перенастройку слотов репликации; без автоматизации это ручной и потенциально ошибочный процесс.  
7. **Каскадная репликация** и задержка применения WAL позволяют строить сложные топологии, включая узлы «отката» с задержанным применением изменений.  
8. Построение отказоустойчивого кластера на основе PostgreSQL требует дополнительных компонентов (Patroni, repmgr, HAProxy, PgBouncer и др.) и чёткого понимания ограничений репликации.  

Лабораторная работа выполнена: рассмотрены физическая и логическая репликация, конфликты применения WAL, слоты репликации, переключение мастера и основы кластерных технологий отказоустойчивости.
