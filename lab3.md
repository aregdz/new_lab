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
