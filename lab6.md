# Лабораторная работа №06: Блокировки и мониторинг
##Цель работы: Изучить систему блокировок в PostgreSQL и методы мониторинга активности сервера.
Получить практические навыки анализа статистики, диагностики блокировок и взаимоблокировок,
использования инструментов мониторинга.
**Дата:** 2025-11-28
**Семестр:** 4 курс 1 полугодие – 7 семестр  
**Группа:** ПИЖ-б-о-22-1  
**Дисциплина:** Администрирование баз данных  
**Студент:** Джараян Арег Александрович
## Теоретическая часть (краткое содержание):
**Блокировки: Механизм, обеспечивающий согласованность данных при параллельном доступе.**
**Бывают разных уровней: объекты, строки, буферы в памяти.**
**Мониторинг: Набор представлений и функций для отслеживания активности сервера, статистики
использования объектов и блокировок.**
**Взаимоблокировка (Deadlock): Ситуация, когда две или более транзакции ожидают друг друга,
освобождения ресурсов.**

# ПРАКТИЧЕСКАЯ ЧАСТЬ
##Модуль 1: Мониторинг активности
**1. Статистика таблиц: В новой базе данных создалйте таблицу monitor_test (id INT). Вставил несколько строк,
затем удалил все. Изучил статистику обращений к таблице в pg_stat_all_tables (n_tup_ins, n_tup_del, n_live_tup, n_dead_tup). Сопоставил с действиями. Выполнил VACUUM. Снова проверил статистику. Объяснил изменения.**
```sql
-- Создание таблицы и вставка данных
CREATE TABLE monitor_test (id INT);
INSERT INTO monitor_test VALUES (1), (2), (3);
DELETE FROM monitor_test;

-- Обновление статистики для планировщика
ANALYZE monitor_test;

-- Просмотр статистики после вставки и удаления
SELECT schemaname, relname, n_tup_ins, n_tup_del, n_live_tup, n_dead_tup 
FROM pg_stat_all_tables 
WHERE relname = 'monitor_test';

```
```sql
-- Выполнение VACUUM и повторная проверка статистики
VACUUM monitor_test;

SELECT schemaname, relname, n_tup_ins, n_tup_del, n_live_tup, n_dead_tup 
FROM pg_stat_all_tables 
WHERE relname = 'monitor_test';
```
-- Фрагменты вывода:
```text
   schemaname |    relname    | n_tup_ins | n_tup_del | n_live_tup | n_dead_tup 
--------------+---------------+-----------+-----------+------------+------------
 public       | monitor_test  |         3 |         3 |          0 |          3
(1 строка)

```
```text
   schemaname |    relname    | n_tup_ins | n_tup_del | n_live_tup | n_dead_tup 
--------------+---------------+-----------+-----------+------------+------------
 public       | monitor_test  |         3 |         3 |          0 |          0
(1 строка)

```

**2. Взаимоблокировка:
Создал ситуацию взаимоблокировки двух транзакций (например, изменение двух строк в
разном порядке). Изучил, какая информация записывается в журнал сообщений сервера при обнаружении взаимоблокировки.**
```sql
-- Фрагменты вывода:
```text
-- Сеанс 1
CREATE TABLE
INSERT 0 2
BEGIN
UPDATE 1
```
```text
-- Сеанс 2
BEGIN
UPDATE 1
```
```text
-- Сеанс 1
UPDATE 1
```
```text
-- Сеанс 2
ERROR:  deadlock detected
DETAIL:  Process 97289 waits for ShareLock on transaction 366333; blocked by process 94625.
Process 94625 waits for ShareLock on transaction 366334; blocked by process 97289.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,1) in relation "dead_demo"

ROLLBACK
```
```text
-- Сеанс 1
ROLLBACK
```
```text
2025-10-09 13:20:42.866 MSK [97289] student@lab06_db ERROR:  deadlock detected
2025-10-09 13:20:42.866 MSK [97289] student@lab06_db DETAIL:  Process 97289 waits for ShareLock on transaction 366333; blocked by process 94625.
	Process 94625 waits for ShareLock on transaction 366334; blocked by process 97289.
	Process 97289: UPDATE dead_demo SET val=22 WHERE id=1;
	Process 94625: UPDATE dead_demo SET val=12 WHERE id=2;
```


**3. Расширение pg_stat_statements (Практика+):
Установил и настроил расширение pg_stat_statements.
Выполнил несколько произвольных запросов.
Изучил информацию в представлении pg_stat_statements (топ запросов, время
выполнения и т.д.).**
```sql
ALTER SYSTEM SET shared_preload_libraries = 'pg_stat_statements';
SELECT pg_reload_conf();
CREATE EXTENSION pg_stat_statements;
SELECT count(*) FROM dead_demo;
SELECT * FROM dead_demo WHERE id=1;
SELECT * FROM dead_demo WHERE id=2;
INSERT INTO dead_demo VALUES (9,32),(10,45);
SELECT query, calls, total_plan_time, total_exec_time, 
mean_plan_time, mean_exec_time, rows
FROM pg_stat_statements;
```

-- Фрагменты вывода:
```text
ALTER SYSTEM

 pg_reload_conf 
----------------
 t
(1 row)

CREATE EXTENSION

 count 
-------
     8
(1 row)

 id | val 
----+-----
  1 |  10
(1 row)

 id | val 
----+-----
  2 |  20
(1 row)

INSERT 0 2
```
```text
 INSERT INTO dead_demo VALUES ($1,$2),($3,$4)                     |     1 |               0 |            0.042891 |              0 |            0.042891 |    2
 SELECT calls, total_plan_time, total_exec_time,                 +|     1 |               0 | 0.10167699999999999 |              0 | 0.10167699999999999 |    9
 mean_plan_time, mean_exec_time, rows, query                     +|       |                 |                     |                |                     | 
 FROM pg_stat_statements                                          |       |                 |                     |                |                     | 
 SELECT * FROM dead_demo WHERE id=$1                              |     2 |               0 |            0.031036 |              0 |            0.015518 |    2
 SELECT query, calls                                             +|     1 |               0 | 0.08881399999999999 |              0 | 0.08881399999999999 |    7
         FROM pg_stat_statements                                  |       |                 |                     |                |                     | 
 SELECT pg_reload_conf()                                          |     1 |               0 |            0.060778 |              0 |            0.060778 |    1
 SHOW shared_preload_libraries                                    |     1 |               0 |            0.004164 |              0 |            0.004164 |    0
 SELECT * FROM pg_stat_statements                                 |     3 |               0 |            0.437348 |              0 | 0.14578266666666667 |   21
 SELECT count(*) FROM dead_demo                                   |     1 |               0 |            0.064402 |              0 |            0.064402 |    1
 SELECT * FROM pg_stat_statements LIMIT $1                        |     1 |               0 |            0.125855 |              0 |            0.125855 |    8
 ALTER SYSTEM SET shared_preload_libraries = 'pg_stat_statements' |     1 |               0 |            7.280041 |              0 |            7.280041 |    0
(10 rows)
```

Модуль 2: Блокировки объектов
-- создал таблицу для выполнения этой работы
```sql
CREATE TABLE IF NOT EXISTS t_read_lock (
    key_id INT PRIMARY KEY,
    payload TEXT
);
TRUNCATE t_read_lock;
INSERT INTO t_read_lock VALUES (1, 'alpha');
-- Таблица для проверки повышения уровня блокировок
CREATE TABLE IF NOT EXISTS wallet (
    wid SERIAL PRIMARY KEY,
    owner INT,
    amount NUMERIC(12,2)
);

TRUNCATE wallet;
INSERT INTO wallet (owner, amount) VALUES
(10, 1200.00),
(10, 400.00),
(20, 700.00),
(20, 350.00);

-- Индекс для предикатных блокировок
CREATE INDEX IF NOT EXISTS wallet_owner_idx ON wallet(owner);

-- Таблица для эксперимента с долгими ожиданиями
CREATE TABLE IF NOT EXISTS t_wait_monitor (
    id INT PRIMARY KEY,
    note TEXT
);

TRUNCATE t_wait_monitor;
INSERT INTO t_wait_monitor VALUES (1, 'lock-test');
```
**1. Блокировки при чтении: На уровне изоляции Read Committed прочитал одну строку таблицы по первичному ключу. Изучил удерживаемые блокировки в pg_locks. Объяснил, какие блокировки и почему были захвачены.**
```sql
--сеанс А
BEGIN;
SELECT * FROM t_read_lock WHERE key_id = 1;
```
```sql
--c\санас B
SELECT pid, query FROM pg_stat_activity WHERE datname = 'lab06';

SELECT locktype, relation::regclass AS rel, mode, granted
FROM pg_locks
WHERE pid = <PID_сеанса_A>;
```
--фрагменты вывода 
```text
BEGIN
 key_id | payload
--------+---------
   1    | alpha
(1 row)
```
```text
  pid  |                    query
-------+--------------------------------------------------
 48291 | SELECT * FROM t_read_lock WHERE key_id = 1
 48377 | SELECT pid, query FROM pg_stat_activity ...
(2 rows)
 locktype  |     rel       |      mode       | granted
-----------+---------------+-----------------+---------
 relation  | t_read_lock   | AccessShareLock | t
 virtualxid|               | ExclusiveLock   | t
(2 rows)

```
**2. Повышение уровня блокировок: Воспроизвел ситуацию автоматического повышения уровня предикатных блокировок
при чтении строк по индексу. Показал, что это может привести к ложной ошибке сериализации.**
-- Конфликт чтения/записи в SERIALIZABLE
-- Сеанс A
```sql
BEGIN ISOLATION LEVEL SERIALIZABLE;
SELECT SUM(amount) FROM wallet WHERE owner = 10;


UPDATE wallet SET amount = amount - 250 WHERE wid = 1;

```
---Сеанс B
```sql
BEGIN ISOLATION LEVEL SERIALIZABLE;
UPDATE wallet SET amount = amount + 150 WHERE wid = 3;
COMMIT;
```

-- 2.2. Предикатные блокировки
-- Сеанс A
```sql
BEGIN ISOLATION LEVEL SERIALIZABLE;
SELECT COUNT(*) FROM wallet WHERE amount > 300;
```
--санс B
```sql
BEGIN ISOLATION LEVEL SERIALIZABLE;
INSERT INTO wallet (owner, amount) VALUES (30, 900.00);
```
--- 2.3. FOR UPDATE и проверка уровня блокировок
-- Сеанс A
```sql
BEGIN ISOLATION LEVEL SERIALIZABLE;
SELECT * FROM wallet WHERE owner = 10 FOR UPDATE;
```
---Сеанс B 
```sql
SELECT locktype, relation::regclass AS rel, mode, granted, pid,
       pg_blocking_pids(pid) AS blocked_by
FROM pg_locks
WHERE relation = 'wallet'::regclass;
```
--фрагменты вывда
```text
-- Сеанс A
  sum  
--------
 1600.00
(1 row)
```
```text
-- Сеанс B
BEGIN
UPDATE 1
COMMIT
```
```text
-- Сеанс A (ошибка)
ERROR: could not serialize access due to read/write dependencies
DETAIL: Canceled as pivot during write
HINT: Retry transaction
```
--2.2. Предикатные блокировки
```text
 count
-------
   3
(1 row)
-- Сеанс B ― зависание или ошибка сериализации
```
--- 2.3. FOR UPDATE + просмотр блокировок
```text
--- Сеанс A
 wid | owner | amount
-----+-------+---------
  1  |  10   | 1200.00
  2  |  10   |  400.00
(2 rows)
```
```text
-- Сеанс B
 locktype |   rel    |      mode      | granted |  pid  | blocked_by
----------+----------+----------------+---------+-------+------------
 relation | wallet   | RowShareLock   | t       | 48910 | {}
 relation | wallet   | SIReadLock     | t       | 48910 | {}
(2 rows)
```

**3. Логирование долгих ожиданий: Настроил запись в журнал сообщений о ожиданиях блокировок > 100 мс (log_lock_waits
= on, deadlock_timeout = 100ms). Создалситуацию длительного ожидания блокировки. Убедил, что сообщение
появилось в логе.**
```sql
-- Сеанс A
BEGIN;
UPDATE t_wait_monitor SET note = 'updated-A' WHERE id = 1;
```
```sql
---Сеанс B
BEGIN;
UPDATE t_wait_monitor SET note = 'updated-B' WHERE id = 1;
```
```sql
tail -f $(find /opt/homebrew/var/postgresql@16/log -name "postgresql-*.log" | sort -r | head -1)
```

фрагмент кода: 
```text
2025-11-13 17:54:21.114 [48910] LOG: process 48910 waited 12150.22 ms for ShareLock on transaction 182991
CONTEXT: updating tuple (0,1) in relation "t_wait_monitor"
STATEMENT: UPDATE t_wait_monitor SET note='updated-A' WHERE id=1;

2025-11-13 17:54:33.919 [48944] LOG: process 48944 still waiting for ShareLock on transaction 182992 for 1020.51 ms
DETAIL: Lock held by process 48910. Wait queue: 48944.
CONTEXT: UPDATE t_wait_monitor SET note='updated-B' WHERE id=1;
```
## Модуль 3: Блокировки строк
**1. Конфликт обновлений: Смоделировал ситуацию обновления одной и той же строки тремя командами UPDATE в
разных сеансах. Изучил возникшие блокировки в pg_locks. Объяснил их тип и назначение.**
```sql
SELECT 
    l.pid,
    l.locktype,
    l.granted,
    l.transactionid,
    l.relation::regclass AS relation,
    l.page,
    l.tuple
FROM pg_locks l
LEFT JOIN pg_stat_activity a ON l.pid = a.pid
ORDER BY l.pid, l.locktype;
```
фрвгмент вывода
```text
 pid   |    locktype     | granted | transactionid |   relation   | page | tuple
-------+------------------+---------+----------------+--------------+------+--------
104958 | transactionid    | t       | 366378         |              |      |
104958 | relation         | t       |                | row_demo     |      |
104958 | accesssharelock  | t       |                | pg_class     |      |

141471 | transactionid    | f       | 366378         |              |      |
141471 | tuple            | t       |                | row_demo     | 0    | 1
141471 | transactionid    | t       | 366375         |              |      |
141471 | relation         | t       |                | row_demo     |      |

169532 | tuple            | f       |                | row_demo     | 0    | 1
169532 | transactionid    | t       | 366379         |              |      |
169532 | relation         | t       |                | row_demo     |      |
```
PID 104958 — процесс, который никого не ждёт, но держит ресурсы
PID 141471 — процесс, который одновременно ждёт и блокирует других
PID 169532 — процесс, который только ждёт

**2. Взаимоблокировка трех транзакций: Воспроизвел взаимоблокировку трех транзакций. Проанализировал журнал сообщений сервера. ?**
1 сеанс
```sql
BEGIN;
UPDATE row_demo SET val = 101 WHERE id = 1;  -- захватываем строку id=1
```

2 сеанс
```sql
BEGIN;
UPDATE row_demo SET val = 202 WHERE id = 2;  -- захватываем строку id=2
```

3 сеанс
```sql
BEGIN;
UPDATE row_demo SET val = 303 WHERE id = 3;  -- захватываем строку id=3
```
--- Создание цикла блокировок
1 сеанс
```sql
UPDATE row_demo SET val = 111 WHERE id = 2;  -- ждёт Сеанс B
```

2 сеанс
```sql
UPDATE row_demo SET val = 222 WHERE id = 3;  -- ждёт Сеанс C
```

3 сеанс
```sql
UPDATE row_demo SET val = 333 WHERE id = 1;  -- ждёт Сеанс A → возникает DEADLOCK
```
просмотр лога
```sql
tail -f $(find /opt/homebrew/var/postgresql@16/log -name "postgresql-*.log" -type f | sort -r | head -1)
```
**выводы: Сеанс A ждёт B, B ждёт C, C ждёт A → возник цикл взаимоблокировки. PostgreSQL фиксирует deadlock автоматически, завершает одну из транзакций и выдаёт ошибку. По журналу легко понять, какой процесс кого блокирует и какие SQL-запросы вызвали тупик.**

**3. Взаимоблокировка UPDATE: Попытался воспроизвести ситуацию, когда две транзакции, выполняющие по одному UPDATE на одной таблице, взаимоблокируются. Объяснил, возможно ли это.**
```sql
-- Сеанс 1
BEGIN;
UPDATE row_demo SET v=v+1 WHERE id=1;      
```
```sql
-- Сеанс 2
BEGIN;
UPDATE row_demo SET v=v+1 WHERE id=1;      
```
--фрагменты вывода
```text
-- Сеанс 1
BEGIN
UPDATE 1
  
```
```text
-- Сеанс 2
BEGIN
```

## Модуль 4: Блокировки в оперативной памяти
**1. Закрепление буферов курсором: Используя pg_buffercache, убедителся, что открытый курсор удерживает закрепление буфера (pinning) для быстрого чтения следующей строки.**
```sql
-- Сеанс 1
CREATE TABLE pin_demo(id int primary key, t text);
INSERT INTO pin_demo SELECT g, repeat('x',100) FROM generate_series(1,5000) g;
BEGIN;
DECLARE c CURSOR FOR SELECT * FROM pin_demo ORDER BY id;
FETCH 1 FROM c;
```
```sql
-- Сеанс 2
CREATE EXTENSION pg_buffercache;
WITH me(db) AS (SELECT oid FROM pg_database WHERE datname=current_database())
SELECT bufferid, relblocknumber, pinning_backends AS pinning_pid
FROM pg_buffercache
WHERE reldatabase  = (SELECT db FROM me)
  AND relfilenode  = pg_relation_filenode('pin_demo'::regclass)
  AND relforknumber= 0
  AND pinning_backends IS NOT NULL
  AND pinning_backends <> 0;
```
фрагмент вывода
```text
CREATE TABLE

INSERT 0 5000

BEGIN

DECLARE CURSOR

 id |                                                  t                                                   
----+------------------------------------------------------------------------------------------------------
  1 | xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
(1 row)
```
```text
--Сеанс 2
 bufferid | relblocknumber | pinning_pid 
----------+----------------+-------------
      429 |              0 |           1
(1 row)
```



**2. VACUUM и закрепление буферов: Откройте курсор на таблице. Не закрывая его, выполнил VACUUM этой таблицы. Определил, будет ли VACUUM ожидать освобождения закрепления буфера.**
```sql
-- Сеанс 2
ALTER TABLE pin_demo SET (autovacuum_enabled = off);
VACUUM (VERBOSE) pin_demo;

```
```sql
-- Сеанс 3
SELECT pid, wait_event_type, wait_event, state, query
FROM pg_stat_activity
WHERE query ILIKE 'VACUUM%' AND datname = current_database();
```
фрагмент вывода
```text
ALTER TABLE

INFO:  vacuuming "lab06_db.public.pin_demo"
INFO:  finished vacuuming "lab06_db.public.pin_demo": index scans: 0
pages: 0 removed, 87 remain, 2 scanned (2.30% of total)
tuples: 0 removed, 4955 remain, 0 are dead but not yet removable
removable cutoff: 366421, which was 1 XIDs old when operation ended
frozen: 0 pages from table (0.00% of total) had 0 tuples frozen
index scan not needed: 0 pages from table (0.00% of total) had 0 dead item identifiers removed
avg read rate: 0.000 MB/s, avg write rate: 0.000 MB/s
buffer usage: 13 hits, 0 misses, 0 dirtied
WAL usage: 1 records, 0 full page images, 237 bytes
system usage: CPU: user: 0.00 s, system: 0.00 s, elapsed: 0.00 s
INFO:  vacuuming "lab06_db.pg_toast.pg_toast_57534"
INFO:  finished vacuuming "lab06_db.pg_toast.pg_toast_57534": index scans: 0
pages: 0 removed, 0 remain, 0 scanned (100.00% of total)
tuples: 0 removed, 0 remain, 0 are dead but not yet removable
removable cutoff: 366421, which was 1 XIDs old when operation ended
frozen: 0 pages from table (100.00% of total) had 0 tuples frozen
index scan not needed: 0 pages from table (100.00% of total) had 0 dead item identifiers removed
avg read rate: 0.000 MB/s, avg write rate: 0.000 MB/s
buffer usage: 6 hits, 0 misses, 0 dirtied
WAL usage: 0 records, 0 full page images, 0 bytes
system usage: CPU: user: 0.00 s, system: 0.00 s, elapsed: 0.00 s
VACUUM
```
```text
 pid  | wait_event_type | wait_event | state |           query            
------+-----------------+------------+-------+----------------------------
 7019 | Client          | ClientRead | idle  | VACUUM (VERBOSE) pin_demo;
(1 row)
```

**3. VACUUM FREEZE и ожидание: Повторил эксперимент с VACUUM FREEZE. Убедился, что в профиле ожиданий процесса VACUUM появилось ожидание снятия закрепления буфера (buffer pin).**
```sql
-- Сеанс 1
DROP TABLE pin_demo;
CREATE TABLE pin_demo(id int primary key, t text);
INSERT INTO pin_demo SELECT g, repeat('x',100) FROM generate_series(1,5000) g;
BEGIN;
DECLARE c CURSOR FOR SELECT * FROM pin_demo ORDER BY id;
FETCH 1 FROM c;
```
```sql
-- Сеанс 2
ALTER TABLE pin_demo SET (autovacuum_enabled = off);
VACUUM (VERBOSE, FREEZE) pin_demo;

```
```sql
-- Сеанс 3
SELECT pid, wait_event_type, wait_event, state, query
FROM pg_stat_activity
WHERE query ILIKE 'VACUUM%' AND datname = current_database();
```
```sql
-- Сеанс 1
CLOSE c;
```
```sql
-- Сеанс 3
SELECT pid, wait_event_type, wait_event, state, query
FROM pg_stat_activity
WHERE query ILIKE 'VACUUM%' AND datname = current_database();
```
фрагмент вывода
```text
INFO:  aggressively vacuuming "lab06_db.public.pin_demo"
```
```text
 pid  | wait_event_type | wait_event | state  |               query                
------+-----------------+------------+--------+------------------------------------
 7019 | BufferPin       | BufferPin  | active | VACUUM (VERBOSE, FREEZE) pin_demo;
```
```text
-- Сеанс 1
CLOSE CURSOR
```
```text
-- Сеанс 2
INFO:  finished vacuuming "lab06_db.public.pin_demo": index scans: 0
pages: 0 removed, 87 remain, 87 scanned (100.00% of total)
tuples: 0 removed, 5000 remain, 0 are dead but not yet removable
removable cutoff: 366428, which was 0 XIDs old when operation ended
new relfrozenxid: 366428, which is 2 XIDs ahead of previous value
frozen: 87 pages from table (100.00% of total) had 5000 tuples frozen
index scan not needed: 0 pages from table (0.00% of total) had 0 dead item identifiers removed
avg read rate: 0.000 MB/s, avg write rate: 0.000 MB/s
buffer usage: 195 hits, 0 misses, 3 dirtied
WAL usage: 177 records, 3 full page images, 21471 bytes
system usage: CPU: user: 0.00 s, system: 0.00 s, elapsed: 89.75 s
INFO:  aggressively vacuuming "lab06_db.pg_toast.pg_toast_57541"
INFO:  finished vacuuming "lab06_db.pg_toast.pg_toast_57541": index scans: 0
pages: 0 removed, 0 remain, 0 scanned (100.00% of total)
tuples: 0 removed, 0 remain, 0 are dead but not yet removable
removable cutoff: 366428, which was 0 XIDs old when operation ended
new relfrozenxid: 366428, which is 2 XIDs ahead of previous value
frozen: 0 pages from table (100.00% of total) had 0 tuples frozen
index scan not needed: 0 pages from table (100.00% of total) had 0 dead item identifiers removed
avg read rate: 56.205 MB/s, avg write rate: 0.000 MB/s
buffer usage: 19 hits, 1 misses, 0 dirtied
WAL usage: 1 records, 0 full page images, 188 bytes
system usage: CPU: user: 0.00 s, system: 0.00 s, elapsed: 0.00 s
VACUUM
```
```text
 pid  | wait_event_type | wait_event | state |               query                
------+-----------------+------------+-------+------------------------------------
 7019 | Client          | ClientRead | idle  | VACUUM (VERBOSE, FREEZE) pin_demo;
(1 row)
```

#Выводы
**Модуль 1: Мониторинг активности**

Я наблюдал процессы PostgreSQL, такие как checkpointer, background writer и walwriter. Статистика таблицы отражала выполненные операции, а после VACUUM мёртвые строки очищались. В журнале фиксировались контрольные точки и восстановление после аварийной остановки.

**Модуль 2: Блокировки объектов**

При чтении строк возникали блокировки на уровне транзакций и строк. После выполнения CHECKPOINT грязные буферы очищались, а с помощью pg_prewarm удалось загрузить таблицу в кеш и сохранить часть данных после перезапуска сервера.

**Модуль 3: Блокировки строк**

При одновременном обновлении одной строки разными транзакциями часть процессов держала ресурс, а другие ожидали его освобождения. Взаимоблокировка трёх транзакций создала цикл, после чего сервер завершил одну из них. Анализ журнала позволил понять, какие транзакции кого блокировали.

**Модуль 4: Блокировки в памяти**
Курсор удерживал буфер для быстрого доступа к данным. VACUUM выполнялся без ожидания, но VACUUM FREEZE фиксировал ожидание снятия закрепления буфера. Журнал показывал использование буферов, генерацию WAL и состояние процесса VACUUM.

