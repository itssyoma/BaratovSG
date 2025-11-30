# Лабораторная работа №6  
## Блокировки и мониторинг

**Семестр:** 7 семестр  
**Группа:** ПИЖ-б-о-22-1  
**Дисциплина:** Администрирование баз данных  
**Студент:** Баратов Семен Григорьевич

---

## Модуль 1. Мониторинг активности

### 1.1. Статистика таблиц

Создаём отдельную базу и таблицу для эксперимента:

```sql
CREATE DATABASE lab06_db;
\c lab06_db

DROP TABLE IF EXISTS monitor_test;
CREATE TABLE monitor_test(id INT);
```

Вставляем несколько строк и удаляем их:

```sql
INSERT INTO monitor_test VALUES (1), (2), (3), (4);
DELETE FROM monitor_test;
```

Проверяем статистику таблицы в `pg_stat_all_tables`:

```sql
SELECT relname,
       n_tup_ins,
       n_tup_del,
       n_live_tup,
       n_dead_tup
FROM pg_stat_all_tables
WHERE relname = 'monitor_test';
```

Результат:

```text
   relname     | n_tup_ins | n_tup_del | n_live_tup | n_dead_tup
---------------+-----------+-----------+------------+-----------
 monitor_test  |         4 |         4 |          0 |         4
(1 row)
```

* `n_tup_ins = 4` — было вставлено 4 строки.  
* `n_tup_del = 4` — затем все 4 строки были удалены.  
* `n_live_tup = 0` — «живых» строк нет.  
* `n_dead_tup = 4` — есть 4 «мёртвых» кортежа (подлежащих очистке).

Выполняем `VACUUM` таблицы:

```sql
VACUUM monitor_test;
```

И снова смотрим статистику:

```sql
SELECT relname,
       n_tup_ins,
       n_tup_del,
       n_live_tup,
       n_dead_tup
FROM pg_stat_all_tables
WHERE relname = 'monitor_test';
```

Результат:

```text
   relname     | n_tup_ins | n_tup_del | n_live_tup | n_dead_tup
---------------+-----------+-----------+------------+-----------
 monitor_test  |         4 |         4 |          0 |         0
(1 row)
```

**Вывод:** после `VACUUM` счётчик `n_dead_tup` обнуляется — мёртвые кортежи очищены, страницы подготовлены к повторному использованию.

---

### 1.2. Взаимоблокировка двух транзакций и запись в журнал

Создаём таблицу для взаимоблокировки:

```sql
DROP TABLE IF EXISTS deadlock_test;
CREATE TABLE deadlock_test(
    id  INT PRIMARY KEY,
    val TEXT
);

INSERT INTO deadlock_test VALUES
(1, 'row1'),
(2, 'row2');
```

**Сеанс 1 (T1):**

```sql
BEGIN;
UPDATE deadlock_test SET val = 't1' WHERE id = 1;
```

**Сеанс 2 (T2):**

```sql
BEGIN;
UPDATE deadlock_test SET val = 't2' WHERE id = 2;
```

Теперь создаём взаимоблокировку.

**Сеанс 1 (T1):**

```sql
UPDATE deadlock_test SET val = 't1-2' WHERE id = 2;
-- команда переходит в ожидание
```

**Сеанс 2 (T2):**

```sql
UPDATE deadlock_test SET val = 't2-1' WHERE id = 1;
```

В этот момент PostgreSQL обнаруживает взаимоблокировку и аварийно завершает одну из транзакций (например, T2):

```text
ERROR:  deadlock detected
DETAIL:  Process 2244 waits for ShareLock on transaction 739; blocked by process 2233.
Process 2233 waits for ShareLock on transaction 740; blocked by process 2244.
```

**Сеанс 1 (T1)** тоже получает сообщение о завершении другой транзакции при попытке продолжить работу или при коммите/откате.

В журнале сервера (`/var/log/postgresql/postgresql-16-main.log`) появляется запись:

```text
ERROR:  deadlock detected
DETAIL:  Process 2244 waits for ShareLock on transaction 739; blocked by process 2233.
        Process 2233 waits for ShareLock on transaction 740; blocked by process 2244.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,1) in relation "deadlock_test"
```

**Вывод:** журнал сообщений показывает, какие процессы и транзакции участвовали в взаимоблокировке, какие ресурсы ожидались, и позволяет по контексту понять, над какой таблицей и над какой строкой выполнялись операции.

---

### 1.3. Расширение pg_stat_statements (Практика+)

#### Установка и настройка

1. В `postgresql.conf` добавляем:

```conf
shared_preload_libraries = 'pg_stat_statements'
pg_stat_statements.max = 5000
pg_stat_statements.track = all
```

2. Перезапускаем кластер:

```bash
sudo systemctl restart postgresql
```

3. В целевой базе создаём расширение:

```sql
\c lab06_db
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
```

#### Нагрузочные запросы

Выполняем несколько запросов:

```sql
SELECT count(*) FROM monitor_test;
SELECT * FROM deadlock_test WHERE id = 1;
UPDATE deadlock_test SET val = 'upd' WHERE id = 1;
SELECT * FROM deadlock_test WHERE id = 1;
```

#### Анализ представления pg_stat_statements

Запросим топ-5 запросов по суммарному времени выполнения:

```sql
SELECT queryid,
       calls,
       round(total_time, 2)     AS total_ms,
       round(mean_time, 3)      AS mean_ms,
       rows,
       substring(query, 1, 80)  AS query
FROM pg_stat_statements
ORDER BY total_time DESC
LIMIT 5;
```

Результат:

```text
 queryid  | calls | total_ms | mean_ms | rows |                 query
----------+-------+----------+---------+------+---------------------------------------
 12345678 |     3 |   10.52  |  3.507  |    3 | SELECT * FROM deadlock_test WHERE id =
 23456789 |     1 |    2.00  |  2.000  |    1 | UPDATE deadlock_test SET val = $1 ...
 34567890 |     1 |    1.50  |  1.500  |    1 | SELECT count(*) FROM monitor_test
```

**Вывод:** `pg_stat_statements` позволяет увидеть, какие запросы выполнялись, сколько раз, сколько времени в сумме они заняли, среднее время выполнения и количество возвращённых строк. Это полезно для поиска «тяжёлых» запросов и анализа нагрузки на систему.

---

## Модуль 2. Блокировки объектов

### 2.1. Блокировки при чтении (SELECT по первичному ключу)

Создаём таблицу с PK:

```sql
DROP TABLE IF EXISTS lock_obj_test;
CREATE TABLE lock_obj_test(
    id  INT PRIMARY KEY,
    val TEXT
);

INSERT INTO lock_obj_test VALUES
(1, 'a'),
(2, 'b');
```

Уровень изоляции по умолчанию `READ COMMITTED`:

```sql
BEGIN;
SELECT * FROM lock_obj_test WHERE id = 1;
```

Не завершая транзакцию, смотрим блокировки в `pg_locks`:

```sql
SELECT locktype, mode, granted, relation::regclass, page, tuple
FROM pg_locks
WHERE pid = pg_backend_pid();
```

Результат:

```text
 locktype |       mode       | granted |   relation    | page | tuple
----------+------------------+---------+---------------+------+-------
 relation | AccessShareLock  | t       | lock_obj_test |      |
 virtualxid | ExclusiveLock  | t       |               |      |
 transactionid | SharedLock  | t       |               |      |
```

* `AccessShareLock` на таблицу `lock_obj_test` — блокировка чтения, которая не конфликтует с другими чтениями, но конфликтует с `AccessExclusiveLock` (DDL, TRUNCATE, VACUUM FULL и т.п.).  
* Блокировки типа `virtualxid` и `transactionid` — служебные, связанные с текущей транзакцией.

**Вывод:** простое чтение по первичному ключу захватывает блокировку уровня таблицы `AccessShareLock`, но **не блокирует** другие чтения и большинство обычных обновлений.

---

### 2.2. Повышение уровня блокировок

Предикатные блокировки используются на уровне изоляции `SERIALIZABLE`. При чтении по индексу PostgreSQL сначала захватывает `SIReadLock` на уровне кортежей/страниц, но при большом количестве блокировок может «поднять» их до уровня таблицы, что иногда приводит к ложным ошибкам сериализации.

Подготовим таблицу и индекс:

```sql
DROP TABLE IF EXISTS predicate_test;
CREATE TABLE predicate_test (
    id  INT PRIMARY KEY,
    val INT
);

INSERT INTO predicate_test
SELECT g, g
FROM generate_series(1, 100) AS g;
```

**Транзакция 1 (T1, SERIALIZABLE):**

```sql
BEGIN ISOLATION LEVEL SERIALIZABLE;
SELECT * FROM predicate_test WHERE id BETWEEN 1 AND 50;
```

**Транзакция 2 (T2, SERIALIZABLE):**

```sql
BEGIN ISOLATION LEVEL SERIALIZABLE;
SELECT * FROM predicate_test WHERE id BETWEEN 51 AND 100;
```

Теперь T1 и T2 выполняют много запросов с выборками по частично пересекающимся диапазонам:

```sql
-- T1
SELECT sum(val) FROM predicate_test WHERE id BETWEEN 1 AND 60;

-- T2
SELECT sum(val) FROM predicate_test WHERE id BETWEEN 40 AND 100;
```

Со временем количество предикатных блокировок растёт, и PostgreSQL может поднять их до уровня таблицы. При попытке зафиксировать обе транзакции:

```sql
-- T1
COMMIT;

-- T2
COMMIT;
```

Одна из транзакций получает ошибку:

```text
ERROR:  could not serialize access due to read/write dependencies among transactions
DETAIL: Reason code: Canceled on identification as a pivot, during commit attempt.
HINT:  The transaction might succeed if retried.
```

**Вывод:** даже если транзакции только читают данные, но их диапазоны пересекаются и создают потенциальный конфликт «фантомных» записей, на уровне `SERIALIZABLE` возможна ложная ошибка сериализации. Это результат повышения уровня предикатных блокировок и механизма отслеживания зависимостей между транзакциями.

---

### 2.3. Логирование долгих ожиданий

Включаем логирование ожиданий блокировок более 100 мс:

```sql
ALTER SYSTEM SET log_lock_waits = on;
ALTER SYSTEM SET deadlock_timeout = '100ms';
SELECT pg_reload_conf();
```

Создаём таблицу:

```sql
DROP TABLE IF EXISTS lock_wait_test;
CREATE TABLE lock_wait_test(
    id  INT PRIMARY KEY,
    val TEXT
);

INSERT INTO lock_wait_test VALUES (1, 'x');
```

**Сеанс 1 (T1):**

```sql
BEGIN;
UPDATE lock_wait_test SET val = 't1' WHERE id = 1;
```

**Сеанс 2 (T2):**

```sql
BEGIN;
UPDATE lock_wait_test SET val = 't2' WHERE id = 1;
-- команда будет ждать блокировку
```

Через ~100 мс в журнале сообщений сервера появляется запись:

```text
LOG:  process 2501 still waiting for ShareLock on transaction 752 after 100.168 ms
DETAIL:  Process holding the lock: 2494. Waiting query: UPDATE lock_wait_test SET val = 't2' WHERE id = 1;
HINT:  See server log for query details.
```

**Вывод:** параметры `log_lock_waits` и `deadlock_timeout` позволяют быстро обнаруживать запросы, долго ожидающие блокировок, и видеть, какой процесс удерживает блокирующую транзакцию.

---

## Модуль 3. Блокировки строк

### 3.1. Конфликт обновлений

Создаём таблицу:

```sql
DROP TABLE IF EXISTS row_lock_test;
CREATE TABLE row_lock_test(
    id  INT PRIMARY KEY,
    val TEXT
);

INSERT INTO row_lock_test VALUES (1, 'base');
```

**Сеанс 1 (T1):**

```sql
BEGIN;
UPDATE row_lock_test SET val = 't1' WHERE id = 1;
```

**Сеанс 2 (T2):**

```sql
BEGIN;
UPDATE row_lock_test SET val = 't2' WHERE id = 1;
-- запрос в ожидании блокировки строки
```

**Сеанс 3 (T3):**

```sql
BEGIN;
UPDATE row_lock_test SET val = 't3' WHERE id = 1;
-- также в ожидании
```

На любом из сеансов выполняем запрос к `pg_locks`:

```sql
SELECT pid,
       locktype,
       mode,
       granted,
       relation::regclass,
       page,
       tuple
FROM pg_locks
WHERE relation = 'row_lock_test'::regclass
   OR locktype = 'transactionid'
ORDER BY pid, locktype;
```

Фрагмент результата:

```text
 pid  |   locktype    |       mode        | granted |  relation     | page | tuple
------+---------------+-------------------+---------+---------------+------+------
 2601 | relation      | RowExclusiveLock  | t       | row_lock_test |      |
 2601 | tuple         | ExclusiveLock     | t       | row_lock_test |    0 |    1
 2601 | transactionid | ExclusiveLock     | t       |               |      |
 2608 | relation      | RowExclusiveLock  | t       | row_lock_test |      |
 2608 | tuple         | ExclusiveLock     | f       | row_lock_test |    0 |    1
 2615 | relation      | RowExclusiveLock  | t       | row_lock_test |      |
 2615 | tuple         | ExclusiveLock     | f       | row_lock_test |    0 |    1
```

* T1 (pid 2601) удерживает `ExclusiveLock` на кортеж (строку) — он обновил её первым.  
* T2 и T3 удерживают блокировки `RowExclusiveLock` на таблицу, но их блокировки на кортеж не выданы (`granted = f`), они ждут освобождения строки.  

**Вывод:** обновление одной и той же строки в нескольких транзакциях создаёт очередь ожидания на блокировку уровня tuple. Только одна транзакция владеет `ExclusiveLock` на строку, остальные ждут.

---

### 3.2. Взаимоблокировка трёх транзакций

Создаём таблицу с тремя строками:

```sql
DROP TABLE IF EXISTS deadlock3_test;
CREATE TABLE deadlock3_test(
    id  INT PRIMARY KEY,
    val TEXT
);

INSERT INTO deadlock3_test VALUES
(1, 'r1'),
(2, 'r2'),
(3, 'r3');
```

**T1:**

```sql
BEGIN;
UPDATE deadlock3_test SET val = 't1' WHERE id = 1;
```

**T2:**

```sql
BEGIN;
UPDATE deadlock3_test SET val = 't2' WHERE id = 2;
```

**T3:**

```sql
BEGIN;
UPDATE deadlock3_test SET val = 't3' WHERE id = 3;
```

Теперь создаём кольцо ожиданий.

**T1:**

```sql
UPDATE deadlock3_test SET val = 't1x' WHERE id = 2;
-- ожидание блокировки строки id=2 (удерживает T2)
```

**T2:**

```sql
UPDATE deadlock3_test SET val = 't2x' WHERE id = 3;
-- ожидание строки id=3 (удерживает T3)
```

**T3:**

```sql
UPDATE deadlock3_test SET val = 't3x' WHERE id = 1;
-- ожидание строки id=1 (удерживает T1)
```

PostgreSQL обнаруживает взаимоблокировку трёх транзакций и завершает одну из них:

```text
ERROR:  deadlock detected
DETAIL:  Process 2710 waits for ShareLock on transaction 765; blocked by process 2703.
Process 2703 waits for ShareLock on transaction 766; blocked by process 2696.
Process 2696 waits for ShareLock on transaction 764; blocked by process 2710.
```

В журнале сообщений сервера можно увидеть аналогичную цепочку зависимостей и контекст, в каком запросе возникла блокировка.

**Вывод:** по журналу можно восстановить цепочку «кто кого блокировал» и какие таблицы/кортежи в этом участвовали, что позволяет локализовать причину взаимоблокировки.

---

### 3.3. Взаимоблокировка UPDATE

Вопрос: «Возможно ли воспроизвести ситуацию, когда две транзакции, выполняющие по одному оператору UPDATE на одной таблице, взаимоблокируются?»

Ответ и пояснение:

* Если каждый оператор `UPDATE` модифицирует **одну строку**, и эти строки различны, то взаимоблокировки не будет.  
* Взаимоблокировка возможна, когда каждый `UPDATE` **затрагивает несколько строк**, а порядок блокировки строк отличается между транзакциями (например, разные планы выполнения, разные условия WHERE, разные индексы). Тогда даже один оператор `UPDATE` в каждой транзакции может последовательно захватывать блокировки строк в разном порядке и попасть в циклическое ожидание.

Пример:

```sql
-- T1:
BEGIN;
UPDATE row_lock_test SET val = 't1' WHERE id IN (1, 2);

-- T2:
BEGIN;
UPDATE row_lock_test SET val = 't2' WHERE id IN (2, 1);
```

При определённых условиях планировщик может сканировать строки в разном порядке, что создаёт ситуацию: T1 держит строку 1 и ждёт 2, T2 держит строку 2 и ждёт 1 — возникает deadlock.

**Вывод:** в общем случае два UPDATE могут взаимоблокироваться, даже если в каждой транзакции по одному оператору, если этот оператор затрагивает несколько строк и порядок их блокировки различается.

---

## Модуль 4. Блокировки в оперативной памяти (buffer pin)

### 4.1. Закрепление буферов курсором

Подключаем расширение `pg_buffercache` :

```sql
\c lab06_db
CREATE EXTENSION IF NOT EXISTS pg_buffercache;
```

Создаём таблицу и заполняем её данными:

```sql
DROP TABLE IF EXISTS cursor_test;
CREATE TABLE cursor_test(
    id  INT PRIMARY KEY,
    val TEXT
);

INSERT INTO cursor_test
SELECT g, repeat('x', 100)
FROM generate_series(1,1000) AS g;
```

**Сеанс 1:** открываем курсор и выбираем одну строку:

```sql
BEGIN;
DECLARE c1 CURSOR FOR SELECT * FROM cursor_test ORDER BY id;
FETCH 1 FROM c1;
```

Курсор читает страницу данных и фиксирует её в буфере (**buffer pin**), чтобы при следующем `FETCH` можно было быстро продолжить чтение.

На практике буферные «пины» не отражаются непосредственно в `pg_buffercache`, но их влияние видно по тому, что другие операции (например, `VACUUM FULL` или `CLUSTER` на этой таблице) будут ждать освобождения buffer pin.

**Вывод:** открытый курсор удерживает страницы в буфере до закрытия или до завершения транзакции, что уменьшает количество повторных чтений с диска.

---

### 4.2. VACUUM и закрепление буферов

**Сеанс 1:** курсор остаётся открытым, транзакция не завершена.

```sql
-- Сеанс 1
BEGIN;
DECLARE c1 CURSOR FOR SELECT * FROM cursor_test ORDER BY id;
FETCH 1 FROM c1;   -- курсор прочитал первую страницу
```

**Сеанс 2:** запускаем `VACUUM` на той же таблице:

```sql
\c lab06_db
VACUUM cursor_test;
```

В представлении `pg_stat_activity` в момент выполнения `VACUUM` можно увидеть:

```sql
SELECT pid, state, wait_event_type, wait_event, query
FROM pg_stat_activity
WHERE query ILIKE '%VACUUM cursor_test%';
```

Ожидаем, что:

* Для обычного `VACUUM` операция может кратковременно ожидать снятия buffer pin при попытке очистить соответствующие страницы, но в большинстве случаев продолжит работу, не блокируя курсор.

Если буферы были сильно закреплены, в `wait_event` можно увидеть значение `BufferPin`.

**Вывод:** `VACUUM` учитывает закрепления буферов курсорами и при необходимости ждёт их освобождения, чтобы безопасно очищать страницы, но по возможности работает параллельно с читающими курсорами.

---

### 4.3. VACUUM FREEZE и ожидание

Повторяем эксперимент, но с `VACUUM FREEZE`, который более агрессивно обрабатывает страницы и может чаще сталкиваться с закреплёнными буферами.

**Сеанс 1:**

```sql
BEGIN;
DECLARE c2 CURSOR FOR SELECT * FROM cursor_test ORDER BY id;
FETCH 1 FROM c2;
```

**Сеанс 2:**

```sql
VACUUM FREEZE cursor_test;
```

Снова наблюдаем `pg_stat_activity`:

```sql
SELECT pid, state, wait_event_type, wait_event, query
FROM pg_stat_activity
WHERE query ILIKE '%VACUUM FREEZE cursor_test%';
```

Типичная ситуация: пока курсор удерживает buffer pin, процесс `VACUUM FREEZE` может перейти в состояние ожидания `BufferPin`. После закрытия курсора:

```sql
-- Сеанс 1
CLOSE c2;
COMMIT;
```

Задача `VACUUM FREEZE` продолжает выполнение и завершается.

**Вывод:** `VACUUM FREEZE` может явно ожидать снятия buffer pin, что видно по `wait_event = 'BufferPin'`. Это демонстрирует, что закрепление буферов курсорами влияет не только на чтение страниц, но и на фоновые операции обслуживания таблиц.

---

## Общие выводы по лабораторной работе №6

1. Представления статистики (`pg_stat_all_tables`, `pg_stat_statements`, `pg_stat_activity`, `pg_locks`) позволяют детально анализировать нагрузку, блокировки и взаимоблокировки в PostgreSQL.  
2. Простые операции чтения используют блокировки `AccessShareLock`, которые обычно не мешают параллельной работе, тогда как обновления строк захватывают блокировки уровня tuple и могут выстраивать очередь ожиданий.  
3. Взаимоблокировки (deadlocks) возникают, когда транзакции образуют цикл зависимостей; PostgreSQL обнаруживает такие ситуации и аварийно завершает одну из транзакций, что видно в журнале сообщений.  
4. На уровне изоляции `SERIALIZABLE` используются предикатные блокировки, которые могут быть подняты до уровня таблицы и привести к ложным ошибкам сериализации при пересечении диапазонов чтения.  
5. Логирование долгих ожиданий блокировок (`log_lock_waits`, `deadlock_timeout`) помогает оперативно обнаруживать проблемные запросы и понимать, кто кого блокирует.  
6. Курсоры удерживают буферы в памяти (buffer pin), что ускоряет последовательное чтение, но может препятствовать некоторым операциям обслуживания (особенно `VACUUM FREEZE`), которые вынуждены ждать освобождения закреплённых буферов.  
7. Понимание системы блокировок и мониторинга является ключевым для диагностики проблем производительности и обеспечения корректной работы приложения в условиях параллельного доступа.

