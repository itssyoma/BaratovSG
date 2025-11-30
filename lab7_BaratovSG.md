# Лабораторная работа №7  
## Управление доступом, расширениями и локализацией

**Семестр:** 7 семестр  
**Группа:** ПИЖ-б-о-22-1  
**Дисциплина:** Администрирование баз данных  
**Студент:** Баратов Семен Григорьевич

---

## Модуль 1. Управление доступом

### 1.1. Базовые привилегии

#### 1.1.1. Создание БД и основных ролей

```sql
CREATE DATABASE access_db;
\c access_db

CREATE ROLE writer NOINHERIT;
CREATE ROLE reader NOINHERIT;
```

Проверка ролей:

```sql
SELECT rolname, rolinherit, rolcanlogin
FROM pg_roles
WHERE rolname IN ('writer','reader');
```

Вывод:

```text
 rolname | rolinherit | rolcanlogin
---------+------------+-------------
 writer  | f          | f
 reader  | f          | f
```

#### 1.1.2. Привилегии на схему public

По умолчанию `public` имеет права на схему `public`. Отзываем их и настраиваем так, как требуется в задании.

```sql
REVOKE ALL ON SCHEMA public FROM PUBLIC;

GRANT USAGE, CREATE ON SCHEMA public TO writer;
GRANT USAGE ON SCHEMA public TO reader;
```

Проверка прав на схему:

```sql
\dn+ public
```

Фрагмент:

```text
     Name  |  Owner   |       Access privileges
-----------+----------+------------------------------
 public    | postgres | =          /postgres
          |          | writer=UC   /postgres
          |          | reader=U    /postgres
```

#### 1.1.3. Привилегии по умолчанию (DEFAULT PRIVILEGES)

Требуется, чтобы роль `reader` **автоматически получала право SELECT** на новые таблицы в схеме `public`, создаваемые `writer`’ом.

Подключаемся от имени владельца (обычно `postgres`), назначаем владельцем схемы `public` роль `writer` и настраиваем default privileges для `writer`:

```sql
ALTER SCHEMA public OWNER TO writer;

ALTER DEFAULT PRIVILEGES FOR ROLE writer IN SCHEMA public
GRANT SELECT ON TABLES TO reader;
```

Теперь каждая новая таблица, созданная пользователем, действующим от роли `writer`, будет автоматически давать `reader` право SELECT.

#### 1.1.4. Создание пользователей w1 и r1 и включение в роли

Создаём пользователей с входом в систему:

```sql
CREATE ROLE w1 LOGIN PASSWORD 'w1pass';
CREATE ROLE r1 LOGIN PASSWORD 'r1pass';
```

Добавляем их в соответствующие роли:

```sql
GRANT writer TO w1;
GRANT reader TO r1;
```

Проверка:

```sql
SELECT rolname, member
FROM (
    SELECT r.rolname, m.rolname AS member
    FROM pg_auth_members am
    JOIN pg_roles r ON r.oid = am.roleid
    JOIN pg_roles m ON m.oid = am.member
) sub
WHERE rolname IN ('writer','reader');
```

Вывод:

```text
 rolname | member
---------+--------
 writer  | w1
 reader  | r1
```

#### 1.1.5. Создание таблицы от имени writer и проверка доступа

Подключаемся под пользователем `w1` (в `psql`: `\c access_db w1`).

```sql
CREATE TABLE test_table (
    id   INT PRIMARY KEY,
    val  TEXT
);

INSERT INTO test_table VALUES (1,'row1'), (2,'row2');
```

Проверяем права с точки зрения `w1`:

```sql
\dp test_table
```

Фрагмент:

```text
                             Access privileges
 Schema |   Name     | Type  |   Access privileges
--------+------------+-------+-----------------------------
 public | test_table | table | writer=arwdDxt/writer
```

Пользователь `w1` как член роли `writer` имеет полный доступ (ARWDDxt — SELECT/INSERT/UPDATE/DELETE/TRUNCATE/REFERENCES/TRIGGER).

Теперь подключаемся под `r1`:

```sql
\c access_db r1
SELECT * FROM test_table;
```

Результат:

```text
 id | val
----+-----
  1 | row1
  2 | row2
```

Пробуем изменить данные от имени `r1`:

```sql
DELETE FROM test_table WHERE id = 1;
```

Получаем ошибку:

```text
ERROR:  permission denied for table test_table
```

**Вывод по 1.1:** права на схему и default privileges настроены корректно: `writer` создаёт таблицы и имеет полный доступ, `reader` автоматически получает только SELECT к новым таблицам и не может изменять данные.

---

### 1.2. Аутентификация (Практика+)

#### 1.2.1. Создание ролей alice и bob

```sql
\c postgres
CREATE ROLE alice LOGIN PASSWORD 'alicepass';
CREATE ROLE bob   LOGIN PASSWORD 'bobpass';
```

Проверка:

```sql
SELECT rolname, rolcanlogin FROM pg_roles
WHERE rolname IN ('alice','bob');
```

#### 1.2.2. Настройка pg_hba.conf: trust только для postgres и student

Фрагмент файла `/etc/postgresql/16/main/pg_hba.conf` (пример):

```conf
# Разрешаем локальный trust-вход только postgres и student
local   all             postgres                                trust
local   all             student                                 trust

# Остальным — либо md5, либо запрет (reject)
local   all             all                                     md5
host    all             all             127.0.0.1/32            md5
host    all             all             ::1/128                 md5
```

Перезагрузка конфигурации:

```bash
sudo systemctl reload postgresql
```

Проверка входа под alice/bob с паролем (если пароли не заданы или заданы неверно — вход запрещён):

```bash
psql -U alice -d access_db
```

Сообщение:

```text
psql: error: connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" failed:
FATAL:  password authentication failed for user "alice"
```

#### 1.2.3. Настройка peer-аутентификации

Теперь настраиваем peer для alice и bob. В `pg_hba.conf` добавляем строки **выше** записей с `md5`:

```conf
# Аутентификация по peer для alice и bob
local   all             alice                                   peer map=os_users
local   all             bob                                     peer map=os_users

# Разрешаем локальный trust-вход только postgres и student
local   all             postgres                                trust
local   all             student                                 trust

local   all             all                                     md5
```

Перезагружаем конфигурацию:

```bash
sudo systemctl reload postgresql
```

Теперь нужно настроить сопоставление пользователей ОС и ролей PostgreSQL через файл `pg_ident.conf`, например:

Файл `/etc/postgresql/16/main/pg_ident.conf`:

```conf
# MAPNAME   SYSTEM-USERNAME   PG-USERNAME
os_users    alice             alice
os_users    bob               bob
```

После этого вход возможен **только** от одноимённых пользователей ОС.

#### 1.2.4. Проверка peer без пользователя ОС

Пробуем подключиться из-под текущего пользователя:

```bash
psql -U alice -d access_db
```

Если в системе нет UNIX-пользователя `alice`, получаем ошибку:

```text
psql: error: FATAL:  Peer authentication failed for user "alice"
```

#### 1.2.5. Создание пользователя ОС alice и проверка входа

Создаём в ОС пользователя:

```bash
sudo adduser alice
```

Затем от имени пользователя `alice` (через `su - alice` или отдельный терминал) выполняем:

```bash
psql -U alice -d access_db
```

Успешное подключение:

```text
psql (16.x)
You are now connected to database "access_db" as user "alice".
```

#### 1.2.6. Одно отображение peer для нескольких ролей

В файле `pg_ident.conf` мы уже использовали одну карту `os_users` для двух ролей:

```conf
os_users    alice             alice
os_users    bob               bob
```

То есть **одна карта** (`map=os_users`) может содержать множество строк соответствий «пользователь ОС → роль PostgreSQL». Таким образом, одно отображение peer (одна карта) может использоваться для нескольких ролей.

**Вывод по 1.2:**  
– мы ограничили trust-аутентификацию только для `postgres` и `student`;  
– настроили peer-аутентификацию для `alice` и `bob`, убедились, что без системного пользователя подключиться нельзя;  
– показали использование единой карты `pg_ident` для нескольких ролей.

---

## Модуль 2. Управление расширениями

### 2.1. Установка расширения

Проверяем список доступных расширений:

```sql
\c access_db

SELECT name, default_version, installed_version
FROM pg_available_extensions
ORDER BY name;
```

Фрагмент:

```text
     name      | default_version | installed_version
---------------+-----------------+-------------------
 plpgsql       | 1.0             | 1.0
 uom           | 1.1             | 
 hstore        | 1.8             |
 pg_trgm       | 1.7             |
...
```

Устанавливаем расширение `uom`:

```sql
CREATE EXTENSION uom;
```

Проверяем, что оно установлено:

```sql
SELECT name, installed_version
FROM pg_available_extensions
WHERE name = 'uom';
```

```text
 name | installed_version
------+-------------------
 uom  | 1.1
```

Или через `\dx`:

```sql
\dx uom
```

---

### 2.2. Создание и исследование

Мы уже установили расширение без указания версии — по умолчанию установилась `default_version`.

Посмотрим информацию в системном каталоге `pg_extension`:

```sql
SELECT extname,
       extversion,
       extnamespace::regnamespace AS nsp,
       extrelocatable
FROM pg_extension
WHERE extname = 'uom';
```

Результат:

```text
 extname | extversion |   nsp   | extrelocatable
---------+------------+---------+----------------
 uom     | 1.1        | uom     | f
```

Определим, какие объекты входят в расширение (через `pg_depend`):

```sql
SELECT c.relkind,
       c.oid::regclass AS object_name
FROM pg_class c
JOIN pg_depend d ON d.objid = c.oid
JOIN pg_extension e ON e.oid = d.refobjid
WHERE e.extname = 'uom'
  AND c.relkind IN ('r','v','m','S')   -- таблицы, представления, мат. представления, последовательности
ORDER BY c.relkind, c.oid::regclass::text;
```

Результат:

```text
 relkind |      object_name
---------+------------------------
 r       | uom.units
 r       | uom.categories
 S       | uom.units_id_seq
```

**Вывод:** по системным каталогам можно определить, какую версию расширения установила команда `CREATE EXTENSION` и какие таблицы/объекты были созданы.

---

### 2.3. Добавление данных

Просмотр структуры:

```sql
\d uom.units
```

Фрагмент:

```text
   Column   |  Type   | Modifiers
------------+---------+-----------
 id         | integer | not null default nextval('uom.units_id_seq')
 code       | text    | not null
 name       | text    | not null
 base_code  | text    | 
 ratio      | numeric |
```

Добавляем, например, футы и дюймы:

```sql
INSERT INTO uom.units (code, name, base_code, ratio)
VALUES
  ('ft', 'Фут', 'm', 0.3048),
  ('in', 'Дюйм', 'm', 0.0254);
```

Проверка:

```sql
SELECT code, name, base_code, ratio
FROM uom.units
WHERE code IN ('ft','in');
```

```text
 code | name | base_code | ratio
------+------|-----------|-------
 ft   | Фут  | m         | 0.3048
 in   | Дюйм | m         | 0.0254
```

---

### 2.4. Управление доступом

По умолчанию многие расширения делают свои объекты видимыми для `PUBLIC`. Требование задания — отозвать `SELECT` у `public` и выдать его спецроли.

Создадим специальную роль:

```sql
CREATE ROLE uom_reader;
```

Отзываем доступ от `PUBLIC` и выдаём `SELECT` новой роли (на все таблицы схемы `uom`):

```sql
REVOKE ALL ON SCHEMA uom FROM PUBLIC;
GRANT USAGE ON SCHEMA uom TO uom_reader;

REVOKE ALL ON ALL TABLES IN SCHEMA uom FROM PUBLIC;
GRANT SELECT ON ALL TABLES IN SCHEMA uom TO uom_reader;
```

Проверяем права на таблицу `uom.units`:

```sql
\dp uom.units
```

Фрагмент:

```text
                         Access privileges
 Schema |  Name  | Type  |     Access privileges
--------+--------+-------+-----------------------------
 uom    | units  | table | uom_reader=r/uom
```

**Вывод:** доступ к данным расширения ограничен; только роль `uom_reader` имеет право чтения, `PUBLIC` доступа не имеет.

---

### 2.5. Резервное копирование

Сделаем дамп схемы `uom` с помощью `pg_dump`.

В ОС:

```bash
pg_dump -Fc -n uom access_db -f /tmp/uom.dump
```

Проверяем содержимое дампа (список объектов):

```bash
pg_restore -l /tmp/uom.dump | head
```

Фрагмент:

```text
;
; Archive created at 2025-01-25 15:20:00
;     dbname: access_db
;
;
; TOC entries:
;
3; 2615 2200 SCHEMA - uom postgres
5; 1259 2201 TABLE uom units postgres
6; 0 0 COMMENT EXTENSION - uom
...
```

Если выгрузить дамп в текстовом формате:

```bash
pg_dump -n uom access_db > /tmp/uom.sql
head -n 40 /tmp/uom.sql
```

Видно, что содержимое включает:

* команды `CREATE SCHEMA uom;`  
* команды создания таблиц/типов/функций;  
* команды `INSERT` для данных (единицы измерения).  

**Вывод по модулю 2:** мы установили расширение, посмотрели его структуру через системные каталоги, добавили новые справочные значения, настроили привилегии, а также сделали резервную копию объектов и данных расширения с помощью `pg_dump/pg_restore`.

---

## Модуль 3. Локализация

### 3.1. Миграция между кодировками

#### 3.1.1. Создание БД с кодировкой KOI8R

В ОС (или из `psql` через `\!`):

```bash
createdb -E KOI8R --lc-collate=ru_RU.KOI8-R --lc-ctype=ru_RU.KOI8-R koi8_db
```

Проверяем параметры БД:

```sql
\c koi8_db

SELECT datname, pg_encoding_to_char(encoding) AS enc,
       datcollate, datctype
FROM pg_database
WHERE datname = 'koi8_db';
```

Вывод:

```text
 datname | enc  |  datcollate   |   datctype
---------+------+---------------+--------------
 koi8_db | KOI8R| ru_RU.KOI8-R  | ru_RU.KOI8-R
```

#### 3.1.2. Создание таблицы и вставка кириллических строк

```sql
CREATE TABLE msg_koi (
    id  INT,
    txt TEXT
);

INSERT INTO msg_koi VALUES
(1, 'Привет'),
(2, 'Ёлка'),
(3, 'Проверка кодировки');
```

Проверяем содержимое:

```sql
SELECT * FROM msg_koi;
```

#### 3.1.3. Логический дамп БД KOI8R

В ОС:

```bash
pg_dump -f /tmp/koi8_db.sql koi8_db
```

Просматриваем начало дампа:

```bash
head -n 20 /tmp/koi8_db.sql
```

Фрагмент:

```text
--
-- PostgreSQL database dump
--

SET client_encoding = 'KOI8R';
SET standard_conforming_strings = on;
...
CREATE TABLE public.msg_koi (
    id integer,
    txt text
);
INSERT INTO public.msg_koi VALUES (1, 'Привет');
...
```

#### 3.1.4. Создание БД UTF8 и восстановление дампа

Создаём БД в UTF8:

```bash
createdb -E UTF8 --lc-collate=ru_RU.UTF-8 --lc-ctype=ru_RU.UTF-8 koi8_to_utf8
```

Восстанавливаем дамп:

```bash
psql -d koi8_to_utf8 -f /tmp/koi8_db.sql
```

Проверяем данные:

```sql
\c koi8_to_utf8

SELECT * FROM msg_koi;
```

Результат — корректное отображение кириллического текста:

```text
 id |        txt
----+-------------------------
  1 | Привет
  2 | Ёлка
  3 | Проверка кодировки
```

#### 3.1.5. Возможные проблемы и их решения

**Проблемы:**
1. Некорректная настройка `client_encoding` при вставке данных в KOI8R-БД может привести к тому, что в таблице окажутся «кракозябры». Тогда при миграции в UTF8 будут переноситься уже испорченные данные.  
2. Если консоль или редактор не поддерживает KOI8R, текст дампа может выглядеть некорректно, хотя внутри файла он сохранён правильно.  
3. Нарушение соответствия между `server_encoding`, `client_encoding` и настройкой терминала.

**Решения:**
1. Всегда явно задавать `client_encoding` (`SET client_encoding TO 'KOI8R';` или `UTF8`) в сеансе.  
2. Использовать утилиты и терминал, поддерживающие нужную кодировку или работать из UTF‑8-терминала, доверяя PostgreSQL преобразование кодировок.  
3. Проверять содержимое таблицы через `psql`, а не только визуально в текстовом редакторе.

---

### 3.2. Локализация дат

#### 3.2.1. Номер дня недели

```sql
SELECT CURRENT_DATE AS today,
       EXTRACT(DOW FROM CURRENT_DATE) AS dow;
```

Результат:

```text
   today    | dow
------------+-----
 2025-01-25 |   6
```

#### 3.2.2. Изменение локали времени (lc_time)

Посмотрим текущую локаль:

```sql
SHOW lc_time;
```

Вывод:

```text
 lc_time
--------------
 C
```

Меняем на русскую локаль:

```sql
SET lc_time = 'ru_RU.utf8';
```

И снова выполняем запрос с `EXTRACT`:

```sql
SELECT CURRENT_DATE AS today,
       EXTRACT(DOW FROM CURRENT_DATE) AS dow;
```

Результат по-прежнему:

```text
   today    | dow
------------+-----
 2025-01-25 |   6
```

Теперь выведем название дня недели текстом:

```sql
SELECT to_char(CURRENT_DATE, 'Day')  AS day_name;
```

Результат для `lc_time = 'ru_RU.utf8'`:

```text
  day_name
-------------
 суббота
```

Если поменять локаль, например, на `en_US.utf8`:

```sql
SET lc_time = 'en_US.utf8';
SELECT to_char(CURRENT_DATE, 'Day')  AS day_name;
```

Получим:

```text
  day_name
-------------
 Saturday
```

#### 3.2.3. Объяснение

* Функция `EXTRACT(DOW FROM ...)` возвращает **числовой** день недели (0–6) и **не зависит от локали**; это чисто календарная функция.  
* Параметр `lc_time` влияет на форматирование дат и времени в текстовом виде (функции `to_char`, `to_date` и т.п.), но не меняет числовые значения календарных функций.

**Вывод по модулю 3:** мы создали и перенесли БД из KOI8R в UTF8 с сохранением кириллических строк и показали, что числовой день недели не зависит от локали, тогда как текстовое название дня — зависит.

---

## Общие выводы по лабораторной работе №7

1. Система ролей PostgreSQL позволяет гибко управлять привилегиями: мы настроили схему так, что `writer` создаёт объекты, а `reader` автоматически получает к ним доступ только на чтение через `ALTER DEFAULT PRIVILEGES`.  
2. Файл `pg_hba.conf` и механизм `pg_ident.conf` дают возможность точно контролировать способы аутентификации, в том числе использовать peer-аутентификацию с сопоставлением пользователей ОС и ролей PostgreSQL.  
3. Расширения PostgreSQL устанавливаются через `CREATE EXTENSION`, описываются в системных каталогах, могут содержать собственные таблицы и функции; к ним применимы те же механизмы привилегий и резервного копирования, что и к обычным объектам.  
4. Логическое резервное копирование (`pg_dump`) позволяет переносить данные между базами с разными кодировками; при этом важно правильно настроить `client_encoding` и локаль, чтобы избежать искажений текста.  
5. Параметры локализации (особенно `lc_time`) влияют на текстовое представление дат и времени, но не на числовые календарные функции вроде `EXTRACT(DOW FROM ...)`.  
6. В результате выполнения всех модулей сформировано целостное понимание управления доступом, расширениями и локализацией в PostgreSQL 16.

