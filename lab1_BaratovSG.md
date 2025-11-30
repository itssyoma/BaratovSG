# Лабораторная работа №1  
## Архитектура СУБД PostgreSQL и конфигурация

**Семестр:** 7 семестр  
**Группа:** ПИЖ-б-о-22-1  
**Дисциплина:** Администрирование баз данных  
**Студент:** Баратов Семен Григорьевич

## Практическая часть

### Выполненные задачи
1. Определил расположение основного конфигурационного файла PostgreSQL.  
2. Изучил параметры конфигурации по уровням применения (`postmaster`, `sighup`, `user`).  
3. Проверил информацию о конфигурации в представлении `pg_file_settings`.  
4. Применил изменения параметров через `ALTER SYSTEM`.  
5. Создал дополнительный файл конфигурации через `include_dir`.  
6. Смоделировал ошибку в конфигурации.  
7. Изучил работу `SET` и `SET LOCAL`.  
8. Создал пользовательский параметр.

---

# Часть 1. Исследование параметров и файлов конфигурации

### Задача 1. Текущая конфигурация

*Подключитесь к серверу с помощью psql. Определите расположение основного файла конфигурации (postgresql.conf) с помощью команды SHOW config_file;*

```sql
SHOW config_file;
```

**Вывод:**

```
 /etc/postgresql/16/main/postgresql.conf
(1 row)
```

---

### Задача 2. Анализ параметров

*Изучите представление pg_settings. Найдите параметры, для изменения которых требуется перезагрузка сервера (context = 'postmaster'). Найдите 2-3 параметра с контекстом sighup и user.*

#### Параметры уровня postmaster

```sql
SELECT name, setting
FROM pg_settings
WHERE context = 'postmaster'
LIMIT 10;
```

**Вывод:**
```
 archive_mode | off
 autovacuum_freeze_max_age | 200000000
 autovacuum_max_workers | 3
 autovacuum_multixact_freeze_max_age | 400000000
 bonjour | off
 bonjour_name |
 cluster_name | 16/main
 config_file | /etc/postgresql/16/main/postgresql.conf
 data_directory | /var/lib/postgresql/16/main
 data_sync_retry | off
(10 rows)
```

---

#### Параметры уровня sighup

```sql
SELECT name, setting
FROM pg_settings
WHERE context = 'sighup'
LIMIT 10;
```

**Вывод:**
```
 archive_cleanup_command |
 archive_command | (disabled)
 archive_library |
 archive_timeout | 0
 authentication_timeout | 60
 autovacuum | on
 autovacuum_analyze_scale_factor | 0.1
 autovacuum_analyze_threshold | 50
 autovacuum_naptime | 60
 autovacuum_vacuum_cost_delay | 2
(10 rows)
```

---

#### Параметры уровня user

```sql
SELECT name, setting
FROM pg_settings
WHERE context = 'user'
LIMIT 10;
```

**Вывод:**
```
 application_name | psql
 array_nulls | on
 backend_flush_after | 0
 backslash_quote | safe_encoding
 bytea_output | hex
 check_function_bodies | on
 client_connection_check_interval | 0
 client_encoding | UTF8
 client_min_messages | notice
 commit_siblings | 5
(10 rows)
```

---

### Задача 3. Анализ файлов

*Изучите представление pg_file_settings. Определите, из каких файлов и с какими значениями были считаны текущие настройки параметров shared_buffers и work_mem.*

```sql
SELECT sourcefile, sourceline, name, setting, applied, error
FROM pg_file_settings
WHERE name IN ('shared_buffers', 'work_mem');
```

**Вывод:**
```
/etc/postgresql/16/main/postgresql.conf | 130 | shared_buffers | 128MB | t |
(1 row)
```

---

# Часть 2. Управление параметрами на уровне экземпляра

### Задача 1. Изменение параметра через ALTER SYSTEM

*Используя команду ALTER SYSTEM, установите для параметра work_mem новое значение. Убедитесь, что изменение записалось в файл postgresql.auto.conf (используйте функцию pg_read_file). Примените изменение, перечитав конфигурацию (SELECT pg_reload_conf();). Проверьте новое значение параметра и его источник в pg_settings.*

```sql
SHOW work_mem;
```
**Вывод до изменения:**
```
4MB
```

```sql
ALTER SYSTEM SET work_mem = '64MB';
SELECT pg_reload_conf();
SHOW work_mem;
```

**Вывод после изменения:**
```
64MB
```

Источник параметра:

```sql
SELECT name, setting, source, sourcefile
FROM pg_settings
WHERE name = 'work_mem';
```

**Фрагмент:**
```
work_mem | 65536 | configuration file | /var/lib/postgresql/16/main/postgresql.auto.conf
```

---

### Задача 2. Изменение через дополнительный файл

*Создайте файл в каталоге, указанном в директиве include_dir основного конфигурационного файла. Установите в этом файле значение для параметра log_min_duration_statement. Примените изменение и проверьте его.*

Файл:
```
/etc/postgresql/16/main/conf.d/lab01.conf
```

Содержимое:
```
log_min_duration_statement = 500
```

Проверка:

```sql
SHOW log_min_duration_statement;
```

**Вывод:**
```
500ms
```

---

### Задача 3. Ошибка в конфигурации

*Намеренно внесите синтаксическую ошибку в один из конфигурационных файлов (например, invalid_value вместо числового значения). Попытайтесь перечитать конфигурацию. Изучите представление pg_file_settings, чтобы найти запись об ошибке. Исправьте ошибку и перечитайте конфигурацию.*

```conf
work_mem = invalid_value
```

Проверка:

```sql
SELECT sourcefile, sourceline, name, setting, applied, error
FROM pg_file_settings
WHERE error IS NOT NULL;
```

**Вывод:**
```
/etc/postgresql/16/main/conf.d/lab01.conf | 2 | work_mem | invalid_value | f | invalid value for parameter "work_mem"
```

---

# Часть 3. Управление параметрами на уровне сеанса

### Задача 1. Команда SET

*В рамках сеанса измените значение параметра work_mem с помощью SET. Проверьте новое значение. Завершите транзакцию с помощью ROLLBACK и проверьте значение параметра again..*

```sql
BEGIN;
SET work_mem='128MB';
SHOW work_mem;
ROLLBACK;
SHOW work_mem;
```

---

### Задача 2. Команда SET LOCAL

*Откройте транзакцию (BEGIN). Inside the transaction, use SET LOCAL to change the work_mem parameter. Verify the change. After committing the transaction (COMMIT), check the parameter value again.*

```sql
BEGIN;
SET LOCAL work_mem='256MB';
SHOW work_mem;
COMMIT;
SHOW work_mem;
```

---

### Задача 3. Пользовательский параметр

Создайте и установите значение для пользовательского параметра (имя должно содержать точку, например, app.my_setting). Прочитайте его значение с помощью current_setting.

```sql
SET app.my_setting='lab01';
SELECT current_setting('app.my_setting', true);
```

**Вывод:**
```
lab01
```

---

# Результаты выполнения

- Изучены конфигурационные файлы PostgreSQL и уровни применения параметров.  
- Подтверждено поведение параметров `postmaster`, `sighup` и `user`.  
- Проверена работа механизма `ALTER SYSTEM` и включения дополнительных конфигов через `include_dir`.  
- Продемонстрирована регистрация ошибок в `pg_file_settings`.  
- Изучено поведение `SET`, `SET LOCAL` и пользовательских параметров.

---

# Выводы

Получены практические навыки управления конфигурацией PostgreSQL и диагностики параметров.
