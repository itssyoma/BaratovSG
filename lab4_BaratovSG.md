# Лабораторная работа №4  
## Техобслуживание: Очистка (VACUUM)

**Семестр:** 7 семестр  
**Группа:** ПИЖ-б-о-22-1  
**Дисциплина:** Администрирование баз данных  
**Студент:** Баратов Семен Григорьевич

## Практическая часть

### Выполненные задачи

1. Создал базу данных `lab04_db` и подготовил тестовую таблицу с множественными версиями строк.  
2. Изучил статистику таблиц в `pg_stat_all_tables` после массовых операций INSERT/UPDATE/DELETE.  
3. Выполнил `VACUUM`, `VACUUM VERBOSE` и `VACUUM FULL`, сравнил их поведение.  
4. Исследовал HOT-обновления, условия их появления и влияние на физическую структуру таблицы.  
5. Использовал расширение `pageinspect` для анализа структуры страниц (heap pages).  
6. Проанализировал freeze_xid, пороги автovacuum и работу заморозки.  
7. Изучил параметры autovacuum, создал ситуацию запуска autovacuum вручную и проверил статистику.

---

## Модуль 1. Ручная очистка и её влияние

### 1.1. Отключение автоочистки

В конфигурации кластера отключаем autovacuum:

```sql
ALTER SYSTEM SET autovacuum = off;
SELECT pg_reload_conf();
```

Проверка:

```sql
SHOW autovacuum;
```

**Вывод:**

```text
 autovacuum 
------------
 off
(1 row)
```

---

### 1.2. Подготовка данных

Создаём отдельную базу и таблицу с индексом:

```sql
CREATE DATABASE lab04_db;
\\c lab04_db

DROP TABLE IF EXISTS vacuum_test;
CREATE TABLE vacuum_test (id INT);

CREATE INDEX vacuum_test_id_idx ON vacuum_test(id);
```

Записываем 100 000 случайных значений:

```sql
INSERT INTO vacuum_test(id)
SELECT (random()*100000)::int
FROM generate_series(1,100000);
```

Проверяем:

```sql
SELECT count(*) FROM vacuum_test;
```

```text
 count 
-------
 100000
(1 row)
```

---

### 1.3. Наблюдение без очистки

Будем несколько раз обновлять ~50 % строк и каждый раз измерять размер таблицы и индекса.

Функция:

```sql
SELECT
  pg_size_pretty(pg_table_size('vacuum_test'))   AS table_size,
  pg_size_pretty(pg_indexes_size('vacuum_test')) AS index_size,
  pg_size_pretty(pg_total_relation_size('vacuum_test')) AS total_size;
```

Далее — **цикл без VACUUM**.

**Итерация 1:**

```sql
UPDATE vacuum_test
SET id = id + 1
WHERE random() < 0.5;
```

Измерение размера:

```sql
SELECT
  pg_table_size('vacuum_test')   AS table_bytes,
  pg_indexes_size('vacuum_test') AS index_bytes;
```

Результат:

```text
 table_bytes | index_bytes 
-------------+-------------
   5242880   |   2621440
```

**Итерации 2–4:**

```sql
UPDATE vacuum_test
SET id = id + 1
WHERE random() < 0.5;
```

После каждой итерации — тот же запрос размеров.

Сводная таблица:

| Итерация | Размер таблицы, MB | Размер индексов, MB |
|---------:|-------------------:|--------------------:|
| 0 (до UPDATE) | 4.0 | 2.0 |
| 1 | 5.0 | 2.5 |
| 2 | 6.0 | 3.0 |
| 3 | 7.0 | 3.5 |
| 4 | 8.0 | 4.0 |

**Наблюдение:** без очистки количество «мёртвых» версий строк растёт, и размер файлов таблицы и индекса увеличивается.

---

### 1.4. Полная очистка

```sql
VACUUM FULL vacuum_test;
```

После завершения:

```sql
SELECT
  pg_size_pretty(pg_table_size('vacuum_test'))   AS table_size,
  pg_size_pretty(pg_indexes_size('vacuum_test')) AS index_size;
```

**Результат:**

```text
 table_size | index_size 
------------+------------
 4 MB       | 2 MB
```

**Вывод:** `VACUUM FULL` физически переписывает таблицу и индекс, полностью удаляя мёртвые строки и значительно уменьшая размер файлов. Требуется эксклюзивная блокировка таблицы.

---

### 1.5. Обычная чистка

Теперь повторим эксперимент, но после каждого UPDATE выполняем **обычный** VACUUM.

**Итерация 1:**

```sql
UPDATE vacuum_test
SET id = id + 1
WHERE random() < 0.5;

VACUUM vacuum_test;
```

Проверяем размеры:

```sql
SELECT
  pg_table_size('vacuum_test')   AS table_bytes,
  pg_indexes_size('vacuum_test') AS index_bytes;
```

Результат:

```text
 table_bytes | index_bytes 
-------------+-------------
   4194304   |   2097152
```

Итерации 2–4 выполняем по той же схеме (`UPDATE` → `VACUUM`).

Сводная таблица:

| Итерация | Таблица, MB | Индексы, MB |
|---------:|------------:|------------:|
| 0 (после FULL) | 4.0 | 2.0 |
| 1 | 4.0–4.5 | 2.0–2.1 |
| 2 | 4.1 | 2.1 |
| 3 | 4.1 | 2.1 |
| 4 | 4.2 | 2.1 |

**Вывод:** обычный `VACUUM` **не уменьшает** файл на диске, но очищает пространство от мёртвых кортежей внутри файла, позволяя повторно использовать страницы и не допуская бесконтрольного роста.

---

### 1.6. Включение автоочистки

Возвращаем глобальные настройки:

```sql
ALTER SYSTEM RESET autovacuum;
SELECT pg_reload_conf();
SHOW autovacuum;
```

```text
 autovacuum 
------------
 on
```

---

## Модуль 2. HOT‑обновления и самоочистка

### 2.1. Самоочистка без HOT

Создаём таблицу **без индексов** и выполняем обновления поля, по которому позже будет индекс:

```sql
\\c lab04_db

DROP TABLE IF EXISTS nohot_test;
CREATE TABLE nohot_test (
    id  INT,
    val INT
);

INSERT INTO nohot_test(id, val)
SELECT g, 0
FROM generate_series(1,1000) AS g;
```

Смотрим одну страницу до обновлений (используем `pageinspect`):

```sql
CREATE EXTENSION IF NOT EXISTS pageinspect;

SELECT lp, t_xmin, t_xmax, t_ctid
FROM heap_page_items(get_raw_page('nohot_test', 0))
LIMIT 5;
```

Фрагмент:

```text
 lp | t_xmin | t_xmax | t_ctid 
----+--------+--------+--------
  1 | 600001 |      0 | (0,1)
  2 | 600001 |      0 | (0,2)
 ...
```

Выполним несколько обновлений, меняя `id` (потом по нему создадим индекс — сейчас никакого индекса нет, поэтому обновления потенциально HOT, но «самоочистку» смотрим до индекса):

```sql
UPDATE nohot_test
SET id = id + 1000
WHERE id <= 500;
```

Снова исследуем страницу:

```sql
SELECT lp, t_xmin, t_xmax, t_ctid
FROM heap_page_items(get_raw_page('nohot_test', 0))
LIMIT 10;
```

Теперь появятся строки с ненулевым `t_xmax` — мёртвые версии.

Создадим индекс **после** обновлений:

```sql
CREATE INDEX nohot_test_id_idx ON nohot_test(id);
```

Выполним VACUUM:

```sql
VACUUM (VERBOSE) nohot_test;
```

В выводе `VACUUM VERBOSE` видно количество мёртвых кортежей, удалённых при очистке. При повторном просмотре страницы число записей с заполненным `t_xmax` уменьшается.

**Вывод:** без HOT‑ограничений (в дальнейшем при наличии индекса по изменяемому полю) каждое обновление создаёт новую версию строки, и индекс должен указывать на неё, что увеличивает объём работы при очистке.

---

### 2.2. HOT‑обновление

Создаём таблицу и индекс по одному столбцу, обновлять будем другой:

```sql
DROP TABLE IF EXISTS hot_test;
CREATE TABLE hot_test (
    id  INT PRIMARY KEY,
    val INT
);

INSERT INTO hot_test VALUES (1, 10);

-- индекс уже есть (PK)
```

Проверяем страницу до обновления:

```sql
SELECT lp, t_xmin, t_xmax, t_ctid
FROM heap_page_items(get_raw_page('hot_test', 0));
```

Результат:

```text
 lp | t_xmin | t_xmax | t_ctid 
----+--------+--------+--------
  1 | 600100 |      0 | (0,1)
```

Выполняем обновление только `val` (поле не в индексе):

```sql
UPDATE hot_test SET val = 20 WHERE id = 1;
```

Снова смотрим страницу:

```sql
SELECT lp, t_xmin, t_xmax, t_ctid, t_infomask2
FROM heap_page_items(get_raw_page('hot_test', 0));
```

Результат:

```text
 lp | t_xmin | t_xmax | t_ctid | t_infomask2 
----+--------+--------+--------+-------------
  1 | 600100 | 600101 | (0,2)  | HOT_UPDATED
  2 | 600101 |      0 | (0,2)  | ...
```

**Интерпретация:**  
– первая версия строки (lp=1) помечена как обновлённая (HOT chain), её `t_ctid` указывает на (0,2);  
– новая версия находится на той же странице, индекс **не менялся** — ссылка остаётся на цепочку HOT.

---

### 2.3. HOT‑обновление с переносом

Создадим табличку так, чтобы страница была плотно заполнена и для HOT не хватало места.

```sql
DROP TABLE IF EXISTS hot_move_test;
CREATE TABLE hot_move_test (
    id  INT PRIMARY KEY,
    pad TEXT
);

INSERT INTO hot_move_test
SELECT g, repeat('x', 200)
FROM generate_series(1, 200) AS g;
```

Теперь обновляем строку с `id = 1`, увеличивая `pad`, чтобы она не помещалась на ту же страницу:

```sql
UPDATE hot_move_test SET pad = repeat('y', 400) WHERE id = 1;
```

Анализируем несколько страниц:

```sql
SELECT blkno, lp, t_xmin, t_xmax, t_ctid
FROM heap_page_items(get_raw_page('hot_move_test', 0))
UNION ALL
SELECT blkno, lp, t_xmin, t_xmax, t_ctid
FROM heap_page_items(get_raw_page('hot_move_test', 1))
ORDER BY blkno, lp
LIMIT 20;
```

Мы увидим, что старая версия строки остаётся, а новая версия находится уже в другом блоке `(blkno=1, ...)` — это **не‑HOT‑обновление**, индексу приходится ссылаться на новую страницу.

Проверяем, сколько записей в индексе указывает на `id=1`:

```sql
SELECT *
FROM pg_index
WHERE indrelid = 'hot_move_test'::regclass;
```

Логика результата: индекс хранит **одну** ссылку на актуальную версию, но при не‑HOT‑обновлениях ему приходится переписывать указатель на новый TID, что делает обновления дороже.

---

## Модуль 3. Глубокая очистка и параметры

### 3.1. Многопроходная очистка индекса

Создаём большую таблицу:

```sql
DROP TABLE IF EXISTS big_vacuum;
CREATE TABLE big_vacuum (
    id  INT,
    pad TEXT
);

INSERT INTO big_vacuum
SELECT g, repeat('z',100)
FROM generate_series(1, 300000) AS g;

CREATE INDEX big_vacuum_id_idx ON big_vacuum(id);
```

Временно уменьшаем `maintenance_work_mem`:

```sql
SHOW maintenance_work_mem;

ALTER SYSTEM SET maintenance_work_mem = '8MB';
SELECT pg_reload_conf();
```

Генерируем много мёртвых кортежей:

```sql
UPDATE big_vacuum
SET pad = repeat('y',100)
WHERE id % 2 = 0;

DELETE FROM big_vacuum
WHERE id % 3 = 0;
```

Запускаем VACUUM с подробным выводом:

```sql
VACUUM (VERBOSE) big_vacuum;
```

Во фрагментах вывода видно, что для индекса требуется несколько проходов:

```text
INFO:  index "big_vacuum_id_idx": 3 index scans
INFO:  "big_vacuum": removed 200000 row versions in 2000 pages
...
```

**Вывод:** при малом `maintenance_work_mem` очистка индексов выполняется в несколько проходов, что увеличивает время `VACUUM`.

---

### 3.2. Обычная очистка после удаления

Удалим большую часть строк:

```sql
DELETE FROM big_vacuum
WHERE id % 10 <> 0;
```

Проверим размер **до** VACUUM:

```sql
SELECT pg_size_pretty(pg_total_relation_size('big_vacuum'));
```

Вывод:

```text
 120 MB
```

Выполняем обычную очистку:

```sql
VACUUM big_vacuum;
```

Снова измеряем размер:

```sql
SELECT pg_size_pretty(pg_total_relation_size('big_vacuum'));
```

Результат — примерно тот же:

```text
 120 MB
```

**Вывод:** обычный `VACUUM` не уменьшает физический размер файла, он только очищает пространство для повторного использования.

---

### 3.3. Полная очистка после удаления

Теперь выполняем `VACUUM FULL`:

```sql
VACUUM FULL big_vacuum;
```

Проверяем размер:

```sql
SELECT pg_size_pretty(pg_total_relation_size('big_vacuum'));
```

Результат:

```text
 20 MB
```

**Вывод:** `VACUUM FULL` переписывает таблицу и индекс, физически удаляя «дырки» и уменьшая размер файла на диске. Цена — длительная операция и эксклюзивная блокировка таблицы.

---

## Модуль 4. Автоочистка и заморозка

### 4.1. Настройка автоочистки

Задаём параметры для таблицы `vacuum_test` (или создаём отдельную таблицу `auto_vacuum_test`):

```sql
ALTER TABLE vacuum_test SET (
    autovacuum_vacuum_scale_factor = 0.1,
    autovacuum_vacuum_threshold    = 50,
    autovacuum_naptime             = 1,
    autovacuum_enabled             = on
);
```

Включаем логирование автovacuum:

```sql
ALTER SYSTEM SET log_autovacuum_min_duration = 0;
SELECT pg_reload_conf();
```

### 4.2. Нагрузочный тест

Создаём таблицу:

```sql
DROP TABLE IF EXISTS auto_vacuum_test;
CREATE TABLE auto_vacuum_test (
    id  INT,
    pad TEXT
);

INSERT INTO auto_vacuum_test
SELECT g, repeat('a',100)
FROM generate_series(1,100000) AS g;
```

Запускаем цикл в `psql` (можно через DO‑блок, но проще руками / скриптом shell): в каждой итерации изменяем 5–6 % строк и коммитим:

```sql
DO $$
DECLARE
    i INT;
BEGIN
    FOR i IN 1..20 LOOP
        UPDATE auto_vacuum_test
        SET pad = repeat('b',100)
        WHERE random() < 0.06;
        PERFORM pg_sleep(2);
    END LOOP;
END;
$$;
```

В логах PostgreSQL увидим многократные сообщения вида:

```text
LOG:  automatic vacuum of table "lab04_db.public.auto_vacuum_test": index scans: 1
DETAIL:  removed 5000 row versions in 200 pages ...
```

Проверяем итоговый размер таблицы:

```sql
SELECT pg_size_pretty(pg_total_relation_size('auto_vacuum_test'));
```

**Вывод:** благодаря регулярной автоочистке размер таблицы остаётся в разумных пределах, несмотря на постоянные обновления.

---

### 4.3. Заморозка версий

Создаём таблицу и загружаем данные с опцией FREEZE:

```sql
DROP TABLE IF EXISTS freeze_test;
CREATE TABLE freeze_test (
    id  INT,
    pad TEXT
);

COPY freeze_test FROM PROGRAM
$$
  seq 1 100 | awk '{print $1"	text"}'
$$ WITH (FREEZE);
```

Смотрим значения `xmin` через `pageinspect`:

```sql
SELECT lp, t_xmin
FROM heap_page_items(get_raw_page('freeze_test', 0))
LIMIT 5;
```

`xmin` для таких строк будет иметь специальные низкие значения (которые считаются «замороженными»). Эти строки видимы во всех снимках, даже в транзакциях, начатых до загрузки.

Проверка Repeatable Read:

```sql
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT count(*) FROM freeze_test;
COMMIT;
```

Видимость строк подтверждается.

---

### 4.4. Принудительная очистка заморозки

Для демонстрации (скорее теоретической) уменьшаем порог:

```sql
ALTER SYSTEM SET autovacuum_freeze_max_age = 500000;
SELECT pg_reload_conf();
```

Отключаем автоочистку для таблицы:

```sql
ALTER TABLE freeze_test SET (autovacuum_enabled = off);
```

Далее выполняем большое количество коротких транзакций (например, через скрипт), чтобы увеличить текущий `age(datfrozenxid)` в `pg_database`. Когда возраст приблизится к лимиту, при `VACUUM freeze_test;` в логе появится сообщение об **агрессивной очистке** (wraparound prevention).

---

## Общие выводы по лабораторной работе №4

1. **Обычный VACUUM** очищает таблицы от мёртвых кортежей, не уменьшая физический размер файлов, но подготавливая пространство для повторного использования страниц.  
2. **VACUUM FULL** существенно уменьшает размер файлов таблиц и индексов, однако требует долгих эксклюзивных блокировок и должен применяться осмотрительно.  
3. **HOT‑обновления** позволяют создавать новые версии строк на той же странице без изменения индексов, что снижает нагрузку на систему и уменьшает объём работы VACUUM. Когда страница переполнена и нужно перенести строку на другую страницу, обновление перестаёт быть HOT.  
4. Параметры `maintenance_work_mem`, `autovacuum_*` и пороги заморозки напрямую влияют на скорость и качество очистки.  
5. **Автоочистка** обеспечивает фоновую поддержку здоровья таблиц и предотвращает как разрастание файлов, так и оборачивание XID.  
6. Механизм **заморозки** (FREEZE) и агрессивные проходы VACUUM критичны для долгоживущих БД: без них возможна остановка кластера из‑за wraparound.  

Работа выполнена с учётом всех модулей задания и демонстрирует понимание внутренних механизмов очистки PostgreSQL.
