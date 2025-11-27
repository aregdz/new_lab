# Отчет по лабораторной работе №2
# Организация данных и системный каталог

**Дата:** 2025-11-27
**Семестр:** 4 курс 1 полугодие – 7 семестр  
**Группа:** ПИЖ-б-о-22-1  
**Дисциплина:** Администрирование баз данных  
**Студент:** Джараян Арег Александрович

## Цель работы
Всестороннее изучение логической и физической структуры хранения данных в
PostgreSQL. Получение практических навыков управления базами данных, схемами, табличными
пространствами. Глубокое освоение работы с системным каталогом для извлечения метаинформации.
Исследование низкоуровневых аспектов хранения, включая TOAST.

## Теоретическая часть
**Логическая структура: Иерархия: Кластер БД -> Базы данных -> Схемы -> Объекты (таблицы,
представления и т.д.).
**Физическая структура: Табличные пространства связывают логические объекты с физическим
расположением файлов на диске. Данные хранятся в файлах.
**Системный каталог: Набор системных таблиц (pg_class, pg_database, pg_namespace) и
представлений (pg_tables, pg_views), содержащих метаданные.
**Низкоуровневое хранение: Механизм TOAST для больших данных, нежурналируемые таблицы,
стратегии хранения столбцов.

## Практическая часть

### Часть 1.Базы данных и схемы

**Выполненные задачи:**
1. Создание и проверка БД: Создал новую базу данных lab02_db. Проверил ее начальный
размер с помощью pg_database_size('lab02_db').
```sql
sudo -u postgres createdb lab02_db
sudo -u postgres psql -At -c "SELECT pg_database_size ('lab02_db');"
```
```text
7602703
```
2. Работа со схемами: Подключился к lab02_db. Создалдве схемы: app и схему с именем
моего пользователя ОС (напр., student). В каждой схеме создайл по одной таблице и вставьте в
них данные.
```sql
sudo -u postgres psql -d lab02_db
```
```text
psql (16.9 (Ubuntu 16.9-1.pgdg24.04+1))
Type "help" for help.
```
```sql
CREATE SCHEMA app;
CREATE SCHEMA student; 
```
```text
CREATE SCHEMA
CREATE SCHEMA
```
```sql
CREATE TABLE app.users (
    id serial PRIMARY KEY,
    name text
);
INSERT INTO app.users (name) VALUES ('Areg');

CREATE TABLE student.items (
    id serial PRIMARY KEY,
    value text
);
INSERT INTO student.items (value) VALUES ('Test item');
```
```text
CREATE TABLE
INSERT 0 1
```
3. Контроль размера: Снова проверил размер базы данных. Объяснил его изменение.
   ```sql
   SELECT pg_size_pretty(pg_database_size('lab02_db'));
```
```text
 pg_size_pretty 
----------------
 7724 kB
(1 row)
```
--Даже пустая таблица занимает место на диске: PostgreSQL создаёт файлы для данных, для системных метаданных и для будущих страниц хранения. После вставки строк размер дополнительно увеличивается за счёт самих данных и служебных структур. То есть рост размера базы — результат появления новых объектов (схемы, таблицы, строки) и системных структур, которые PostgreSQL создаёт автоматически.

4. Управление путем поиска: Настройте параметр search_path для текущего сеанса так, чтобы
при обращении по неполному имени приоритет имела ваша пользовательская схема, а затем
схема app. Продемонстрируйте работу, обратившись к таблицам без указания схемы.
-- Настройка search_path в текущем сеансе
  ```sql
SET search_path = student, app, public;
SHOW search_path;
```
```text
     search_path      
----------------------
 student, app, public
(1 row)
```
-- Демонстрация работы
```sql
SELECT * FROM items;   
SELECT * FROM users;
```
```text
 id |   value   
----+-----------
  1 | Test item
(1 row)

ab02_db=# SELECT * FROM users;
 id | name 
----+------
  1 | Areg
(1 row)
```

5. Практика+ (Настройка параметра БД): Для базы lab02_db установите значение параметра
temp_buffers так, чтобы в каждом новом сеансе, подключенном к этой БД, оно было в 4 раза
больше значения по умолчанию. Проверьте работу.
-- Узнал значение по умолчанию для temp_buffers
   ```sql
   sudo -u postgres psql -d lab02_db
   SHOW temp_buffers;
   ```
   ```text
   lab02_db=# SHOW temp_buffers;
     temp_buffers 
    --------------
     8MB
    (1 row)
   ```
   -- Установил параметр для базы так, чтобы он был в 4 раза больше и проверил работу
   ```sql
   ALTER DATABASE lab02_db SET temp_buffers = '32MB';
   \c lab02_db
    SHOW temp_buffers;
    ```
   ```text
   lab02_db=# SHOW temp_buffers;
     temp_buffers 
    --------------
     32MB
    (1 row)
    ```
## Модуль 2: Системный каталог
**1. Исследование pg_class: Получил описание системной таблицы pg_class (команда \d
pg_class).
```sql
\d pg_class
```
фрагмент вывода
```text
                    Table "pg_catalog.pg_class"
       Column        |     Type     | Collation | Nullable | Default 
---------------------+--------------+-----------+----------+---------
 oid                 | oid          |           | not null | 
 relname             | name         |           | not null | 
 relnamespace        | oid          |           | not null | 
 reltype             | oid          |           | not null | 
 reloftype           | oid          |           | not null | 
 relowner            | oid          |           | not null | 
 relam               | oid          |           | not null | 
 relfilenode         | oid          |           | not null | 
 reltablespace       | oid          |           | not null | 
 relpages            | integer      |           | not null | 
 reltuples           | real         |           | not null | 
 relallvisible       | integer      |           | not null | 
 reltoastrelid       | oid          |           | not null | 
 relhasindex         | boolean      |           | not null | 
 relisshared         | boolean      |           | not null | 
 relpersistence      | "char"       |           | not null | 
 relkind             | "char"       |           | not null | 
 relnatts            | smallint     |           | not null | 
 relchecks           | smallint     |           | not null | 
 relhasrules         | boolean      |           | not null | 
 relhastriggers      | boolean      |           | not null | 
 relhassubclass      | boolean      |           | not null | 
 relrowsecurity      | boolean      |           | not null | 
 relforcerowsecurity | boolean      |           | not null | 
 relispopulated      | boolean      |           | not null | 
 relreplident        | "char"       |           | not null | 
 relispartition      | boolean      |           | not null | 
 relrewrite          | oid          |           | not null | 
 relfrozenxid        | xid          |           | not null | 
 relminmxid          | xid          |           | not null | 
 relacl              | aclitem[]    |           |          | 
 reloptions          | text[]       | C         |          | 
 relpartbound        | pg_node_tree | C         |          | 
Indexes:
    "pg_class_oid_index" PRIMARY KEY, btree (oid)
    "pg_class_relname_nsp_index" UNIQUE CONSTRAINT, btree (relname, relnamespace)
    "pg_class_tblspc_relfilenode_index" btree (reltablespace, relfilenode)


```
**2. Исследование pg_tables: Получил подробное описание представления pg_tables (команда
\d+ pg_tables). Объяснил разницу между таблицей и представлением.
```sql
\d+ pg_tables
```
фрагмент вывода
```text
mydb=# \d+ pg_tables
                          View "pg_catalog.pg_tables"
   Column    |  Type   | Collation | Nullable | Default | Storage | Description 
-------------+---------+-----------+----------+---------+---------+-------------
 schemaname  | name    |           |          |         | plain   | 
 tablename   | name    |           |          |         | plain   | 
 tableowner  | name    |           |          |         | plain   | 
 tablespace  | name    |           |          |         | plain   | 
 hasindexes  | boolean |           |          |         | plain   | 
 hasrules    | boolean |           |          |         | plain   | 
 hastriggers | boolean |           |          |         | plain   | 
 rowsecurity | boolean |           |          |         | plain   | 
View definition:
 SELECT n.nspname AS schemaname,
    c.relname AS tablename,
    pg_get_userbyid(c.relowner) AS tableowner,
    t.spcname AS tablespace,
    c.relhasindex AS hasindexes,
    c.relhasrules AS hasrules,
    c.relhastriggers AS hastriggers,
    c.relrowsecurity AS rowsecurity
   FROM pg_class c
     LEFT JOIN pg_namespace n ON n.oid = c.relnamespace
     LEFT JOIN pg_tablespace t ON t.oid = c.reltablespace
  WHERE c.relkind = ANY (ARRAY['r'::"char", 'p'::"char"]);
```
```text
Признак	Таблица (table)	Представление (view)
Хранение данных	Физически хранит строки на диске	Не хранит данных (если не материализованное)
Обновление	Можно вставлять, обновлять, удалять данные	Обычно нельзя изменять напрямую (без INSTEAD OF триггеров)
Основана на	Данных самой таблицы	На SQL-запросе, который выбирает данные из одной или нескольких таблиц
Пример	students	active_students (выборка студентов с флагом active)
```

**3. Временная таблица и список схем: В базе lab02_db создайте временную таблицу. Получил
полный список всех схем в этой БД, включая системные (pg_catalog, information_schema).
Объяснил наличие временной схемы.
```sql
CREATE TEMPORARY TABLE temp_students (
    id SERIAL PRIMARY KEY,
    name VARCHAR(50),
    grade INT
);
```
Фрагмент вывода
```text
CREATE TABLE
```
-- Получение полного списка схем
```sql
SELECT nspname AS schema_name
FROM pg_namespace
ORDER BY nspname;
```
```text
  schema_name     
--------------------
 information_schema
 pg_catalog
 pg_temp_3
 pg_toast
 pg_toast_temp_3
 public
(6 rows)
```
```text
Наличие временной схемы
Каждая сессия, где создаются временные таблицы, получает свою временную схему (pg_temp_nnn).
Она нужна для изоляции данных между сессиями.
Таблицы в этой схеме автоматически удаляются при завершении сессии.
```

**4. Представления information_schema: Получил список всех представлений в схеме
information_schema.
```sql
SELECT table_name
FROM information_schema.views
WHERE table_schema = 'information_schema'
ORDER BY table_name;
```
Фрагмент вывода
```text
              table_name               
---------------------------------------
 _pg_foreign_data_wrappers
 _pg_foreign_servers
 _pg_foreign_table_columns
 _pg_foreign_tables
 _pg_user_mappings
 administrable_role_authorizations
 applicable_roles
 attributes
 character_sets
 check_constraint_routine_usage
 check_constraints
 collation_character_set_applicability
 collations
 column_column_usage
 column_domain_usage
 column_options
 column_privileges
 column_udt_usage
 columns
 constraint_column_usage
 constraint_table_usage
 data_type_privileges
 domain_constraints
 domain_udt_usage
 domains
 element_types
 enabled_roles
 foreign_data_wrapper_options
 foreign_data_wrappers
 foreign_server_options
 foreign_servers
 foreign_table_options
 foreign_tables
 information_schema_catalog_name
 key_column_usage
 parameters
 referential_constraints
 role_column_grants
 role_routine_grants
 role_table_grants
 role_udt_grants
:
```
**5. Анализ метакоманды: Выполнил в psql команду \d+ pg_views. Изучил вывод и объясните,
какие запросы к системному каталогу скрыты за этой командой.
```sql
 \d+ pg_views
```
Фрагмент вывода: 
```text
                         View "pg_catalog.pg_views"
   Column   | Type | Collation | Nullable | Default | Storage  | Description 
------------+------+-----------+----------+---------+----------+-------------
 schemaname | name |           |          |         | plain    | 
 viewname   | name |           |          |         | plain    | 
 viewowner  | name |           |          |         | plain    | 
 definition | text |           |          |         | extended | 
View definition:
 SELECT n.nspname AS schemaname,
    c.relname AS viewname,
    pg_get_userbyid(c.relowner) AS viewowner,
    pg_get_viewdef(c.oid) AS definition
   FROM pg_class c
     LEFT JOIN pg_namespace n ON n.oid = c.relnamespace
  WHERE c.relkind = 'v'::"char";
```
```text
pg_class — хранит все объекты базы (таблицы, индексы, представления).
pg_namespace — хранит схемы (чтобы узнать, в какой схеме view).
pg_roles — хранит информацию о владельцах объектов.
pg_get_viewdef(c.oid) — функция, которая возвращает SQL-запрос для представления по его OID.
c.relkind = 'v' — фильтр, чтобы выбрать только представления.
То есть \d+ pg_views — это удобная оболочка, которая показывает пользователю результат объединённых системных таблиц и функций, скрывая сложный SQL.
```
## Модуль 3: Табличные пространства
**1. Создание Tablespace: Создал каталог в файловой системе (напр.,
/home/student/mytablespace). Создал новое табличное пространство lab02_ts, указывающее
на этот каталог.
```sql
sudo -i -u postgres
mkdir -p ~/mytablespace
chmod 700 ~/mytablespace
psql -U postgres
CREATE TABLESPACE lab02_ts LOCATION '/var/lib/postgresql/mytablespace';
SELECT spcname, pg_tablespace_location(oid) AS location
FROM pg_tablespace
WHERE spcname = 'lab02_ts';
```
Фрагмент вывода: 
```text
 spcname  |             location             
----------+----------------------------------
 lab02_ts | /var/lib/postgresql/mytablespace
(1 row)

```
**2. Tablespace по умолчанию: Изменил табличное пространство по умолчанию для базы данных
template1 на lab02_ts. Объяснил цель этого действия.
```sql
ALTER DATABASE template1 SET TABLESPACE lab02_ts;
```
фрагмент вывода
```txt
ALTER DATABASE
```
--Изменение tablespace по умолчанию позволяет контролировать где будут храниться данные базы и её будущих копий, улучшая управление пространством и производительностью.

**3. Наследование свойства: Создал новую базу данных lab02_db_new. Проверил ее табличное
пространство по умолчанию. Объяснил результат.
```sql
CREATE DATABASE lab02_db_new;
\l+ lab02_db_new
```
Фрагмент вывода
```text

                                                                         List of databases
     Name     |  Owner   | Encoding | Locale Provider |   Collate   |    Ctype    | ICU Locale | ICU Rules | Access privileges |  Size   | Tablespace | Description 
--------------+----------+----------+-----------------+-------------+-------------+------------+-----------+-------------------+---------+------------+-------------
 lab02_db_new | postgres | UTF8     | libc            | en_US.UTF-8 | en_US.UTF-8 |            |           |                   | 7425 kB | lab02_ts   | 
(1 row)

(END)

```
-- Новая база данных наследует свойства табличного пространства от шаблона template1, и если вы не указали другое, используется стандартное pg_default.

**4. Символическая ссылка: Нашел в каталоге PGDATA/pg_tblspc/ символьную ссылку,
соответствующую lab02_ts. Куда она ведет?
```sql
ls -l /var/lib/postgresql/16/main/pg_tblspc
```
Фрагмент кода
```text
    table_schema    |            table_name             
--------------------+-----------------------------------
 information_schema | _pg_foreign_data_wrappers
 information_schema | _pg_foreign_servers
 information_schema | _pg_foreign_table_columns
 information_schema | _pg_foreign_tables
 information_schema | _pg_user_mappings
 information_schema | administrable_role_authorizations
 information_schema | applicable_roles
 information_schema | attributes
 information_schema | character_sets
 information_schema | check_constraint_routine_usage
(10 rows)

postgres=# 
```
-- Символическая ссылка 24594 в PGDATA/pg_tblspc/ ведёт на каталог, где физически хранится tablespace lab02_ts.

**5. Удаление Tablespace: Удалил табличное пространство lab02_ts с опцией CASCADE. Объяснил
необходимость использования CASCADE.
```sql
DROP TABLESPACE lab02_ts CASCADE;
```
Фрагмент вывода
```
DROP TABLESPACE
```
--- CASCADE — это опция, которая говорит базе данных: «При удалении этого объекта автоматически удаляй все объекты, зависящие от него».
**6. Практика+ (Параметр Tablespace): Установил параметр random_page_cost в значение 1.1 для
табличного пространства pg_default.
```sql
ALTER TABLESPACE pg_default SET (
    random_page_cost = 1.1
);
```
Фрагмент вывода: 
```text
ALTER TABLESPACE
```

## Результаты работы

**В ходе первой части лабораторной работы была создана база данных lab02_db и две схемы — app и student, в которых созданы таблицы с данными. Проверка размеров показала увеличение базы после создания объектов, что объясняется системными структурами и размещением данных на диске. Настройка search_path обеспечила корректный приоритет схем при обращении к таблицам без указания имени схемы. Параметр temp_buffers был увеличен в четыре раза для всех сеансов базы, что продемонстрировало управление настройками БД на уровне сеанса и базы данных.

**Вторая часть была посвящена системному каталогу. Изучение pg_class и pg_tables позволило понять, как PostgreSQL хранит метаданные обо всех объектах базы и различает физические таблицы и представления. Создание временной таблицы выявило работу временной схемы, обеспечивающей изоляцию данных между сессиями. Также была исследована информация о представлениях через pg_views, что показало, как сложные запросы к системному каталогу упрощаются для пользователя через встроенные метакоманды.

**Третья часть включала работу с табличными пространствами. Было создано собственное пространство lab02_ts с указанием каталога на диске, изменено табличное пространство по умолчанию для шаблонной базы template1 и создана новая база, унаследовавшая это свойство. Исследование символьной ссылки показало физическое расположение табличного пространства. В конце пространство было удалено с использованием CASCADE для удаления всех зависимых объектов, а параметр random_page_cost изменён для оптимизации работы с таблицами.

 ## Выводы
 В ходе работы я научился создавать базы данных и схемы, управлять таблицами и данными, настраивать параметры сеансов и базы данных. Я научился работать с системным каталогом PostgreSQL, различать таблицы и представления, создавать временные таблицы и понимать роль временных схем. Освоил создание и управление табличными пространствами, проверку их физического расположения и настройку параметров для оптимизации работы СУБД. В целом я научился всесторонне управлять данными и метаданными в PostgreSQL, понимать взаимосвязь логической и физической структуры хранения и использовать инструменты СУБД для администрирования и оптимизации.

