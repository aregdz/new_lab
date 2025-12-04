Лабораторная работа №07: Управление доступом, расширениями и локализацией
Цель работы: Освоить управление правами доступа пользователей, работу с расширениями
PostgreSQL и настройку параметров локализации. Получить практические навыки настройки
аутентификации, управления привилегиями, установки расширений и миграции данных между разными
кодировками.
**Дата:** 2025-11-30
**Семестр:** 4 курс 1 полугодие – 7 семестр  
**Группа:** ПИЖ-б-о-22-1  
**Дисциплина:** Администрирование баз данных  
**Студент:** Джараян Арег Александрович
### Теоретическая часть (краткое содержание):
**Управление доступом: Система ролей и привилегий в PostgreSQL. Настройка аутентификации
через файл pg_hba.conf.**
**Расширения: Способ упаковки и установки дополнительного функционала (типы данных,
функции, операторы) в PostgreSQL.**
**Локализация: Настройки кодировки, правил сортировки (collation) и формата даты/времени.**
## Практическая работа


# 1. Модуль 1. Управление доступом

## 1.1. Создание ролей и базовой структуры прав

Подключаемся к кластеру:

```sql
\c postgres
```

Создаём новую БД:

```sql
CREATE DATABASE access_db;
```

Создаём две роли без входа:

```sql
CREATE ROLE writer NOINHERIT;
CREATE ROLE reader NOINHERIT;
```

Проверяем роли:

```sql
SELECT rolname, rolinherit, rolcanlogin
FROM pg_roles
WHERE rolname IN ('writer','reader');
```

### 1.1.2. Настройка привилегий схемы public

По умолчанию у роли PUBLIC слишком много прав. Отключим:

```sql
\c access_db

REVOKE ALL ON SCHEMA public FROM PUBLIC;
```

Теперь выдаём нужные права:

```sql
GRANT USAGE, CREATE ON SCHEMA public TO writer;
GRANT USAGE ON SCHEMA public TO reader;
```

Проверяем:

```sql
\dn+ public
```

Ожидаемый вывод (фрагмент):

```
 writer=UC/postgres
 reader=U/postgres
```

### 1.1.3. Привилегии по умолчанию

Чтоб reader автоматически получал доступ к новым таблицам writer:

```sql
ALTER DEFAULT PRIVILEGES FOR ROLE writer IN SCHEMA public
GRANT SELECT ON TABLES TO reader;
```

### 1.1.4. Создание логин-пользователей

```sql
CREATE ROLE w1 LOGIN PASSWORD 'w1pass';
CREATE ROLE r1 LOGIN PASSWORD 'r1pass';
```

Привязываем:

```sql
GRANT writer TO w1;
GRANT reader TO r1;
```

### 1.1.5. Проверка доступа

Под пользователем writer:

```sql
\c access_db w1

CREATE TABLE test_table (
    id int primary key,
    val text
);

INSERT INTO test_table VALUES (1, 'a'), (2, 'b');
```

Под пользователем reader:

```sql
\c access_db r1

SELECT * FROM test_table;
```

Ок. Пробуем изменить:

```sql
DELETE FROM test_table WHERE id=1;
```

Ожидаем:

```
ERROR: permission denied for table test_table
```

---

## 1.2. Аутентификация (peer, trust, md5)

### 1.2.1. Создаём роли alice и bob

```sql
CREATE ROLE alice LOGIN PASSWORD 'alicepass';
CREATE ROLE bob   LOGIN PASSWORD 'bobpass';
```

### 1.2.2. Настраиваем pg_hba.conf

Фрагмент:

```
# Разрешаем trust только postgres и student
local   all   postgres                 trust
local   all   student                  trust

# peer для alice и bob
local   all   alice                    peer map=os_users
local   all   bob                      peer map=os_users

# всем остальным md5
local   all   all                      md5
host    all   all   127.0.0.1/32        md5
host    all   all   ::1/128             md5
```

### 1.2.3. Настройка pg_ident.conf

```
os_users   alice   alice
os_users   bob     bob
```

Перегружаем конфигурацию:

```
sudo systemctl reload postgresql
```

### 1.2.4. Проверка peer

Если нет пользователя ОС alice:

```
FATAL: Peer authentication failed for user "alice"
```

Создаём пользователя ОС:

```
sudo adduser alice
```

Теперь:

```
su - alice
psql -d access_db
```

Успешно.

---

# 2. Модуль 2. Управление расширениями

Мы моделируем расширение `uom` (единицы измерения), если в системе нет оригинального.

## 2.1. Установка расширения

```sql
\c access_db

SELECT name FROM pg_available_extensions;

CREATE EXTENSION uom;
```

Проверяем:

```sql
\dx uom
```

## 2.2. Изучение структуры расширения

Таблицы, созданные расширением:

```sql
SELECT c.oid::regclass
FROM pg_class c
JOIN pg_depend d ON d.objid = c.oid
JOIN pg_extension e ON e.oid = d.refobjid
WHERE e.extname = 'uom'
  AND c.relkind='r';
```

Ожидаемый вывод:

```
uom.units
uom.categories
```

## 2.3. Добавляем новые единицы

```sql
INSERT INTO uom.units (code, name, base_code, ratio)
VALUES
 ('ft','Фут','m',0.3048),
 ('in','Дюйм','m',0.0254);
```

Проверяем:

```sql
SELECT * FROM uom.units WHERE code IN ('ft','in');
```

## 2.4. Права на объекты расширения

```sql
CREATE ROLE uom_reader;

REVOKE ALL ON SCHEMA uom FROM PUBLIC;
GRANT USAGE ON SCHEMA uom TO uom_reader;

REVOKE ALL ON ALL TABLES IN SCHEMA uom FROM PUBLIC;
GRANT SELECT ON ALL TABLES IN SCHEMA uom TO uom_reader;
```

Проверка:

```sql
\dp uom.units
```

## 2.5. Резервное копирование

```
pg_dump -n uom access_db > /tmp/uom.sql
```

Дамп содержит DDL + данные таблиц расширения.

---

# 3. Модуль 3. Локализация и кодировки

## 3.1. Перенос KOI8R → UTF8

### 3.1.1. Создание KOI8R-БД

```
createdb -E KOI8R --lc-collate=ru_RU.KOI8-R --lc-ctype=ru_RU.KOI8-R koi8_db
```

Проверяем:

```sql
\c koi8_db
SELECT datname, pg_encoding_to_char(encoding), datcollate, datctype
FROM pg_database WHERE datname='koi8_db';
```

### 3.1.2. Вставка кириллицы

```sql
CREATE TABLE msg_koi (id int, txt text);

INSERT INTO msg_koi VALUES
(1,'Привет'),
(2,'Ёлка'),
(3,'Проверка кодировки KOI8R');
```

### 3.1.3. Дамп

```
pg_dump -f /tmp/koi8.sql koi8_db
```

### 3.1.4. Создание UTF8-БД и импорт

```
createdb -E UTF8 --lc-collate=ru_RU.UTF-8 --lc-ctype=ru_RU.UTF-8 koi8_to_utf8
psql -d koi8_to_utf8 -f /tmp/koi8.sql
```

Проверяем:

```sql
SELECT * FROM msg_koi;
```

## 3.2. Локаль lc_time и EXTRACT(DOW)

```sql
SELECT CURRENT_DATE, EXTRACT(DOW FROM CURRENT_DATE);
```

Меняем локаль:

```sql
SET lc_time='ru_RU.utf8';
SELECT to_char(CURRENT_DATE,'Day');
```

Результат:

```
суббота
```

С английской локалью:

```
Saturday
```

EXTRACT(DOW) не меняется.

---

# Итоговые выводы

1. PostgreSQL позволяет гибко управлять ролями, привилегиями и аутентификацией, включая peer и trust только для отдельных пользователей.  
2. Расширения устанавливаются через `CREATE EXTENSION` и полностью управляются как обычные объекты БД (привилегии, дампы, структура).  
3. Миграция между кодировками KOI8R → UTF8 полностью возможна через `pg_dump` и `psql`, при корректном client_encoding.  
4. `lc_time` влияет только на текстовое представление дат, но не на числовой EXTRACT(DOW).  

