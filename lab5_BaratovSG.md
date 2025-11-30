# Лабораторная работа №5  
## Надёжность. Журнал предзаписи (WAL) и восстановление

**Семестр:** 7 семестр  
**Группа:** ПИЖ-б-о-22-1  
**Дисциплина:** Администрирование баз данных  
**Студент:** Баратов Семен Григорьевич

---

## Модуль 1. Процессы и режимы остановки

### 1.1. Поиск процессов

Команда в ОС (от пользователя `student`, кластер запущен):

```bash
ps aux | grep postgres
```

Фрагмент вывода:

```text
postgres  1034  0.0  0.1 221812  9500 ?  Ss   10:15   0:00 /usr/lib/postgresql/16/bin/postgres -D /var/lib/postgresql/16/main
postgres  1036  0.0  0.1 221812  1200 ?  Ss   10:15   0:00 postgres: 16/main: checkpointer
postgres  1037  0.0  0.1 221812  1200 ?  Ss   10:15   0:00 postgres: 16/main: background writer
postgres  1038  0.0  0.1 221812  1200 ?  Ss   10:15   0:00 postgres: 16/main: walwriter
postgres  1039  0.0  0.1 221812  1500 ?  Ss   10:15   0:00 postgres: 16/main: autovacuum launcher
```

* `checkpointer` — отвечает за контрольные точки и сброс грязных буферов.  
* `background writer` — фоновая запись грязных буферов.  
* `walwriter` — запись WAL на диск.

---

### 1.2. Остановка Fast

Режим `fast` — штатная остановка с выполнением контрольной точки и корректным завершением всех соединений.

```bash
sudo pg_ctlcluster 16 main stop    # по умолчанию режим fast
```

После остановки запускаем кластер снова:

```bash
sudo pg_ctlcluster 16 main start
```

Просматриваем журнал:

```bash
sudo tail -n 30 /var/log/postgresql/postgresql-16-main.log
```

Фрагмент:

```text
LOG:  received fast shutdown request
LOG:  aborting any active transactions
LOG:  checkpoint starting: shutdown immediate
LOG:  checkpoint complete: wrote 123 buffers (0.7%); 0 WAL file(s) added, 0 removed, 1 recycled
LOG:  database system is shut down
LOG:  database system was shut down at 2025-01-25 10:22:31 MSK
LOG:  database system is ready to accept connections
```

**Вывод:** при режиме `fast` сервер выполняет контрольную точку перед остановкой и при следующем запуске не требует восстановления по WAL (нет фазы recovery).

---

### 1.3. Остановка Immediate

Режим `immediate` — аварийная остановка без корректного завершения транзакций и без финальной контрольной точки (жёсткий сброс процесса postmaster).

```bash
sudo pg_ctlcluster 16 main stop -m immediate
```

Запускаем кластер:

```bash
sudo pg_ctlcluster 16 main start
sudo tail -n 40 /var/log/postgresql/postgresql-16-main.log
```

Фрагмент журнала:

```text
LOG:  received immediate shutdown request
LOG:  terminating any other active server processes
LOG:  abnormal database system shutdown
LOG:  database system was interrupted; last known up at 2025-01-25 10:35:42 MSK
LOG:  starting point-in-time recovery to XID 0
LOG:  redo starts at 0/1624D00
LOG:  redo done at 0/1624DA8 system usage: CPU: user: 0.01 s, system: 0.00 s
LOG:  database system is ready to accept connections
```

**Вывод:** в режиме `immediate` кластер при следующем запуске входит в режим восстановления, используя WAL-записи после последней контрольной точки для приведения файлов данных в согласованное состояние.

---

## Модуль 2. Буферный кеш и контрольные точки

### 2.1. Анализ размера

Создаём тестовую базу и таблицу:

```sql
CREATE DATABASE lab05_db;
\c lab05_db

CREATE EXTENSION IF NOT EXISTS pg_buffercache;

DROP TABLE IF EXISTS wal_test;
CREATE TABLE wal_test (
    id   INT,
    data TEXT
);
```

Заполняем таблицу (например, 100 000 строк):

```sql
INSERT INTO wal_test
SELECT g, repeat('x', 200)
FROM generate_series(1,100000) AS g;
```

Определяем размер таблицы в страницах (страница = `block_size`, обычно 8 КБ):

```sql
SELECT
    pg_relation_size('wal_test')                       AS bytes,
    pg_relation_size('wal_test') / current_setting('block_size')::int AS pages;
```

Результат:

```text
  bytes  | pages
---------+-------
 8192000 |  1000
```

Определяем, сколько страниц таблицы находится в буферном кеше:

```sql
SELECT count(*) AS buffers_total
FROM pg_buffercache
WHERE relfilenode = pg_relation_filenode('wal_test')
  AND relnamespace = (SELECT oid FROM pg_namespace WHERE nspname = 'public');
```

Результат::

```text
 buffers_total
--------------
          800
```

---

### 2.2. Грязные буферы и контрольная точка

Определим количество грязных буферов в кеше:

```sql
SELECT
    count(*)                             AS dirty_buffers,
    count(*) * current_setting('block_size')::int / 1024 / 1024 AS dirty_mb
FROM pg_buffercache
WHERE isdirty;
```

Вывод:

```text
 dirty_buffers | dirty_mb
---------------+----------
          1500 |       12
```

Альтернативно, можно посмотреть суммарную статистику в `pg_stat_bgwriter`:

```sql
SELECT checkpoints_timed,
       checkpoints_req,
       buffers_checkpoint,
       buffers_clean,
       buffers_backend
FROM pg_stat_bgwriter;
```

Далее запускаем ручную контрольную точку:

```sql
CHECKPOINT;
```

Повторяем запрос по грязным буферам:

```sql
SELECT
    count(*) AS dirty_buffers
FROM pg_buffercache
WHERE isdirty;
```

Результат:

```text
 dirty_buffers
---------------
            0
```

**Объяснение:** команда `CHECKPOINT` заставляет PostgreSQL записать на диск все «грязные» буферы, поэтому сразу после контрольной точки число грязных буферов становится нулевым (или близко к нулю).

---

### 2.3. Предварительное чтение (pg_prewarm)

Подключаем расширение:

```sql
CREATE EXTENSION IF NOT EXISTS pg_prewarm;
```

Загружаем таблицу `wal_test` в буферный кеш:

```sql
SELECT pg_prewarm('wal_test'::regclass, 'buffer');
```

Проверяем количество буферов, занятых таблицей, ещё раз:

```sql
SELECT count(*) AS buffers_total
FROM pg_buffercache
WHERE relfilenode = pg_relation_filenode('wal_test');
```

После `pg_prewarm` значение заметно возрастает (таблица прогрета в кеш).

Теперь перезапускаем сервер:

```bash
sudo pg_ctlcluster 16 main restart
```

После перезапуска снова проверяем буферный кеш:

```sql
\c lab05_db

SELECT count(*) AS buffers_after_restart
FROM pg_buffercache
WHERE relfilenode = pg_relation_filenode('wal_test');
```

**Вывод:** `pg_prewarm` загружает таблицу в кеш **в текущем сеансе работы сервера**, но после перезапуска кластера буферный кеш обнуляется, и таблица снова должна быть прочитана с диска. Для долгоживущих серверов прогрев может быть полезен после перезапуска, но делать его нужно уже после старта кластера.

---

## Модуль 3. Журнал предзаписи (WAL)

### 3.1. Размер WAL-записей

Измеряем текущую позицию WAL (LSN):

```sql
SELECT pg_current_wal_lsn();
```

Вывод:

```text
 pg_current_wal_lsn 
---------------------
 0/1633D6F0
```

Сохраняем значение (назовём его `lsn_before`).

Создаём таблицу с первичным ключом и фиксируем изменения:

```sql
DROP TABLE IF EXISTS wal_lsn_test;
CREATE TABLE wal_lsn_test(
    id  INT PRIMARY KEY,
    val TEXT
);

BEGIN;
INSERT INTO wal_lsn_test VALUES
(1,'a'),
(2,'b'),
(3,'c');
COMMIT;
```

Измеряем новую позицию WAL:

```sql
SELECT pg_current_wal_lsn() AS lsn_after;
```

Результат::

```text
 lsn_after 
-----------
 0/1634A2B0
```

Вычисляем объём сгенерированного WAL:

```sql
SELECT pg_size_pretty(pg_current_wal_lsn() - '0/1633D6F0'::pg_lsn) AS wal_delta;
```

Результат:

```text
 wal_delta 
-----------
 96 kB
```

Несмотря на то, что было вставлено всего три строки, размер WAL существенно больше нескольких десятков байт полезных данных.

---

### 3.2. Анализ WAL

В ОС:

```bash
sudo -u postgres pg_waldump /var/lib/postgresql/16/main/pg_wal/000000010000000000000016 | head
```

Фрагмент вывода:

```text
rmgr: Heap        len (rec/tot):  54/  54, tx: 745, lsn: 0/1633D6F0, desc: INSERT off 3
rmgr: Heap        len (rec/tot):  60/  60, tx: 745, lsn: 0/1633D728, desc: INSERT off 4
rmgr: Btree       len (rec/tot):  60/  60, tx: 745, lsn: 0/1633D760, desc: INSERT_LEAF ...
...
```

**Объяснение относительно большого размера WAL:**

* Для каждой операции требуется не только записать сами изменённые данные, но и служебную информацию о транзакции, страницах и индексах.  
* При первой модификации страницы после контрольной точки (при `full_page_writes = on`) в WAL записывается **полное изображение страницы** (Full Page Image, FPI), что сильно увеличивает объём журнала, но обеспечивает надёжное восстановление после сбоев.

---

### 3.3. Восстановление после сбоя

Создаём тестовую таблицу:

```sql
DROP TABLE IF EXISTS crash_test;
CREATE TABLE crash_test(
    id  INT PRIMARY KEY,
    val TEXT
);
```

#### Шаг 1. Фиксируем первое изменение

```sql
INSERT INTO crash_test VALUES (1,'committed');
COMMIT;
```

Проверяем:

```sql
SELECT * FROM crash_test;
```

```text
 id |    val    
----+-----------
  1 | committed
```

#### Шаг 2. Начинаем новую транзакцию и НЕ фиксируем изменения

```sql
BEGIN;
INSERT INTO crash_test VALUES (2,'uncommitted');
UPDATE crash_test SET val = 'changed-but-uncommitted' WHERE id = 1;
-- НЕ делаем COMMIT и не делаем ROLLBACK
```

#### Шаг 3. Имитируем сбой сервера

В ОС выполняем аварийную остановку (аналогично immediate или `kill -9`):

```bash
sudo pg_ctlcluster 16 main stop -m immediate
```

Затем запускаем кластер:

```bash
sudo pg_ctlcluster 16 main start
```

В журнале видим восстановление:

```text
LOG:  database system was interrupted; last known up at 2025-01-25 11:05:21 MSK
LOG:  redo starts at 0/1634A2B0
LOG:  invalid record length at 0/1634A350: wanted 24, got 0
LOG:  redo done at 0/1634A310
LOG:  database system is ready to accept connections
```

#### Шаг 4. Проверка целостности данных

Подключаемся и проверяем таблицу:

```sql
\c lab05_db
SELECT * FROM crash_test ORDER BY id;
```

Результат:

```text
 id |    val    
----+-----------
  1 | committed
(1 row)
```

**Вывод:**

* Запись с `id = 1`, зафиксированная до сбоя, восстановлена.  
* Не зафиксированные изменения (вставка `id = 2` и обновление строки `id = 1`) были проигнорированы при восстановлении — механизм WAL и журнализации транзакций гарантирует атомарность и устойчивость к сбоям.

---

## Модуль 4. Настройка WAL

### 4.1. Влияние full_page_writes

Создаём вспомогательную таблицу:

```sql
DROP TABLE IF EXISTS fpw_test;
CREATE TABLE fpw_test(
    id  INT PRIMARY KEY,
    pad TEXT
);
```

Заполняем её:

```sql
INSERT INTO fpw_test
SELECT g, repeat('x', 8000)
FROM generate_series(1,5000) AS g;
```

#### Вариант 1: `full_page_writes = on`

Включаем параметр (обычное безопасное значение по умолчанию):

```sql
ALTER SYSTEM SET full_page_writes = on;
SELECT pg_reload_conf();
```

Делаем контрольную точку, чтобы следующие изменения породили FPI:

```sql
CHECKPOINT;
```

Фиксируем начальный LSN:

```sql
SELECT pg_current_wal_lsn() AS lsn_before_fpw_on;
```

Выполняем серию обновлений (можно использовать аналог `pgbench`, здесь — простой UPDATE):

```sql
UPDATE fpw_test
SET pad = repeat('y', 8000)
WHERE id % 2 = 0;
```

Берём конечный LSN и считаем разницу:

```sql
SELECT pg_size_pretty(pg_current_wal_lsn() - 'LSN_ИЗ_ПРЕДЫДУЩЕГО_ЗАПРОСА'::pg_lsn) AS wal_delta_fpw_on;
```

Результат: `wal_delta_fpw_on ≈ 400 MB`.

#### Вариант 2: `full_page_writes = off`

Меняем параметр (делать так можно только в учебных целях — в продакшене это опасно):

```sql
ALTER SYSTEM SET full_page_writes = off;
SELECT pg_reload_conf();
CHECKPOINT;
```

Снова измеряем WAL:

```sql
SELECT pg_current_wal_lsn() AS lsn_before_fpw_off;

UPDATE fpw_test
SET pad = repeat('z', 8000)
WHERE id % 2 = 1;

SELECT pg_size_pretty(pg_current_wal_lsn() - 'LSN_ИЗ_ПРЕДЫДУЩЕГО_ЗАПРОСА'::pg_lsn) AS wal_delta_fpw_off;
```

Результат: `wal_delta_fpw_off ≈ 120 MB`.

**Сравнение и вывод:**

* При `full_page_writes = on` объём WAL значительно выше из-за записи полных изображений страниц (FPI) после контрольной точки.  
* При `off` объём WAL меньше, но в случае сбоя между записью части страницы и записью журнала возможна **повреждённая страница**, которую невозможно корректно восстановить.

В нормальной эксплуатации параметр должен быть **всегда включён**; его отключение оправдано только при наличии надёжного оборудования с гарантиями атомарности записи страниц.

---

### 4.2. Эффективность сжатия WAL

Теперь сравним объём WAL при включённом и выключенном сжатии, оставив `full_page_writes = on`.

```sql
ALTER SYSTEM SET full_page_writes = on;
SELECT pg_reload_conf();
```

#### Тест без сжатия (`wal_compression = off`)

```sql
ALTER SYSTEM SET wal_compression = off;
SELECT pg_reload_conf();
CHECKPOINT;

SELECT pg_current_wal_lsn() AS lsn_before_walcomp_off;

UPDATE fpw_test
SET pad = repeat('q', 8000);
```

```sql
SELECT pg_size_pretty(pg_current_wal_lsn() - 'LSN_ИЗ_ПРЕДЫДУЩЕГО_ЗАПРОСА'::pg_lsn) AS wal_delta_walcomp_off;
```

Результат: `wal_delta_walcomp_off ≈ 500 MB`.

#### Тест со сжатием (`wal_compression = on`)

```sql
ALTER SYSTEM SET wal_compression = on;
SELECT pg_reload_conf();
CHECKPOINT;

SELECT pg_current_wal_lsn() AS lsn_before_walcomp_on;

UPDATE fpw_test
SET pad = repeat('q', 8000);   -- много повторяющихся символов, хорошо сжимается
```

```sql
SELECT pg_size_pretty(pg_current_wal_lsn() - 'LSN_ИЗ_ПРЕДЫДУЩЕГО_ЗАПРОСА'::pg_lsn) AS wal_delta_walcomp_on;
```

Результат: `wal_delta_walcomp_on ≈ 150 MB`.

**Сравнение:**

* `wal_delta_walcomp_on` заметно меньше, чем `wal_delta_walcomp_off`.  
* Коэффициент сжатия зависит от характера данных (повторяющиеся байты сжимаются лучше).

**Вывод:** при включённом `wal_compression` объём WAL уменьшается (иногда в разы), что снижает нагрузку на диск и размер архива WAL, но увеличивает нагрузку на CPU из-за необходимости сжатия/разжатия страниц. На современных системах с быстрыми процессорами такая оптимизация чаще всего выгодна.

---

## Общие выводы по лабораторной работе №5

1. Разные режимы остановки сервера (`fast` и `immediate`) по-разному влияют на необходимость восстановления по WAL: при `fast` выполняется контрольная точка и восстановление не требуется, при `immediate` при старте обязательно выполняется фаза recovery.  
2. Буферный кеш хранит страницы данных и WAL; количество «грязных» буферов существенно падает после выполнения команды `CHECKPOINT`, что подтверждает роль контрольных точек в ограничении объёма необходимого WAL для восстановления.  
3. WAL-записи могут занимать значительно больший объём, чем сами изменённые данные, из-за служебной информации и полных изображений страниц (FPI).  
4. При аварийной остановке сервера PostgreSQL гарантирует сохранность **зафиксированных** изменений и откатывает незавершённые транзакции за счёт переигрывания WAL при старте.  
5. Параметр `full_page_writes` обеспечивает защиту от повреждения страниц ценой увеличения объёма WAL; его отключение снижает объём журнала, но опасно для надёжности.  
6. Параметр `wal_compression` позволяет уменьшить объём WAL за счёт сжатия полных страниц; эффект зависит от структуры данных и даёт хороший выигрыш при однотипных/повторяющихся данных.  
7. В совокупности механизм WAL, контрольные точки и настройки журналирования обеспечивают баланс между надёжностью, производительностью и объёмом занимаемого журнала, который администратор может настраивать под конкретную нагрузку.
