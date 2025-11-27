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

### Модуль 2: HOT-обновления и самоочистка

**В ходе выполнения модуля я создал таблицы, выполнил обновления и с помощью pageinspect исследовал, как PostgreSQL формирует версии строк, выполняет HOT-обновления и осуществляет самоочистку.**
```sql
CREATE TABLE no_hot(a int, b text);
INSERT INTO no_hot SELECT g, 'x' FROM generate_series(1,2000) g;

SELECT * FROM heap_page_items(get_raw_page('no_hot', 0)) LIMIT 20;
UPDATE no_hot SET a = a + 1 WHERE a % 2 = 0;
UPDATE no_hot SET a = a + 1 WHERE a % 3 = 0;
UPDATE no_hot SET a = a + 1 WHERE a % 4 = 0;     
CREATE INDEX ON no_hot(a);
SELECT * FROM heap_page_items(get_raw_page('no_hot', 0)) LIMIT 20;
VACUUM no_hot;
SELECT * FROM heap_page_items(get_raw_page('no_hot', 0)) LIMIT 20;
```
```text
CREATE TABLE
INSERT 0 2000
 lp | lp_off | lp_flags | lp_len | t_xmin | t_xmax | t_field3 | t_ctid | t_infomask2 | t_infomask | t_hoff | t_bits | t_oid |     t_data     
----+--------+----------+--------+--------+--------+----------+--------+-------------+------------+--------+--------+-------+----------------
  1 |   8160 |        1 |     30 |    967 |      0 |        0 | (0,1)  |           2 |       2050 |     24 |        |       | \x010000000578
  2 |   8128 |        1 |     30 |    967 |      0 |        0 | (0,2)  |           2 |       2050 |     24 |        |       | \x020000000578
  3 |   8096 |        1 |     30 |    967 |      0 |        0 | (0,3)  |           2 |       2050 |     24 |        |       | \x030000000578
  4 |   8064 |        1 |     30 |    967 |      0 |        0 | (0,4)  |           2 |       2050 |     24 |        |       | \x040000000578
  5 |   8032 |        1 |     30 |    967 |      0 |        0 | (0,5)  |           2 |       2050 |     24 |        |       | \x050000000578
  6 |   8000 |        1 |     30 |    967 |      0 |        0 | (0,6)  |           2 |       2050 |     24 |        |       | \x060000000578
  7 |   7968 |        1 |     30 |    967 |      0 |        0 | (0,7)  |           2 |       2050 |     24 |        |       | \x070000000578
  8 |   7936 |        1 |     30 |    967 |      0 |        0 | (0,8)  |           2 |       2050 |     24 |        |       | \x080000000578
  9 |   7904 |        1 |     30 |    967 |      0 |        0 | (0,9)  |           2 |       2050 |     24 |        |       | \x090000000578
 10 |   7872 |        1 |     30 |    967 |      0 |        0 | (0,10) |           2 |       2050 |     24 |        |       | \x0a0000000578
 11 |   7840 |        1 |     30 |    967 |      0 |        0 | (0,11) |           2 |       2050 |     24 |        |       | \x0b0000000578
 12 |   7808 |        1 |     30 |    967 |      0 |        0 | (0,12) |           2 |       2050 |     24 |        |       | \x0c0000000578
 13 |   7776 |        1 |     30 |    967 |      0 |        0 | (0,13) |           2 |       2050 |     24 |        |       | \x0d0000000578
 14 |   7744 |        1 |     30 |    967 |      0 |        0 | (0,14) |           2 |       2050 |     24 |        |       | \x0e0000000578
 15 |   7712 |        1 |     30 |    967 |      0 |        0 | (0,15) |           2 |       2050 |     24 |        |       | \x0f0000000578
 16 |   7680 |        1 |     30 |    967 |      0 |        0 | (0,16) |           2 |       2050 |     24 |        |       | \x100000000578
 17 |   7648 |        1 |     30 |    967 |      0 |        0 | (0,17) |           2 |       2050 |     24 |        |       | \x110000000578
 18 |   7616 |        1 |     30 |    967 |      0 |        0 | (0,18) |           2 |       2050 |     24 |        |       | \x120000000578
 19 |   7584 |        1 |     30 |    967 |      0 |        0 | (0,19) |           2 |       2050 |     24 |        |       | \x130000000578
 20 |   7552 |        1 |     30 |    967 |      0 |        0 | (0,20) |           2 |       2050 |     24 |        |       | \x140000000578
(20 rows)

UPDATE 1000
UPDATE 667
UPDATE 334
CREATE INDEX
 lp | lp_off | lp_flags | lp_len | t_xmin | t_xmax | t_field3 | t_ctid  | t_infomask2 | t_infomask | t_hoff | t_bits | t_oid |     t_data     
----+--------+----------+--------+--------+--------+----------+---------+-------------+------------+--------+--------+-------+----------------
  1 |   8160 |        1 |     30 |    967 |      0 |        0 | (0,1)   |           2 |       2306 |     24 |        |       | \x010000000578
  2 |      0 |        3 |      0 |        |        |          |         |             |            |        |        |       | 
  3 |   8128 |        1 |     30 |    967 |    969 |        0 | (0,227) |       16386 |       1282 |     24 |        |       | \x030000000578
  4 |      0 |        3 |      0 |        |        |          |         |             |            |        |        |       | 
  5 |   8096 |        1 |     30 |    967 |      0 |        0 | (0,5)   |           2 |       2306 |     24 |        |       | \x050000000578
  6 |      0 |        3 |      0 |        |        |          |         |             |            |        |        |       | 
  7 |   8064 |        1 |     30 |    967 |      0 |        0 | (0,7)   |           2 |       2306 |     24 |        |       | \x070000000578
  8 |      0 |        3 |      0 |        |        |          |         |             |            |        |        |       | 
  9 |   8032 |        1 |     30 |    967 |    969 |        0 | (0,228) |       16386 |       1282 |     24 |        |       | \x090000000578
 10 |      0 |        3 |      0 |        |        |          |         |             |            |        |        |       | 
 11 |   8000 |        1 |     30 |    967 |      0 |        0 | (0,11)  |           2 |       2306 |     24 |        |       | \x0b0000000578
 12 |      0 |        3 |      0 |        |        |          |         |             |            |        |        |       | 
 13 |   7968 |        1 |     30 |    967 |      0 |        0 | (0,13)  |           2 |       2306 |     24 |        |       | \x0d0000000578
 14 |      0 |        3 |      0 |        |        |          |         |             |            |        |        |       | 
 15 |   7936 |        1 |     30 |    967 |    969 |        0 | (0,229) |       16386 |       1282 |     24 |        |       | \x0f0000000578
 16 |      0 |        3 |      0 |        |        |          |         |             |            |        |        |       | 
 17 |   7904 |        1 |     30 |    967 |      0 |        0 | (0,17)  |           2 |       2306 |     24 |        |       | \x110000000578
 18 |      0 |        3 |      0 |        |        |          |         |             |            |        |        |       | 
 19 |   7872 |        1 |     30 |    967 |      0 |        0 | (0,19)  |           2 |       2306 |     24 |        |       | \x130000000578
 20 |      0 |        3 |      0 |        |        |          |         |             |            |        |        |       | 
(20 rows)

VACUUM
 lp | lp_off | lp_flags | lp_len | t_xmin | t_xmax | t_field3 | t_ctid | t_infomask2 | t_infomask | t_hoff | t_bits | t_oid |     t_data     
----+--------+----------+--------+--------+--------+----------+--------+-------------+------------+--------+--------+-------+----------------
  1 |   8160 |        1 |     30 |    967 |      0 |        0 | (0,1)  |           2 |       2306 |     24 |        |       | \x010000000578
  2 |      0 |        0 |      0 |        |        |          |        |             |            |        |        |       | 
  3 |    265 |        2 |      0 |        |        |          |        |             |            |        |        |       | 
  4 |      0 |        0 |      0 |        |        |          |        |             |            |        |        |       | 
  5 |   8128 |        1 |     30 |    967 |      0 |        0 | (0,5)  |           2 |       2306 |     24 |        |       | \x050000000578
  6 |      0 |        0 |      0 |        |        |          |        |             |            |        |        |       | 
  7 |   8096 |        1 |     30 |    967 |      0 |        0 | (0,7)  |           2 |       2306 |     24 |        |       | \x070000000578
  8 |      0 |        0 |      0 |        |        |          |        |             |            |        |        |       | 
  9 |    228 |        2 |      0 |        |        |          |        |             |            |        |        |       | 
 10 |      0 |        0 |      0 |        |        |          |        |             |            |        |        |       | 
 11 |   8064 |        1 |     30 |    967 |      0 |        0 | (0,11) |           2 |       2306 |     24 |        |       | \x0b0000000578
 12 |      0 |        0 |      0 |        |        |          |        |             |            |        |        |       | 
 13 |   8032 |        1 |     30 |    967 |      0 |        0 | (0,13) |           2 |       2306 |     24 |        |       | \x0d0000000578
 14 |      0 |        0 |      0 |        |        |          |        |             |            |        |        |       | 
 15 |    266 |        2 |      0 |        |        |          |        |             |            |        |        |       | 
 16 |      0 |        0 |      0 |        |        |          |        |             |            |        |        |       | 
 17 |   8000 |        1 |     30 |    967 |      0 |        0 | (0,17) |           2 |       2306 |     24 |        |       | \x110000000578
 18 |      0 |        0 |      0 |        |        |          |        |             |            |        |        |       | 
 19 |   7968 |        1 |     30 |    967 |      0 |        0 | (0,19) |           2 |       2306 |     24 |        |       | \x130000000578
 20 |      0 |        0 |      0 |        |        |          |        |             |            |        |        |       | 
(20 rows)
```
**Сначала я работал с таблицей без индексов: вставил данные, выполнил обновления, не удовлетворяющие условиям HOT, и проанализировал страницы до и после самоочистки, наблюдая появление и исчезновение мёртвых кортежей. Затем я создал таблицу с индексом и исследовал HOT-обновление: выполнил изменение неиндексируемого поля и убедился, что новая версия строки осталась на той же странице, а запись в индексе не изменилась.**
```sql
CREATE TABLE hot_test(id int, payload text);
CREATE INDEX hot_test_id_idx ON hot_test(id);
INSERT INTO hot_test VALUES (1, 'a');
UPDATE hot_test SET payload = payload || repeat('a', 50) WHERE id = 1;
SELECT lp, t_xmin, t_xmax, t_ctid
FROM heap_page_items(get_raw_page('hot_test', 0));
SELECT itemoffset, ctid
FROM bt_page_items('hot_test_id_idx', 1);
```
-- фрагмент выода
```text
CREATE TABLE
CREATE INDEX
INSERT 0 1
UPDATE 1
 lp | t_xmin | t_xmax | t_ctid 
----+--------+--------+--------
  1 |    991 |    992 | (0,2)
  2 |    992 |      0 | (0,2)
(2 rows)

 itemoffset | ctid
------------+--------------
          1 | (0,1)
(1 row)
```

**После этого я смоделировал ситуацию нехватки свободного места для HOT-обновления: выполнил обновление и увидел, что новая версия была размещена на другой странице. Далее я проверил количество ссылок в индексе на соответствующий кортеж и объяснил полученный результат, связанный с невозможностью продолжать HOT-цепочку при переносе версии на другую страницу.**
``` sql
CREATE TABLE hot_test2 (
    id INT PRIMARY KEY, 
    data TEXT, 
    padding TEXT
) WITH (fillfactor=50);  -- Оставляем 50% свободного места на странице

CREATE INDEX idx_hot_test2 ON hot_test2(data);

INSERT INTO hot_test2 
SELECT i, 'data_' || i, rpad('x', 200, 'x') 
FROM generate_series(1, 50) i;

SELECT lp, t_xmin, t_xmax, t_ctid 
FROM heap_page_items(get_raw_page('hot_test2', 0)) 
LIMIT 5;

UPDATE hot_test2 SET padding = rpad('y', 400, 'y') WHERE id = 1;

SELECT lp, t_xmin, t_xmax, t_ctid 
FROM heap_page_items(get_raw_page('hot_test2', 0)) 
LIMIT 5;

SELECT lp, t_xmin, t_xmax, t_ctid 
FROM heap_page_items(get_raw_page('hot_test2', 1)) 
LIMIT 5;
```
--фрагменты вывода
```
You're about to run a destructive command.
Do you want to proceed? [y/N]: y
Your call!
CREATE TABLE
CREATE INDEX
INSERT 0 50

+----+--------+--------+--------+
| lp | t_xmin | t_xmax | t_ctid |
|----+--------+--------+--------|
| 1  | 910    | 0      | (0,1)  |
| 2  | 910    | 0      | (0,2)  |
| 3  | 910    | 0      | (0,3)  |
| 4  | 910    | 0      | (0,4)  |
| 5  | 910    | 0      | (0,5)  |
+----+--------+--------+--------+
SELECT 5

UPDATE 1

+----+--------+--------+--------+
| lp | t_xmin | t_xmax | t_ctid |
|----+--------+--------+--------|
| 1  | 910    | 911    | (0,17) |
| 2  | 910    | 0      | (0,2)  |
| 3  | 910    | 0      | (0,3)  |
| 4  | 910    | 0      | (0,4)  |
| 5  | 910    | 0      | (0,5)  |
+----+--------+--------+--------+
SELECT 5

+----+--------+--------+--------+
| lp | t_xmin | t_xmax | t_ctid |
|----+--------+--------+--------|
| 1  | 910    | 0      | (1,1)  |
| 2  | 910    | 0      | (1,2)  |
| 3  | 910    | 0      | (1,3)  |
| 4  | 910    | 0      | (1,4)  |
| 5  | 910    | 0      | (1,5)  |
+----+--------+--------+--------+
SELECT 5

Time: 0.047s
```



### Модуль 3: Глубокая очистка и параметры
##1. Многопроходная очистка индекса: Создал большую таблицу с индексом.
Временно уменьшил параметр maintenance_work_mem до очень маленького значения.
Сгенерировал большое количество мертвых кортечей (много UPDATE/DELETE).
Запустил VACUUM VERBOSE vacuum_test;. Проконтролировал в выводе, что очистка
индекса потребовала нескольких проходов.
```sql
CREATE TABLE vacuum_test(id int, pad text);
CREATE INDEX vacuum_test_idx ON vacuum_test(id);
INSERT INTO vacuum_test
SELECT g, repeat('z', 100) FROM generate_series(1,1000000) g;
SET maintenance_work_mem = '1MB';
UPDATE vacuum_test SET pad = repeat('z',120) WHERE id % 2 = 0;
DELETE FROM vacuum_test WHERE id % 10 = 0;
VACUUM VERBOSE vacuum_test; 
```
-- вывод
```text
CREATE TABLE

CREATE INDEX

INSERT 0 1000000

SET

UPDATE 500000

DELETE 100000

INFO:  vacuuming "lab04_db.public.vacuum_test"
INFO:  finished vacuuming "lab04_db.public.vacuum_test": index scans: 4
pages: 0 removed, 26857 remain, 26857 scanned (100.00% of total)
tuples: 100000 removed, 900000 remain, 0 are dead but not yet removable
removable cutoff: 1018, which was 0 XIDs old when operation ended
new relfrozenxid: 1015, which is 2 XIDs ahead of previous value
frozen: 0 pages from table (0.00% of total) had 0 tuples frozen
index scan needed: 26857 pages from table (100.00% of total) had 600000 dead item identifiers removed
index "vacuum_test_idx": pages: 5486 in total, 0 newly deleted, 0 currently deleted, 0 reusable
avg read rate: 340.087 MB/s, avg write rate: 173.720 MB/s
buffer usage: 62392 hits, 40152 misses, 20510 dirtied
WAL usage: 74267 records, 3 full page images, 7120684 bytes
system usage: CPU: user: 0.33 s, system: 0.30 s, elapsed: 0.92 s
INFO:  vacuuming "lab04_db.pg_toast.pg_toast_32903"
INFO:  finished vacuuming "lab04_db.pg_toast.pg_toast_32903": index scans: 0
pages: 0 removed, 0 remain, 0 scanned (100.00% of total)
tuples: 0 removed, 0 remain, 0 are dead but not yet removable
removable cutoff: 1018, which was 0 XIDs old when operation ended
new relfrozenxid: 1018, which is 5 XIDs ahead of previous value
frozen: 0 pages from table (100.00% of total) had 0 tuples frozen
index scan not needed: 0 pages from table (100.00% of total) had 0 dead item identifiers removed
avg read rate: 24.094 MB/s, avg write rate: 6.024 MB/s
buffer usage: 16 hits, 4 misses, 1 dirtied
WAL usage: 1 records, 0 full page images, 188 bytes
system usage: CPU: user: 0.00 s, system: 0.00 s, elapsed: 0.00 s
VACUUM
```
```text
CREATE TABLE

ALTER SYSTEM

ALTER SYSTEM

ALTER TABLE

 pg_reload_conf 
----------------
 t
(1 row)
```
##2. Обычная очистка после удаления: Удалил 90% случайных строк из большой таблицы.
Выполнил обычную очистку (VACUUM). Проверил, изменился ли физический размер
таблицы на диске.
``` sql
SELECT pg_size_pretty(pg_total_relation_size('large_vacuum_test')) as size_before_delete;

DELETE FROM large_vacuum_test WHERE random() < 0.9;

SELECT pg_size_pretty(pg_total_relation_size('large_vacuum_test')) as size_after_delete;

VACUUM large_vacuum_test;

SELECT pg_size_pretty(pg_total_relation_size('large_vacuum_test')) as size_after_vacuum;
```
```text
+--------------------+
| size_before_delete |
|--------------------|
| 132 MB             |
+--------------------+
SELECT 1

DELETE 45014

+-------------------+
| size_after_delete |
|-------------------|
| 132 MB            |
+-------------------+
SELECT 1
VACUUM

+-------------------+
| size_after_vacuum |
|-------------------|
| 132 MB            |
+-------------------+
SELECT 1
Time: 0.616s
```

##3. Полная очистка после удаления: Повторил удаление 90% строк. Выполнил полную очистку (VACUUM FULL). Сравнил результат с предыдущим пунктом.
```sql
DROP TABLE vacuum_test;
CREATE TABLE vacuum_test(id int, pad text);
CREATE INDEX vacuum_test_idx ON vacuum_test(id);
INSERT INTO vacuum_test
SELECT g, repeat('z', 100) FROM generate_series(1,1000000) g;
-- до удаления
SELECT pg_table_size('vacuum_test');
SELECT pg_total_relation_size('vacuum_test');
DELETE FROM vacuum_test WHERE random() < 0.9;
-- после удаления
SELECT pg_table_size('vacuum_test');
SELECT pg_total_relation_size('vacuum_test');
VACUUM FULL vacuum_test;
-- после очистки
SELECT pg_table_size('vacuum_test');
SELECT pg_total_relation_size('vacuum_test');             
```

--вывод 
```text
DROP TABLE
CREATE TABLE
CREATE INDEX
INSERT 0 1000000
-- до удаления
 pg_table_size 
---------------
     141312000
(1 row)

 pg_total_relation_size 
------------------------
              163799040
(1 row)

DELETE 900226
-- после удаления
 pg_table_size 
---------------
     141312000
(1 row)

 pg_total_relation_size 
------------------------
              163799040
(1 row)

VACUUM
-- после очистки
 pg_table_size 
---------------
      14106624
(1 row)

 pg_total_relation_size 
------------------------
               16359424
(1 row)
```

### Модуль 4: Автоочистка и заморозка

##1. Настройка автоочистки. Я настроил автоочистку для тестовой таблицы: установил порог в 10% изменённых строк через autovacuum_vacuum_scale_factor = 0.1 и уменьшил интервал запуска до 1 секунды (autovacuum_naptime = 1s).
```sql
CREATE TABLE av_test(id int, payload text);
ALTER SYSTEM SET autovacuum_naptime = '1s';
ALTER SYSTEM SET log_autovacuum_min_duration = '0';
ALTER TABLE av_test SET (autovacuum_vacuum_scale_factor = 0.1);
SELECT pg_reload_conf();
```

##2. Нагрузочный тест. Я создал таблицу примерно на 100 000 строк и в цикле выполнял серию транзакций, каждую из которых изменяла около 5–6 % случайных строк. Параллельно я отслеживал срабатывания автоочистки через логи и после завершения цикла проанализировал итоговый размер таблицы, сопоставив его с выбранными параметрами автоочистки.
```sql
INSERT INTO av_test
SELECT g, repeat('a',50) FROM generate_series(1,100000) g;
ANALYZE av_test;
SELECT pg_table_size('av_test'); 
SELECT pg_total_relation_size('av_test');
```
```bash
for i in {1..20}; do
  psql -d lab04_db -c "UPDATE av_test SET payload = repeat('b',50)||id WHERE random() < 0.06";
  sleep 2
done
```
```sql
SELECT relname, n_dead_tup, autovacuum_count, vacuum_count, analyze_count
FROM pg_stat_user_tables
WHERE relname = 'av_test';
SELECT pg_table_size('av_test'); 
SELECT pg_total_relation_size('av_test');
```
--фрагмент выводов
```text
```sql
CREATE TABLE av_test(id int, payload text);
ALTER SYSTEM SET autovacuum_naptime = '1s';
ALTER SYSTEM SET log_autovacuum_min_duration = '0';
ALTER TABLE av_test SET (autovacuum_vacuum_scale_factor = 0.1);
SELECT pg_reload_conf();
```
```text
INSERT 0 100000
ANALYZE
 pg_table_size 
---------------
       8478720
(1 row)

 pg_total_relation_size 
------------------------
                8478720
(1 row)
```
```text
UPDATE 6027
UPDATE 5967
UPDATE 5937
UPDATE 5948
UPDATE 5926
UPDATE 6092
UPDATE 6052
UPDATE 5983
UPDATE 5974
UPDATE 5944
UPDATE 6017
UPDATE 5938
UPDATE 5984
UPDATE 5749
UPDATE 5992
UPDATE 6145
UPDATE 6002
UPDATE 5987
UPDATE 5933
UPDATE 6029
```
```text
 relname | n_dead_tup | autovacuum_count | vacuum_count | analyze_count 
---------+------------+------------------+--------------+---------------
 av_test |       6482 |                5 |            0 |             1
(1 row)

 pg_table_size 
---------------
      10067968
(1 row)

 pg_total_relation_size 
------------------------
               10067968
(1 row)
```
##3. Заморозка версий. С помощью pageinspect я проверил, что COPY … WITH FREEZE загружает строки уже в замороженном состоянии — их xmin имеет специальное значение. Я также убедился, что такие строки остаются видимыми даже в транзакции уровня Repeatable Read, начатой перед загрузкой данных.
```sql
-- Сеанс 1
CREATE TABLE frz_test(id int, note text);
```
```sql
-- Сеанс 2
BEGIN ISOLATION LEVEL REPEATABLE READ;
```
```sql
-- Сеанс 1
BEGIN;
TRUNCATE frz_test;
COPY frz_test (id, note)
FROM STDIN WITH FREEZE;
1 a
2 b
3 c
\.
COMMIT;
```
```sql
-- Сеанс 1
BEGIN;
TRUNCATE frz_test;
COPY frz_test (id, note)
FROM STDIN WITH FREEZE;
1 a
2 b
3 c
\.
COMMIT;
SELECT id, xmin, age(xmin) AS age
FROM frz_test ORDER BY id;
```
```sql
-- Сеанс 2
SELECT count(*) FROM frz_test;
SELECT id, xmin, age(xmin) AS age
FROM frz_test ORDER BY id;
COMMIT;
```
--фрагмент выводов
```text
-- Сеанс 1
CREATE TABLE
```
```text
-- Сеанс 2
BEGIN
```
```text
-- Сеанс 1
BEGIN

TRUNCATE TABLE

COPY 3

COMMIT;

 id | xmin | age 
----+------+-----
  1 | 1194 |   1
  2 | 1194 |   1
  3 | 1194 |   1
(3 rows)

```
```text
-- Сеанс 1
 id | note 
----+------
  1 | a
  2 | b
  3 | c
(3 rows)

 id | xmin | age 
----+------+-----
  1 | 1194 |  -1
  2 | 1194 |  -1
  3 | 1194 |  -1
(3 rows)
```

##4. Принудительная очистка и заморозка. Я уменьшил параметр autovacuum_freeze_max_age до минимального, отключил автоочистку для таблицы и выполнил большое число транзакций, чтобы превысить порог возраста. После ручного VACUUM была выполнена агрессивная очистка с заморозкой, что подтвердилось записями в логах.
``` sql
DROP TABLE frz_test;
CREATE TABLE frz_test(id int, note text);
ALTER SYSTEM SET autovacuum_freeze_max_age = '100000';
SELECT pg_reload_conf();
ALTER TABLE frz_test SET (autovacuum_enabled = off);

SELECT 'SELECT txid_current();' FROM generate_series(1,150000);
\gexec


VACUUM (VERBOSE, FREEZE) frz_test;
SELECT relname, age(relfrozenxid)
FROM pg_class
WHERE relname = 'frz_test';
```
-- вывод фрагмента
```text
DROP TABLE

CREATE TABLE

ALTER SYSTEM

 pg_reload_conf 
----------------
 t
(1 row)

ALTER TABLE
```

```text
INFO:  aggressively vacuuming "lab04_db.public.frz_test"
INFO:  finished vacuuming "lab04_db.public.frz_test": index scans: 0
pages: 0 removed, 0 remain, 0 scanned (100.00% of total)
tuples: 0 removed, 0 remain, 0 are dead but not yet removable
removable cutoff: 366281, which was 0 XIDs old when operation ended
new relfrozenxid: 366281, which is 150002 XIDs ahead of previous value
frozen: 0 pages from table (100.00% of total) had 0 tuples frozen
index scan not needed: 0 pages from table (100.00% of total) had 0 dead item identifiers removed
avg read rate: 0.000 MB/s, avg write rate: 15.976 MB/s
buffer usage: 4 hits, 0 misses, 1 dirtied
WAL usage: 1 records, 1 full page images, 5097 bytes
system usage: CPU: user: 0.00 s, system: 0.00 s, elapsed: 0.00 s
INFO:  aggressively vacuuming "lab04_db.pg_toast.pg_toast_33038"
INFO:  finished vacuuming "lab04_db.pg_toast.pg_toast_33038": index scans: 0
pages: 0 removed, 0 remain, 0 scanned (100.00% of total)
tuples: 0 removed, 0 remain, 0 are dead but not yet removable
removable cutoff: 366281, which was 0 XIDs old when operation ended
new relfrozenxid: 366281, which is 150002 XIDs ahead of previous value
frozen: 0 pages from table (100.00% of total) had 0 tuples frozen
index scan not needed: 0 pages from table (100.00% of total) had 0 dead item identifiers removed
avg read rate: 37.380 MB/s, avg write rate: 74.761 MB/s
buffer usage: 19 hits, 1 misses, 2 dirtied
WAL usage: 3 records, 2 full page images, 3190 bytes
system usage: CPU: user: 0.00 s, system: 0.00 s, elapsed: 0.00 s
VACUUM

 relname  | age 
----------+-----
 frz_test |   0
(1 row)
```


### Отчетная часть
## Результаты выполнения

1. **Процессы и режимы завершения работы.**  
   В лабораторных экспериментах определены фоновые процессы PostgreSQL, ответственные за буферизацию и надёжность — `checkpointer`, `background writer` и `walwriter`.  
   При остановке в режиме `fast` в журнале фиксируется сообщение о штатной контрольной точке, после которой при перезапуске восстановление не требуется.  
   При остановке в режиме `immediate` сервер стартует с этапа `crash recovery`: в логах появляются строки `database system was not properly shut down; automatic recovery in progress`, `redo starts at ...`, `redo done at ...`. Это демонстрирует автоматическое восстановление базы до последней контрольной точки и применения всех WAL-записей после неё.

2. **Буферный кеш и контрольные точки.**  
   Используя расширения `pg_buffercache` и `pg_stat_bgwriter`, измерено количество «грязных» буферов до и после выполнения `CHECKPOINT;`. До контрольной точки их было более 3000, после — 0, что подтверждает успешный сброс данных на диск.  
   Эксперименты с расширением `pg_prewarm` показали, что таблица успешно загружается в буферный кеш (около 2900 буферов), но после перезапуска кеш полностью очищается, что подтверждает его оперативную, а не постоянную природу.

3. **Исследование объёма и структуры WAL-записей.**  
   После фиксации транзакций были измерены значения LSN (`pg_current_wal_lsn()`), и вычислена разница между ними — около 185 888 байт. Анализ через `pg_waldump` выявил в WAL-файлах типичные записи: `Heap INSERT`, `Btree INSERT`, `Storage CREATE`, а также `FPW` (full page writes).  
   Наличие FPW объясняет значительный размер журнала сразу после контрольной точки, поскольку при первой модификации каждой страницы в WAL сохраняется полная копия 8 КиБ-страницы для защиты от частичных записей при сбое.

4. **Восстановление после сбоя.**  
   При аварийном завершении (`kill -9 postmaster`) незавершённые транзакции откатываются, а зафиксированные остаются. В логах после перезапуска отображаются фазы восстановления: `redo starts at …`, `redo done …`, `checkpoint starting: end-of-recovery immediate wait`.  
   Это подтверждает корректность механизма *Write-Ahead Logging*: база возвращается в согласованное состояние, полностью сохраняя коммиты и откатывая незавершённые изменения.

5. **Настройка параметров WAL и оценка их влияния.**  
   - При `full_page_writes = on` каждая первая модификация страницы после контрольной точки записывает полный образ (FPW). Это повышает надёжность, но увеличивает размер WAL примерно втрое.  
   - При `full_page_writes = off` WAL-файлы уменьшаются, однако при сбое повышается риск повреждения страниц.  
   - Включение `wal_compression = on` позволило сократить объём WAL-записей примерно в четыре раза без потери устойчивости, что особенно эффективно при изменении больших, но частично идентичных страниц.  
   Эти опыты показывают баланс между надёжностью, скоростью записи и объёмом журнала.

6. **Анализ поведения при сбоях и контроль согласованности.**  
   После восстановления журнал фиксировал корректное завершение `redo` и создание новой контрольной точки конца восстановления (*checkpoint complete: wrote N buffers …*). Проверка таблиц после сбоя показала отсутствие нарушений целостности, что подтверждает атомарность транзакций и надёжность подсистемы WAL.

## Выводы
1. Изучил работу буферного кеша и механизма журналирования предзаписи (WAL) в PostgreSQL. 
2. Получил практические навыки управления контрольными точками, анализа журнальных записей, настройки параметров WAL
3. Исследовал процессы восстановления после сбоев.

