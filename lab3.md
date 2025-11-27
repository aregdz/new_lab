### Модель многопользовательского доступа: MVCC

**Дата:** 2025-10-27  
**Семестр:** 4 курс 1 полугодие – 7 семестр  
**Группа:** ПИЖ-б-о-22-1  
**Дисциплина:** Администрирование баз данных  
**Студент:** Джараян Арег Александрович

### Цель работы
Изучить принципы многоверсионного управления конкурентным доступом (MVCC) в PostgreSQL. Получить практические навыки наблюдения за работой MVCC, анализа версий строк, снимков данных и уровней изоляции транзакций. Освоить использование расширений и системных представлений для исследования внутренней структуры данных.

### Теоретическая часть (кратко):
MVCC (Multiversion Concurrency Control) позволяет нескольким транзакциям работать с одними данными одновременно, минимизируя блокировки. Каждая транзакция видит согласованный «снимок» данных на момент своего начала. При изменении строки создаётся новая версия, старая хранится до очистки. Системные поля: xmin — транзакция, создавшая версию, xmax — транзакция, удалившая или заблокировавшая версию, ctid — физическое расположение строки. Уровни изоляции: Read Committed — видны только зафиксированные данные, Repeatable Read — предотвращает неповторяемое чтение, Serializable — полностью исключает аномалии. Снимок данных определяет, какие версии строк видимы транзакции.

### Модуль 1: Уровни изоляции и аномалии
## 1. Read Committed vs Удаление:
Создал таблицу iso_test (id INT, data TEXT) и вставил одну строку.
В сеансе 1 начал транзакцию с уровнем READ COMMITTED и выполнил SELECT * FROM
iso_test;.
В сеансе 2 удалил строку и зафиксировал изменения (DELETE ...; COMMIT;).
В сеансе 1 выполнил тот же SELECT повторно. 
```sql
-- Сеанс 1
CREATE DATABASE lab03_db;
\c lab03_db
CREATE TABLE iso_test(id INT, data TEXT);
INSERT INTO iso_test VALUES (1,'row1');

BEGIN ISOLATION LEVEL READ COMMITTED;
SELECT * FROM iso_test;
```
```sql
-- Сеанс 2
\c lab03_db
DELETE FROM iso_test WHERE id=1;
COMMIT;
```
```sql
-- Сеанс 1
SELECT * FROM iso_test;
COMMIT;
```
Фрагмент вывода: 
```text
CREATE DATABASE
You are now connected to database "lab03_db" as user "student".

CREATE TABLE

INSERT 0 1

BEGIN

 id | data 
----+------
  1 | row1
(1 row)
```
```text
You are now connected to database "lab03_db" as user "student".

DELETE 1

WARNING:  there is no transaction in progress
COMMIT

```
```text
 id | data 
----+------
  1 | row1
(1 row)

COMMIT
```
**Вывод:**  cоздалась таблица, добавили данные и начали транзакцию. Пока видим одну строку. Псоле удаление всех данные из таблицы и подтверждение изменений. После а уровне `READ COMMITTED` мы увидим, что строки исчезли, потому что эта транзакция видит изменения, подтвержденные другими.


## 2. Repeatable Read vs Удаление:
Повторил предыдущий эксперимент, но в сеансе 1 начал транзакцию с BEGIN
ISOLATION LEVEL REPEATABLE READ;.
Объяснил разницу в результатах между двумя уровнями изоляции.
```sql
-- Сеанс 1
INSERT INTO iso_test VALUES (1,'row1');
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT * FROM iso_test;
```
```sql
-- Сеанс 2
DELETE FROM iso_test WHERE id=1;
COMMIT;
```
```sql
-- Сеанс 1
SELECT * FROM iso_test;
COMMIT;
```
Фрагменты вывода: 
```text
INSERT 0 1

BEGIN

 id | data 
----+------
  1 | row1
(1 row)

```
```text
DELETE 1

WARNING:  there is no transaction in progress
COMMIT
lab03_db=# 
```
**Вывод:** На уровне `REPEATABLE READ` мы продолжим видеть строку, даже если ее удалили в другом сеансе. Транзакция работает с "снимком" данных на момент своего начала.

## 3. Создание таблицы в транзакции:
В сеансе 1 начал транзакцию и создал новую таблицу new_table, вставил в нее строку.
Не фиксировал. В сеансе 2 выполнил SELECT * FROM new_table;. Что произойдет?
Зафиксировал транзакцию в сеансе 1. Повторил запрос в сеансе 2.
Повторил процесс, но вместо фиксации откатил транзакцию в сеансе 1. Что изменилось?
```sql
-- Сеанс 1
BEGIN;
CREATE TABLE new_table(id int);
INSERT INTO new_table VALUES (10);
```
```sql
-- Сеанс 2
SELECT * FROM new_table;
```
```sql
-- Сеанс 1
COMMIT;
```
```sql
-- Сеанс 2
SELECT * FROM new_table;
```
```sql
-- Сеанс 1
DROP TABLE new_table;
BEGIN;
CREATE TABLE new_table(id int);
INSERT INTO new_table VALUES (10);
```
```sql
-- Сеанс 2
SELECT * FROM new_table;
```
```sql
-- Сеанс 1
ROLLBACK;
```
```sql
-- Сеанс 2
SELECT * FROM new_table;
```
```text
BEGIN
CREATE TABLE
INSERT 0 1
```
```text
ERROR:  relation "new_table" does not exist
LINE 1: SELECT * FROM new_table;
```
```text
COMMIT
```
```text
 id 
----
 10
(1 row)
```
```text
DROP TABLE
BEGIN
CREATE TABLE
INSERT 0 1
```
```text
ROLLBACK
```
```text
ERROR:  relation "new_table" does not exist
LINE 1: SELECT * FROM new_table;
                      ^
```
```text
ERROR:  relation "new_table" does not exist
LINE 1: SELECT * FROM new_table;
                      ^
```
**Вывод:** Если транзакция с созданием таблицы не зафиксирована, другие сеансы не видят таблицу и при SELECT получат ошибку. После COMMIT таблица и данные становятся видимыми для всех. При ROLLBACK таблица и данные удаляются, и другие сеансы снова не могут к ней обратиться.

## 4. Блокировка DDL:
В сеансе 1 начал транзакцию и выполнил SELECT * FROM iso_test; (даже если
таблица пуста). Попытался в сеансе 2 выполнить DROP TABLE iso_test;. Получится ли? Объясните,
почему.
```sql
-- Сеанс 1
BEGIN;
SELECT * FROM iso_test;
```

```sql
-- Сеанс 2
DROP TABLE iso_test;
```

```sql
-- Сеанс 1
COMMIT;
```
-- фаргмент вывода
```text
-- Сеанс 1
BEGIN
 id | data 
----+------
(0 rows)
```

```text
-- Сеанс 1
COMMIT
```
```text
-- Сеанс 2
DROP TABLE
```
**Вывод:** Объяснение: Сеанс 1 держит блокировку на таблицу, поэтому DROP TABLE в сеансе 2 не может получить эксклюзивную блокировку и ждёт завершения транзакции в сеансе 1.

### Модуль 2: Фантомное чтение и снимки
**1. Фантомное чтение (Read Committed):**
Создал пустую таблицу phantom_test (id INT).
Продемонстрировал на уровне Read Committed, что аномалия "фантомное чтение" не
предотвращается (вставка новых строк в другом сеансе становится видимой).
```sql
-- Сеанс 1
CREATE TABLE phantom_test(id INT);
BEGIN ISOLATION LEVEL READ COMMITTED;
SELECT count(*) FROM phantom_test;  -- 0
```
```sql
-- Сеанс 2
INSERT INTO phantom_test VALUES (1),(2),(3);
COMMIT;
```
```sql
-- Сеанс 1
SELECT count(*) FROM phantom_test;
COMMIT;
```
Фрагменты выводов: 
```text
-- Сеанс 1
CREATE TABLE

BEGIN

 count 
-------
     0
(1 row)
```
```text
-- Сеанс 2
INSERT 0 3

WARNING:  there is no transaction in progress
COMMIT
```
```text
-- Сеанс 1
 count 
-------
     3
(1 row)

COMMIT
```
**Выводы:** На уровне Read Committed транзакция видит только зафиксированные данные на момент запроса. В примере сеанс 1 сначала видит таблицу пустой, а после фиксации вставки в сеансе 2 видит новые строки. Это демонстрирует фантомное чтение, которое на этом уровне не предотвращается.
**2. Невидимость удалений (Repeatable Read):**
В сеансе 1 начал транзакцию с уровнем Repeatable Read (пока без запросов).
В сеансе 2 удалил все строки из phantom_test и зафиксировал.
В сеансе 1 выполните SELECT * FROM phantom_test;. Увидятся ли удаленные строки?
Выполнил в сеансе 1 запрос SELECT * FROM pg_database; (не касаясь phantom_test).
Повлияет ли это на видимость строк в phantom_test при последующем запросе?
```sql
-- Сеанс 1
BEGIN ISOLATION LEVEL REPEATABLE READ;
```
```sql
-- Сеанс 2
DELETE FROM phantom_test;
COMMIT;
```
```sql
-- Сеанс 1
SELECT * FROM phantom_test; 
SELECT * FROM pg_database;   
SELECT * FROM phantom_test;
COMMIT;
```
Фрагменты вывода

```text
-- Сеанс 1
BEGIN
```
```text
-- Сеанс 2
DELETE 3

WARNING:  there is no transaction in progress
COMMIT
```
```text
 id 
----
  1
  2
  3
(3 rows)
```
**Выводы**: После удаления в сеансе 2 строки в сеансе 1 при SELECT * FROM phantom_test; будут видны. Запрос к pg_database не влияет на видимость строк в phantom_test.
**3. Транзакционность DDL:** Убедился, что DROP TABLE является транзакционной операцией (можно
откатить).
```sql
CREATE TABLE drop_demo(id int);
BEGIN;
DROP TABLE drop_demo;
ROLLBACK;

SELECT * FROM drop_demo;
```
Фрагменты вывода: 
```text
CREATE TABLE

BEGIN

DROP TABLE

ROLLBACK

 id 
----
(0 rows)
```
**Выводы: ** Даже после отката таблица drop_demo продолжает существовать, но она пустая. Это подтверждает, что DROP TABLE в PostgreSQL является транзакционной операцией и может быть отменена с помощью ROLLBACK.

### Модуль 3: Версии строк и pageinspect
**1. Жизненный цикл строки:**
Создал таблицу version_test (id INT). Вставил одну строку.
Дважды обновил эту строку (UPDATE ...), а затем удалил ее (DELETE).
Используя расширение pageinspect (heap_page_items), определил, сколько версий строк
находится сейчас в таблице. Объяснил их состояние по полям t_xmin, t_xmax, t_ctid.
```sql
CREATE EXTENSION pageinspect;
CREATE TABLE version_test(id INT);
INSERT INTO version_test VALUES (1);
UPDATE version_test SET id=2;    
UPDATE version_test SET id=3;    
DELETE FROM version_test;           
```
```sql
SELECT t_xmin, t_xmax, t_ctid  FROM heap_page_items(get_raw_page('version_test',0));
```
Фрагменты вывода
```text
CREATE EXTENSION

CREATE TABLE

INSERT 0 1

UPDATE 1

UPDATE 1

DELETE 1
```
```text
 t_xmin | t_xmax | t_ctid 
--------+--------+--------
    836 |    837 | (0,2)
    837 |    838 | (0,3)
    838 |    839 | (0,3)
(3 rows)
```
**Выводы:** После двух обновлений и удаления в таблице остаются три версии строки. У каждой t_xmin соответствует транзакции, которая создала версию, t_xmax указывает транзакцию, которая сделала её устаревшей или удалила. t_ctid отражает физическое положение каждой версии на странице.



**2. Анализ системной таблицы:**
Определил, в какой странице (блоке) находится строка в pg_class, описывающая саму
таблицу pg_class.
Используя pageinspect, подсчитал количество актуальных (видимых) версий строк в
этой странице.
```sql
SELECT ctid FROM pg_class WHERE relname = 'pg_class';
```
```sql
SELECT t_xmin, t_xmax, t_ctid  FROM heap_page_items(get_raw_page('pg_class',0));
```
Фрагменты вывода
```text
  ctid  
--------
 (7,65)
(1 row)
```
```text

 t_xmin | t_xmax | t_ctid 
--------+--------+--------
        |        | 
    778 |    778 | (0,5)
    778 |    788 | (0,3)
    778 |    788 | (0,4)
    778 |    788 | (0,5)
    778 |    788 | (0,6)
        |        | 
        |        | 
        |        | 
        |        | 
        |        | 
        |        | 
        |        | 
        |        | 
        |        | 
        |        | 
        |        | 
        |        | 
        |        | 
        |        | 
        |        | 
        |        | 
        |        | 
        |        | 
        |        | 
        |        | 
        |        | 
        |        | 
        |        | 
        |        | 
        |        | 
        |        | 
        |        | 
        |        | 
        |        | 
        |        | 
        |        | 
        |        | 
        |        | 
        |        | 
        |        | 
        |        | 
        |        | 
        |        | 
        |        | 
    270 |      0 | (0,46)
    522 |      0 | (0,47)
(47 row)
```

**3. ON_ERROR_ROLLBACK:** Включил в psql параметр ON_ERROR_ROLLBACK. Создал ситуацию с
ошибкой в транзакции и убедителся, что этот режим использует точки сохранения (SAVEPOINT),
позволяя продолжить работу транзакции после ошибки.
```sql
\set ON_ERROR_ROLLBACK on
BEGIN;
CREATE TABLE err_demo(id int PRIMARY KEY);
INSERT INTO err_demo VALUES (1);
INSERT INTO err_demo VALUES (1);  
INSERT INTO err_demo VALUES (2);    
COMMIT;
SELECT * FROM err_demo;              
```
фрагменты вывода
```text
BEGIN
   
COMMIT;

CREATE TABLE

INSERT 0 1

ERROR:  duplicate key value violates unique constraint "err_demo_pkey"
DETAIL:  Key (id)=(1) already exists.

INSERT 0 1

COMMIT

 id 
----
  1
  2
(2 rows)
```

