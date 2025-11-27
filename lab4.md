### Лабораторная работа №04: Техобслуживание: Очистка (VACUUM)
## Цель работы: Всестороннее изучение механизмов очистки (VACUUM) в PostgreSQL. Получение
практических навыков управления ручной и автоматической очисткой, анализа работы HOTобновлений, исследования влияния очистки на размер таблиц и индексов, а также работы с
заморозкой версий строк.
Семестр: 7 семестр
Группа: ПИЖ-б-о-22-1
Дисциплина: Администрирование баз данных
Студент: Джараян Арег Александроич
### Теоретическая часть (краткое содержание):
**Очистка (VACUUM):** Удаляет мертвые версии строк (кортежи), ставшие таковыми в результате
**UPDATE или DELETE.** Освобождает место для повторного использования. Не уменьшает
физический размер файла таблицы.
**Полная очистка (VACUUM FULL):** Перезаписывает файл таблицы, полностью удаляя мертвые
кортежи и уплотняя данные. Уменьшает физический размер файла, но требует эксклюзивной
блокировки.
**Автоочистка (AUTOVACUUM):** Фоновый процесс, автоматически выполняющий VACUUM для
таблиц по мере накопления мертвых кортежей.
**HOT-обновления (Heap-Only Tuple):** Специальный вид UPDATE, при котором новая версия
строки помещается на ту же страницу, что и старая. Это позволяет избежать обновления
индексов и повышает производительность.
**Заморозка (FREEZE):** Помечает версии строк как "замороженные", что позволяет предотвратить
проблему оборачивания идентификаторов транзакций (XID).
### Практическая часть
### Модуль 1: Ручная очистка и ее влияние
**1. Отключение автоочистки: Глобально отключил процесс автоочистки (autovacuum = off в
конфиге, требует перезагрузки, или ALTER SYSTEM + pg_reload_conf()). Убедился, что он не
работает.**
```sql
CREATE DATABASE lab04_db;
\c lab04_db
CREATE EXTENSION pageinspect;

ALTER SYSTEM SET autovacuum = off;
SELECT pg_reload_conf();
```

```sql
SELECT name, setting
FROM pg_settings
WHERE name IN ('autovacuum');
```
--вывод резльтатов
```text
CREATE DATABASE

You are now connected to database "lab04_db" as user "student".

ALTER SYSTEM

 pg_reload_conf 
----------------
 t
```
```text
    name    | setting 
------------+---------
 autovacuum | off
(1 row)

```
**2. Подготовка данных: В новой базе данных создал таблицу vacuum_test (id INT) и индекс по
полю id. Вставил в таблицу 100 000 случайных чисел.**
```sql
CREATE TABLE vacuum_test(id int);
CREATE INDEX ON vacuum_test(id);
INSERT INTO vacuum_test
SELECT (random()*1000000) int FROM generate_series(1,100000);
```
--вывод резльтатов
```text
CREATE TABLE
CREATE INDEX
INSERT 0 100000
```

**3. Наблюдение без очистки: Несколько раз (3-5) обновил половину строк в таблице (UPDATE vacuum_test SET id =
id + 1 WHERE random() < 0.5;). После каждого обновления контролируйте размер таблицы и индекса с помощью
pg_total_relation_size. Зафиксировал рост размеров.**
```sql
-- До обновлений
SELECT pg_indexes_size('vacuum_test');
SELECT pg_total_relation_size('vacuum_test');

-- Обновление 1
UPDATE vacuum_test SET id = id + 1 WHERE random() < 0.5;
SELECT pg_indexes_size('vacuum_test');
SELECT pg_total_relation_size('vacuum_test');

-- Обновление 2
UPDATE vacuum_test SET id = id + 1 WHERE random() < 0.5;
SELECT pg_indexes_size('vacuum_test');
SELECT pg_total_relation_size('vacuum_test');

-- Обновление 3
UPDATE vacuum_test SET id = id + 1 WHERE random() < 0.5;
SELECT pg_indexes_size('vacuum_test');
SELECT pg_total_relation_size('vacuum_test');
```
```text
-- До обновлений
 pg_indexes_size 
-----------------
         2555904
(1 row)

 pg_total_relation_size 
------------------------
                6209536
(1 row)


-- Обновление 1
UPDATE 49981

 pg_indexes_size 
-----------------
         4718592
(1 row)

 pg_total_relation_size 
------------------------
               10182656
(1 row)

-- Обновление 2
UPDATE 50020

 pg_indexes_size 
-----------------
         5046272
(1 row)

 pg_total_relation_size 
------------------------
               11419648
(1 row)

-- Обновление 3
UPDATE 49966

 pg_indexes_size 
-----------------
         7462912
(1 row)

 pg_total_relation_size 
------------------------
               15056896
(1 row)
```

**4. Полная очистка: Выполнил VACUUM FULL vacuum_test;. Сравнил размеры таблицы и индекса
до и после.**
```sql
VACUUM FULL vacuum_test;
SELECT pg_indexes_size('vacuum_test');
SELECT pg_total_relation_size('vacuum_test');
```
--вывод резльтатов
```text
VACUUM

 pg_indexes_size 
-----------------
         2236416
(1 row)

 pg_total_relation_size 
------------------------
                5865472
(1 row)
```

**5. Обычная очистка: Повторил цикл обновлений из пункта 3, но после каждого обновления вызывал
обычную очистку (VACUUM vacuum_test;). Сравнил динамику размеров с результатами из пункта 3.**
```sql
-- Обновление 1
UPDATE vacuum_test SET id = id + 1 WHERE random() < 0.5;
VACUUM vacuum_test;
SELECT pg_indexes_size('vacuum_test');
SELECT pg_total_relation_size('vacuum_test');

-- Обновление 2
UPDATE vacuum_test SET id = id + 1 WHERE random() < 0.5;
VACUUM vacuum_test;
SELECT pg_indexes_size('vacuum_test');
SELECT pg_total_relation_size('vacuum_test');

-- Обновление 3
UPDATE vacuum_test SET id = id + 1 WHERE random() < 0.5;
VACUUM vacuum_test;
SELECT pg_indexes_size('vacuum_test');
SELECT pg_total_relation_size('vacuum_test');
```
--вывод резльтатов
```text
UPDATE 49934

VACUUM

 pg_indexes_size 
-----------------
         4464640
(1 row)

 pg_total_relation_size 
------------------------
                9936896
(1 row)
```

```text
UPDATE 50005
VACUUM
 pg_indexes_size 
-----------------
         4464640
(1 row)

 pg_total_relation_size 
------------------------
                9936896
(1 row)
```
```text
UPDATE 50287
VACUUM
 pg_indexes_size 
-----------------
         4464640
(1 row)

 pg_total_relation_size 
------------------------
                9945088
(1 row)
```
**6. Включение автоочистки: Включил автоочистку обратно.**
```sql
ALTER SYSTEM RESET autovacuum;
SELECT pg_reload_conf();
```
--вывод резльтатов
```text
ALTER SYSTEM
 pg_reload_conf 
----------------
 t
(1 row)
```
