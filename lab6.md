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
```bash
ps -ef | grep -E "post(gres|master)|checkpointer|writer|wal" | grep -v grep
```
-- Фрагменты вывода:
```text
/main -c config_file=/etc/postgresql/16/main/postgresql.conf
postgres     797     787  0 08:34 ?        00:00:00 postgres: 16/main: checkpointer 
postgres     798     787  0 08:34 ?        00:00:00 postgres: 16/main: background writer 
postgres     977     787  0 08:34 ?        00:00:00 postgres: 16/main: walwriter 
postgres     978     787  0 08:34 ?        00:00:02 postgres: 16/main: autovacuum launcher 
postgres     979     787  0 08:34 ?        00:00:00 postgres: 16/main: logical replication launcher 
student     6148    1601  0 08:46 pts/0    00:00:00 /usr/lib/postgresql/16/bin/psql
```


**2. Взаимоблокировка:
Создал ситуацию взаимоблокировки двух транзакций (например, изменение двух строк в
разном порядке). Изучил, какая информация записывается в журнал сообщений сервера при обнаружении взаимоблокировки.**
```bash
sudo pg_ctlcluster 16 main stop -m fast
sudo pg_ctlcluster 16 main start
sudo tail -n 200 /var/log/postgresql/postgresql-16-main.log
```
-- Фрагменты вывода:
```text
2025-10-09 08:53:20.243 MSK [797] LOG:  checkpoint starting: shutdown immediate
2025-10-09 08:53:20.250 MSK [797] LOG:  checkpoint complete: wrote 0 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.001 s, sync=0.001 s, total=0.009 s; sync files=0, longest=0.000 s, average=0.000 s; distance=0 kB, estimate=22877 kB; lsn=0/64E8C0F0, redo lsn=0/64E8C0F0
```


**3. Расширение pg_stat_statements (Практика+):
Установил и настроил расширение pg_stat_statements.
Выполнил несколько произвольных запросов.
Изучил информацию в представлении pg_stat_statements (топ запросов, время
выполнения и т.д.).**
```bash
sudo pg_ctlcluster 16 main stop -m immediate
sudo pg_ctlcluster 16 main start
sudo tail -n 200 /var/log/postgresql/postgresql-16-main.log
```
-- Фрагменты вывода:
```text
2025-10-09 10:06:08.573 MSK [38667] LOG:  database system was not properly shut down; automatic recovery in progress
2025-10-09 10:06:08.786 MSK [38667] LOG:  redo starts at 0/5837B058
2025-10-09 10:06:08.984 MSK [38667] LOG:  invalid record length at 0/5837B058: expected at least 24, got 0
2025-10-09 10:06:09.345 MSK [38667] LOG:  redo done at 0/5837B058 system usage: CPU: user: 0.00 s, system: 0.00 s, elapsed: 0.00 s
2025-10-09 10:06:09.563 MSK [38666] LOG:  checkpoint starting: end-of-recovery immediate wait
2025-10-09 10:06:12.351 MSK [39813] LOG:  checkpoint complete: wrote 3 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.003 s, sync=0.002 s, total=0.013 s; sync files=2, longest=0.001 s, average=0.001 s; distance=0 kB, estimate=0 kB; lsn=0/5837B058, redo lsn=0/5837B058
```

Модуль 2: Блокировки объектов
**1. Блокировки при чтении: На уровне изоляции Read Committed прочитал одну строку таблицы по первичному
ключу. Изучил удерживаемые блокировки в pg_locks. Объяснил, какие блокировки и почему были захвачены.**
```sql
CREATE DATABASE lab05_db;
\c lab05_db
CREATE TABLE wal_test(id int PRIMARY KEY, data text);
INSERT INTO wal_test
SELECT g, repeat('x', 200)
FROM generate_series(1, 100000) AS g;
SELECT pg_relation_size('wal_test');
SELECT current_setting('block_size')::int;
WITH rel AS (
  SELECT oid, pg_relation_filenode(oid) AS fn
  FROM pg_class WHERE relname = 'wal_test'
)
SELECT
  count(*) AS buffers
FROM pg_buffercache b
JOIN rel r
  ON b.reldatabase = (SELECT oid FROM pg_database WHERE datname = current_database())
 AND b.relfilenode = r.fn
 AND b.relforknumber = 0;

```
-- фрагмент кода
```text
CREATE TABLE
INSERT 0 100000
 pg_relation_size 
------------------
         24100864
(1 row)

 current_setting 
-----------------
            8192
(1 row)

 buffers 
---------
    2942
(1 row)

```

**2. Повышение уровня блокировок: Воспроизвел ситуацию автоматического повышения уровня предикатных блокировок
при чтении строк по индексу. Показал, что это может привести к ложной ошибке сериализации.**
```sql
SELECT count(*) FROM pg_buffercache WHERE isdirty IS TRUE;
CHECKPOINT;
SELECT count(*) FROM pg_buffercache WHERE isdirty IS TRUE;
```
-- фрагмент кода
```text
 count 
-------
  3291
(1 row)

CHECKPOINT

 count 
-------
     0
(1 row)

```

**3. Логирование долгих ожиданий: Настроил запись в журнал сообщений о ожиданиях блокировок > 100 мс (log_lock_waits
= on, deadlock_timeout = 100ms). Создалситуацию длительного ожидания блокировки. Убедил, что сообщение
появилось в логе.**
```sql
CREATE EXTENSION pg_prewarm;
SELECT pg_prewarm('wal_test');
```
```bash
sudo pg_ctlcluster 16 main restart
```
```sql
WITH rel AS (
  SELECT oid, pg_relation_filenode(oid) AS fn
  FROM pg_class WHERE relname = 'wal_test'
)
SELECT count(*) AS buffers
FROM pg_buffercache b
JOIN rel r
  ON b.reldatabase = (SELECT oid FROM pg_database WHERE datname = current_database())
 AND b.relfilenode = r.fn
 AND b.relforknumber = 0;
```
-- фрагмент кода
```text
 pg_prewarm 
------------
       2942
(1 row)
```
```text
 buffers 
---------
       0
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

