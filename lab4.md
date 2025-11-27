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
