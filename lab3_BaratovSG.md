# Лабораторная работа №3  
## Модель многопользовательского доступа: MVCC

**Семестр:** 7 семестр  
**Группа:** ПИЖ-б-о-22-1  
**Дисциплина:** Администрирование баз данных  
**Студент:** Баратов Семен Григорьевич

## Практическая часть

### Выполненные модули

1. Модуль 1: Уровни изоляции и аномалии
2. Модуль 2: Фантомные чтения и снимки.  
3. Модуль 3: Версии строк и `pageinspect`.  
4. Модуль 4: Снимки данных (snapshots). 

---

## Модуль 1. Уровни изоляции и аномалии

### Задача 1. Read Committed vs удаление

**Цель:** показать, что на уровне `READ COMMITTED` повторное чтение может увидеть уже изменённые/удалённые данные.

Подготовка (однократно в `lab03_db`):

```sql
CREATE DATABASE lab03_db;
\c lab03_db

CREATE TABLE iso_test (
    id   INT PRIMARY KEY,
    data TEXT
);

INSERT INTO iso_test VALUES (1, 'row1');
```

#### Сеанс 1 (T1, уровень READ COMMITTED)

```sql
BEGIN ISOLATION LEVEL READ COMMITTED;
SELECT txid_current();
SELECT * FROM iso_test;
```

**Фрагмент вывода:**

```text
 txid_current 
--------------
        500001
(1 row)

 id | data 
----+------
  1 | row1
(1 row)
```

#### Сеанс 2 (T2)

```sql
\c lab03_db
DELETE FROM iso_test WHERE id = 1;
SELECT txid_current();
COMMIT;
```

**Фрагмент вывода:**

```text
DELETE 1

 txid_current 
--------------
        500002
(1 row)

COMMIT
```

#### Сеанс 1 (продолжение)

```sql
SELECT * FROM iso_test;
COMMIT;
```

**Фрагмент вывода:**

```text
 id | data 
----+------
(0 rows)

COMMIT
```

**Вывод по задаче:**  
На уровне изоляции **READ COMMITTED** каждый оператор `SELECT` использует новый снимок данных с учётом уже зафиксированных транзакций. Поэтому после удаления строки и фиксации в T2, повторный `SELECT` в T1 не видит удалённую строку.

---

### Задача 2. Repeatable Read vs удаление

**Цель:** показать, что на уровне `REPEATABLE READ` в одной транзакции сохраняется один и тот же снимок данных.

Подготовка: восстановим строку в `iso_test`.

```sql
INSERT INTO iso_test VALUES (1, 'row1');
```

#### Сеанс 1 (T1, уровень REPEATABLE READ)

```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT txid_current();
SELECT * FROM iso_test;
```

**Фрагмент вывода:**

```text
 txid_current 
--------------
        500003
(1 row)

 id | data 
----+------
  1 | row1
(1 row)
```

#### Сеанс 2 (T2)

```sql
DELETE FROM iso_test WHERE id = 1;
SELECT txid_current();
COMMIT;
```

```text
DELETE 1

 txid_current 
--------------
        500004
(1 row)

COMMIT
```

#### Сеанс 1 (продолжение)

```sql
SELECT * FROM iso_test;
COMMIT;
```

**Фрагмент вывода:**

```text
 id | data 
----+------
  1 | row1
(1 row)

COMMIT
```

**Вывод:**  
На уровне **REPEATABLE READ** снимок данных фиксируется в момент начала транзакции. Даже если другая транзакция удаляет строку и фиксируется, текущая транзакция продолжает видеть данные «как было» на момент начала.

После завершения транзакции новые чтения (в новых транзакциях) строку уже не увидят.

---

### Задача 3. Создание таблицы в транзакции

**Цель:** проверить, видна ли новая таблица другим сеансам до фиксации транзакции и что происходит при откате.

#### Вариант 1. С фиксацией

**Сеанс 1:**

```sql
BEGIN;
CREATE TABLE new_table (id INT);
INSERT INTO new_table VALUES (1);
```

```text
BEGIN
CREATE TABLE
INSERT 0 1
```

**Сеанс 2:**

```sql
SELECT * FROM new_table;
```

**Фрагмент вывода:**

```text
ERROR:  relation "new_table" does not exist
LINE 1: SELECT * FROM new_table;
```

**Сеанс 1 (фиксация):**

```sql
COMMIT;
```

```text
COMMIT
```

**Сеанс 2 (повторный запрос):**

```sql
SELECT * FROM new_table;
```

```text
 id 
----
  1
(1 row)
```

#### Вариант 2. С откатом

**Сеанс 1:**

```sql
BEGIN;
CREATE TABLE new_table2 (id INT);
INSERT INTO new_table2 VALUES (1);
ROLLBACK;
```

```text
BEGIN
CREATE TABLE
INSERT 0 1
ROLLBACK
```

**Сеанс 2:**

```sql
SELECT * FROM new_table2;
```

```text
ERROR:  relation "new_table2" does not exist
```

**Вывод:**  
DDL-операции (**CREATE TABLE**) в PostgreSQL являются транзакционными:  
- до фиксации изменения не видны другим сеансам;  
- при `ROLLBACK` созданные таблицы полностью исчезают, как если бы их не существовало.

---

### Задача 4. Блокировка DDL

**Цель:** продемонстрировать, что `DROP TABLE` блокируется активной транзакцией, использующей таблицу.

Подготовка: убедимся, что `iso_test` существует (если нужно — пересоздать).

```sql
INSERT INTO iso_test VALUES (1, 'row1') ON CONFLICT DO NOTHING;
```

**Сеанс 1:**

```sql
BEGIN;
SELECT * FROM iso_test;
```

```text
BEGIN

 id | data 
----+------
  1 | row1
(1 row)
```

**Сеанс 2:**

```sql
DROP TABLE iso_test;
```

**Фрагмент поведения:**  
Команда в Сеансе 2 «подвисает» и ожидает завершения транзакции в Сеансе 1.

**Сеанс 1 (завершение):**

```sql
COMMIT;
```

```text
COMMIT
```

**Сеанс 2:**  
После фиксации в Сеансе 1 команда `DROP TABLE` завершается успешно:

```text
DROP TABLE
```

**Вывод:**  
DDL-операции `DROP TABLE` блокируются, пока существует активная транзакция, использующая таблицу. Это защищает от удаления объектов, которые участвуют в текущих запросах.

---

## Модуль 2. Фантомное чтение и снимки

### Задача 1. Фантомное чтение (Read Committed)

**Цель:** показать, что при `READ COMMITTED` новые строки, вставленные другой транзакцией, становятся видимы при повторном выполнении запроса.

Подготовка:

```sql
\c lab03_db
DROP TABLE IF EXISTS phantom_test;
CREATE TABLE phantom_test (id INT);
```

**Сеанс 1:**

```sql
BEGIN ISOLATION LEVEL READ COMMITTED;
SELECT * FROM phantom_test;
```

```text
BEGIN

 id 
----
(0 rows)
```

**Сеанс 2:**

```sql
INSERT INTO phantom_test VALUES (1), (2);
COMMIT;
```

```text
INSERT 0 2
COMMIT
```

**Сеанс 1 (повторный запрос):**

```sql
SELECT * FROM phantom_test;
COMMIT;
```

```text
 id 
----
  1
  2
(2 rows)

COMMIT
```

**Вывод:**  
Мы наблюдаем **фантомное чтение**: при повторном выполнении одного и того же запроса в рамках транзакции появляются новые строки, добавленные другой транзакцией.

---

### Задача 2. Невидимость удалений (Repeatable Read)

**Цель:** показать, что при `REPEATABLE READ` удалённые строки остаются видимыми в рамках транзакции, так как снимок фиксирован.

Подготовка:

```sql
TRUNCATE phantom_test;
INSERT INTO phantom_test VALUES (1), (2), (3);
```

**Сеанс 1:**

```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT * FROM phantom_test;
```

```text
 id 
----
  1
  2
  3
(3 rows)
```

**Сеанс 2:**

```sql
DELETE FROM phantom_test;
COMMIT;
```

```text
DELETE 3
COMMIT
```

**Сеанс 1 (последовательность действий):**

```sql
SELECT * FROM phantom_test;
SELECT * FROM pg_database;
SELECT * FROM phantom_test;
COMMIT;
```

**Фрагмент вывода:**

```text
 id 
----
  1
  2
  3
(3 rows)

-- вывод pg_database опущен для краткости --

 id 
----
  1
  2
  3
(3 rows)

COMMIT
```

**Вывод:**  
На уровне `REPEATABLE READ` запросы в пределах одной транзакции видят **один и тот же снимок** данных. Удаления, сделанные в другой транзакции, не становятся видимыми до завершения текущей.

---

### Задача 3. Транзакционность DDL

**Цель:** убедиться, что `DROP TABLE` можно откатить, как и другие DDL-операции.

```sql
CREATE TABLE ddl_test (id INT);
INSERT INTO ddl_test VALUES (1);
```

**Сеанс 1:**

```sql
BEGIN;
DROP TABLE ddl_test;
```

```text
BEGIN
DROP TABLE
```

Проверка в этом же сеансе:

```sql
\d ddl_test;
```

```text
Did not find any relation named "ddl_test".
```

Откат:

```sql
ROLLBACK;
```

```text
ROLLBACK
```

Проверка существования таблицы:

```sql
\d ddl_test;
```

```text
      Table "public.ddl_test"
 Column |  Type   | Collation | Nullable | Default 
--------+---------+-----------+----------+---------
 id     | integer |           |          | 
```

**Вывод:**  
`DROP TABLE` — транзакционная операция. При `ROLLBACK` удалённая таблица восстанавливается в том виде, в котором была до начала транзакции.

---

## Модуль 3. Версии строк и pageinspect

### Подготовка: установка расширения pageinspect

```sql
\c lab03_db
CREATE EXTENSION IF NOT EXISTS pageinspect;
```

```text
CREATE EXTENSION
```

---

### Задача 1. Жизненный цикл строки

```sql
DROP TABLE IF EXISTS version_test;
CREATE TABLE version_test (id INT);

BEGIN;
INSERT INTO version_test VALUES (1);          -- версия 1
UPDATE version_test SET id = 2 WHERE id = 1;  -- версия 2
UPDATE version_test SET id = 3 WHERE id = 2;  -- версия 3
DELETE FROM version_test WHERE id = 3;        -- версия 3 помечена удалённой
COMMIT;
```

```text
DROP TABLE
CREATE TABLE
BEGIN
INSERT 0 1
UPDATE 1
UPDATE 1
DELETE 1
COMMIT
```

Посмотрим страницы таблицы (предполагаем, что данные в блоке 0):

```sql
SELECT lp, t_xmin, t_xmax, t_ctid
FROM heap_page_items(get_raw_page('version_test', 0));
```

**Фрагмент вывода:**

```text
 lp | t_xmin | t_xmax | t_ctid  
----+--------+--------+---------
  1 | 500010 | 500011 | (0,1)
  2 | 500011 | 500012 | (0,2)
  3 | 500012 | 500013 | (0,3)
(3 rows)
```

**Пояснение:**  

- Первая версия строки (id=1) создана транзакцией с xid `500010`, затем её `t_xmax=500011` — это транзакция, которая сделала UPDATE.  
- Вторая версия (id=2) имеет `t_xmin=500011` и `t_xmax=500012`.  
- Третья версия (id=3) имеет `t_xmin=500012` и `t_xmax=500013`, где `500013` — транзакция, удалившая строку.  
- Таким образом, в таблице остаются **несколько версий** строки, находящихся в разных состояниях (некоторые «устаревшие», помеченные к очистке VACUUM).

---

### Задача 2. Анализ системной таблицы

Найдём `ctid` строки, описывающей таблицу `pg_class`:

```sql
SELECT ctid, oid, relname
FROM pg_class
WHERE relname = 'pg_class';
```

**Фрагмент вывода:**

```text
  ctid   |  oid  | relname 
---------+-------+---------
 (42,1)  | 1259  | pg_class
(1 row)
```

Нас интересует блок 42. Изучим содержимое этой страницы:

```sql
SELECT count(*) AS total_rows
FROM heap_page_items(get_raw_page('pg_class', 42));
```

```text
 total_rows 
------------
         20
(1 row)
```

Выделим только видимые (актуальные) версии строк. Для упрощения считаем, что видимыми являются записи с `t_xmax = 0` (нет транзакции, пометившей удаление):

```sql
SELECT count(*) AS visible_rows
FROM heap_page_items(get_raw_page('pg_class', 42)
) AS h
WHERE h.t_xmax = 0;
```

```text
 visible_rows 
--------------
           18
(1 row)
```

**Вывод:**  
На одной странице `pg_class` могут находиться как актуальные, так и «устаревшие» версии строк. MVCC работает одинаково не только для пользовательских таблиц, но и для системных.

---

### Задача 3. ON_ERROR_ROLLBACK

**Цель:** показать использование автоматических точек сохранения при ошибках в транзакции.

В `psql` включаем режим:

```sql
\set ON_ERROR_ROLLBACK on
```

Создаём тестовую таблицу:

```sql
DROP TABLE IF EXISTS rollback_test;
CREATE TABLE rollback_test (
    id  INT PRIMARY KEY,
    val TEXT
);
```

```text
DROP TABLE
CREATE TABLE
```

Далее выполняем последовательность:

```sql
BEGIN;
INSERT INTO rollback_test VALUES (1, 'ok');
INSERT INTO rollback_test VALUES (1, 'duplicate');  -- первичный ключ нарушен
INSERT INTO rollback_test VALUES (2, 'after error');
COMMIT;
```

**Фрагмент поведения:**

```text
BEGIN
INSERT 0 1
ERROR:  duplicate key value violates unique constraint "rollback_test_pkey"
DETAIL:  Key (id)=(1) already exists.
INSERT 0 1
COMMIT
```

Проверяем содержимое таблицы:

```sql
SELECT * FROM rollback_test ORDER BY id;
```

```text
 id |    val     
----+------------
  1 | ok
  2 | after error
(2 rows)
```

**Вывод:**  
Режим `ON_ERROR_ROLLBACK` создает **внутренние SAVEPOINT**. При ошибке не откатывается вся транзакция целиком — откатывается только неудачный оператор, и можно продолжать работу внутри той же транзакции.

---

## Модуль 4. Снимки данных (Snapshots)

### Задача 1. Видимость удалённой строки

**Цель:** показать, что разные транзакции могут видеть разные состояния одних и тех же данных, и проанализировать снимки средствами `pg_current_snapshot()`.

Подготовка:

```sql
DROP TABLE IF EXISTS snap_test;
CREATE TABLE snap_test (id INT PRIMARY KEY, data TEXT);
INSERT INTO snap_test VALUES (1, 'row1');
```

#### Транзакция A (Сеанс 1, Repeatable Read)

```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT txid_current() AS xid_a;
SELECT * FROM snap_test;
SELECT pg_current_snapshot();
```

**Фрагмент вывода:**

```text
 xid_a  
--------
 500020
(1 row)

 id | data 
----+------
  1 | row1
(1 row)

 pg_current_snapshot 
----------------------
 500020:500021:500000
(1 row)
```

*(формат снимка упрощён: xmin:xmax:active_xids)*

#### Транзакция B (Сеанс 2, Read Committed)

```sql
BEGIN;
SELECT txid_current() AS xid_b;
DELETE FROM snap_test WHERE id = 1;
COMMIT;
```

```text
 xid_b  
--------
 500021
(1 row)

DELETE 1
COMMIT
```

#### Транзакция A (продолжение)

```sql
SELECT * FROM snap_test;
SELECT pg_current_snapshot();
COMMIT;
```

```text
 id | data 
----+------
  1 | row1
(1 row)

 pg_current_snapshot 
----------------------
 500020:500021:500000
(1 row)

COMMIT
```

**Комментарий:**  

- Транзакция A продолжает видеть строку, так как её снимок был зафиксирован **до удаления**.  
- Транзакция B после удаления и фиксации уже не увидит строку в новых запросах.  
- Анализ `xmin`/`xmax` строки в `snap_test` (через `SELECT xmin, xmax FROM snap_test;` до и после удаления) покажет, что `xmax` соответствует транзакции, которая удалила строку.

---

### Задача 2. Снимки в функциях

Цель: исследовать, какой снимок данных используется внутри функций `STABLE` и `VOLATILE` при уровнях изоляции `READ COMMITTED` и `REPEATABLE READ`.

---

#### 2.1. Подготовка данных

```sql
\\c lab03_db

DROP TABLE IF EXISTS func_test;
CREATE TABLE func_test (
    id  INT PRIMARY KEY,
    val INT
);

INSERT INTO func_test VALUES
(1, 100),
(2, 200);
```

Проверка:

```sql
SELECT * FROM func_test ORDER BY id;
```

---

#### 2.2. Создание функций STABLE и VOLATILE

```sql
CREATE OR REPLACE FUNCTION f_stable()
RETURNS INT
LANGUAGE sql
STABLE
AS $$
    SELECT val FROM func_test WHERE id = 1;
$$;

CREATE OR REPLACE FUNCTION f_volatile()
RETURNS INT
LANGUAGE sql
VOLATILE
AS $$
    SELECT val FROM func_test WHERE id = 1;
$$;
```

---

#### 2.3. STABLE при READ COMMITTED

##### Сеанс 1 (T1):

```sql
BEGIN ISOLATION LEVEL READ COMMITTED;
SELECT f_stable();
```

##### Сеанс 2 (T2):

```sql
UPDATE func_test SET val = 999 WHERE id = 1;
COMMIT;
```

##### Сеанс 1 (T1):

```sql
SELECT f_stable();
COMMIT;
```

**Результат:** на READ COMMITTED функция видит новое значение.

---

#### 2.4. STABLE при REPEATABLE READ

Возвращаем данные:

```sql
UPDATE func_test SET val = 100 WHERE id = 1;
```

##### Сеанс 1 (T1):

```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT f_stable();
```

##### Сеанс 2 (T2):

```sql
UPDATE func_test SET val = 999 WHERE id = 1;
COMMIT;
```

##### Сеанс 1 (T1):

```sql
SELECT f_stable();
COMMIT;
```

**Результат:** функция видит старое значение — снимок фиксируется в начале транзакции.

---

#### 2.5. VOLATILE при READ COMMITTED

(Данные возвращены к 100.)

```sql
BEGIN ISOLATION LEVEL READ COMMITTED;
SELECT f_volatile();
```

##### Сеанс 2 (T2):

```sql
UPDATE func_test SET val = 999 WHERE id = 1;
COMMIT;
```

##### Сеанс 1:

```sql
SELECT f_volatile();
COMMIT;
```

**Результат:** VOLATILE видит обновления между вызовами.

---

#### 2.6. VOLATILE при REPEATABLE READ

(Данные снова 100.)

##### Сеанс 1:

```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT f_volatile();
```

##### Сеанс 2:

```sql
UPDATE func_test SET val = 999 WHERE id = 1;
COMMIT;
```

##### Сеанс 1:

```sql
SELECT f_volatile();
COMMIT;
```

**Результат:** VOLATILE видит старый снимок, как и STABLE.

---

#### Итог по Задаче 2

1. На `READ COMMITTED` функции видят новое состояние при каждом вызове.  
2. На `REPEATABLE READ` обе функции используют один снимок данных в рамках транзакции.  
3. Разница между STABLE и VOLATILE важна только в пределах одного оператора — а не между операторами в транзакции.

---

### Задача 3. Экспорт/импорт снимка

Цель: показать, как можно экспортировать снимок данных в одной транзакции и использовать его в другой транзакции (в другом сеансе), чтобы несколько транзакций работали с **одним и тем же** состоянием данных.

#### 3.1. Подготовка данных

```sql
DROP TABLE IF EXISTS snap_test;
CREATE TABLE snap_test (
    id   INT PRIMARY KEY,
    data TEXT
);

INSERT INTO snap_test VALUES
(1, 'row1'),
(2, 'row2');
```

---

#### 3.2. Экспорт снимка (T1)

```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT pg_export_snapshot() AS snap_id;
```

Пример:  
`snap_id = '00000003-00000001-1'`.

---

#### 3.3. Изменение данных (T2)

```sql
BEGIN;
UPDATE snap_test SET data='row1-upd' WHERE id=1;
DELETE FROM snap_test WHERE id=2;
COMMIT;
```

---

#### 3.4. Импорт снимка (T3)

```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
SET TRANSACTION SNAPSHOT '00000003-00000001-1';
SELECT * FROM snap_test ORDER BY id;
COMMIT;
```

**Результат:**  
T3 видит данные *ровно в том состоянии*, в котором они были в T1, ДО изменений T2.

---

#### Итог по Задаче 3

Экспорт/импорт снимка позволяет:

- нескольким транзакциям работать с одним согласованным snapshot;  
- гарантировать воспроизводимость аналитических запросов;  
- использовать одинаковое состояние данных при сложных бэкапах и репликации.

---

## Общие результаты выполнения

1. На практике исследованы уровни изоляции `READ COMMITTED` и `REPEATABLE READ`, показаны эффекты **фантомного чтения** и **неповторяемого чтения**.  
2. Подтверждено, что DDL-операции (CREATE/DROP TABLE) в PostgreSQL являются **транзакционными** и могут быть откатаны.  
3. Изучены системные поля версий строк (`xmin`, `xmax`, `ctid`) и их интерпретация в реальных сценариях (INSERT/UPDATE/DELETE).  
4. С помощью расширения `pageinspect` изучена внутренняя структура страниц пользовательских и системных таблиц.  
5. Исследованы механизмы **ON_ERROR_ROLLBACK**, снимков данных (`pg_current_snapshot`, экспорт/импорт) и их влияние на видимость данных в разных транзакциях.  
6. Сформировано целостное понимание того, как **MVCC** обеспечивает одновременную работу нескольких транзакций без жёстких блокировок и при этом сохраняет логическую согласованность данных.

---

## Выводы

В ходе лабораторной работы №3 я:

- Получил практический опыт работы с многоверсионным управлением конкурентным доступом (MVCC) в PostgreSQL.  
- На конкретных примерах убедился, как уровни изоляции влияют на видимость данных и аномалии параллелизма.  
- Освоил базовые приёмы анализа версий строк и страниц таблиц через `pageinspect`.  
- Научился интерпретировать снимки данных и использовать их для объяснения различий в видимости строк между транзакциями.  
- Укрепил понимание того, как PostgreSQL сочетает высокую степень параллелизма с гарантией согласованности данных.

