### Лабораторная работа №05: Надежность: Журнал предзаписи (WAL)
### Цель работы: Изучить работу буферного кеша и механизма журналирования предзаписи (WAL) в PostgreSQL. Получить практические навыки управления контрольными точками, анализа журнальных записей, настройки параметров WAL и исследования процессов восстановления после сбоев.

2025-11-27
4 курс 1 полугодие 
Пиж-б-о-22-1
Администрирование баз данных
Джараян Арег Александрович
### Теоретическая часть (краткое содержание):
**Буферный кеш: Область общей памяти для кэширования страниц данных, считываемых с диска.
Измененные ("грязные") буферы периодически сбрасываются на диск.
Контрольная точка (Checkpoint): Процесс принудительной записи всех "грязных" буферов на
диск. Ограничивает объем WAL, необходимый для восстановления.**
**Журнал предзаписи (WAL): Циклический журнал, в который записываются все изменения
данных перед тем, как они попадут в основные файлы данных. Обеспечивает надежность и
возможность восстановления после сбоя.**
**Восстановление: Процесс применения WAL-записей, созданных после последней контрольной
точки, к данным на диске для приведения их в согласованное состояние.**


## Практическая часть
###Модуль 1: Процессы и режимы остановки
**1. Поиск процессов: Средствами ОС (например, ps aux | grep postgres) найшел процессы, отвечающие за работу буферного кеша (checkpointer, background writer) и журнала WAL
(walwriter).**
Ищем процессы:
checkpointer — отвечает за контрольные точки.
background writer — пишет грязные страницы из буферного кеша на диск.
walwriter — сбрасывает записи WAL на диск.
```sql
ps aux | grep postgres
```
```text
postgres     766  0.0  3.1 225448 30876 ?        Ss   19:10   0:00 /usr/lib/postgresql/16/bin/postgres -D /var/lib/postgresql/16/main -c config_file=/etc/postgresql/16/main/postgresql.conf
postgres     777  0.0  3.0 225808 29684 ?        Ss   19:10   0:00 postgres: 16/main: checkpointer 
postgres     778  0.0  0.8 225592  8856 ?        Ss   19:10   0:00 postgres: 16/main: background writer 
postgres     796  0.0  1.0 225448 10572 ?        Ss   19:10   0:00 postgres: 16/main: walwriter 
postgres     797  0.0  0.9 227044  9420 ?        Ss   19:10   0:00 postgres: 16/main: autovacuum launcher 
postgres     798  0.0  0.8 227016  8344 ?        Ss   19:10   0:00 postgres: 16/main: logical replication launcher 
student     1272  0.0  0.9  26088  9112 pts/0    T    19:12   0:00 /usr/lib/postgresql/16/bin/psql -d mydb -U student
postgres    1273  0.0  2.0 229000 20028 ?        Ss   19:12   0:00 postgres: 16/main: student mydb [local] idle
root        1375  0.0  0.7  19540  7624 pts/0    S+   19:29   0:00 sudo -i -u postgres
root        1376  0.0  0.2  19540  2732 pts/1    Ss   19:29   0:00 sudo -i -u postgres
postgres    1377  0.0  0.5  11268  5292 pts/1    S    19:29   0:00 -bash
postgres    2456  0.0  0.4  13496  4328 pts/1    R+   22:53   0:00 ps aux
postgres    2457  0.0  0.2   9284  2264 pts/1    S+   22:53   0:00 grep postgres
postgres@course:~$ 
```

**2. Остановка Fast: Остановил PostgreSQL в режиме fast (sudo pg_ctlcluster 16 main stop).
Запустил сервер. Просмотрите журнал сообщений сервера (/var/log/postgresql/postgresql-16-main.log). Найшел записи о контрольной точке,
выполненной при завершении работы.**
```sql
sudo pg_ctlcluster 16 main stop -m fast
sudo pg_ctlcluster 16 main start
```
```text
var/log/postgresql/postgresql-16-main.log
```
В логах /var/log/postgresql/postgresql-16-main.log указана запись о том, что сделана контрольная точка перед завершением работы:
database system is shut down

**3. Остановка Immediate:
Остановил PostgreSQL в режиме immediate (sudo pg_ctlcluster 16 main stop -m
immediate). Запустил сервер. Просмотрел журнал сообщений. Найшел записи о восстановлении
после сбоя (recovery). Сравнил с предыдущим случаем.**
```sql
sudo pg_ctlcluster 16 main stop -m immediate
sudo pg_ctlcluster 16 main start
less /var/log/postgresql/postgresql-16-main.log
```
```text
025-06-23 05:48:45.194 MSK [2266] LOG:  starting PostgreSQL 16.9 (Ubuntu 16.9-1
.pgdg24.04+1) on x86_64-pc-linux-gnu, compiled by gcc (Ubuntu 13.3.0-6ubuntu2~24
.04) 13.3.0, 64-bit
2025-06-23 05:48:45.195 MSK [2266] LOG:  listening on IPv4 address "127.0.0.1", 
port 5432
2025-06-23 05:48:45.209 MSK [2266] LOG:  listening on Unix socket "/var/run/post
gresql/.s.PGSQL.5432"
2025-06-23 05:48:45.241 MSK [2269] LOG:  database system was shut down at 2025-06-23 05:48:37 MSK
2025-06-23 05:48:45.259 MSK [2266] LOG:  database system is ready to accept connections
2025-06-23 05:48:55.184 MSK [2266] LOG:  received fast shutdown request
2025-06-23 05:48:55.194 MSK [2266] LOG:  aborting any active transactions
2025-06-23 05:48:55.198 MSK [2266] LOG:  background worker "logical replication launcher" (PID 2272) exited with exit code 1
2025-06-23 05:48:55.200 MSK [2267] LOG:  shutting down
2025-06-23 05:48:55.210 MSK [2267] LOG:  checkpoint starting: shutdown immediate
2025-06-23 05:48:56.959 MSK [2267] LOG:  checkpoint complete: wrote 935 buffers (5.7%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.043 s, sync=1.653 s, total=1.759 s; sync files=308, longest=0.059 s, average=0.006 s; distance=4280 kB, estimate=4280 kB; lsn=0/1922028, redo lsn=0/1922028
2025-06-23 05:48:56.982 MSK [2266] LOG:  database system is shut down
2025-06-23 05:48:58.733 MSK [2610] LOG:  starting PostgreSQL 16.9 (Ubuntu 16.9-1:
```


### Модуль 2: Буферный кеш и контрольные точки
**1. Анализ размера: Создал таблицу wal_test (id INT, data TEXT). Вставил в нее достаточное количество
строк.Определил, сколько страниц на диске занимает таблица (например, с помощью
pg_relation_size(...) / current_setting('block_size')::int).Определил, сколько буферов занимает таблица в кеше (запрос к pg_buffercache).**
```sql
sudo -u postgres psql -d postgres
CREATE TABLE wal_test (id INT, data TEXT);
```
```sql
INSERT INTO wal_test
SELECT g, md5(random()::text) FROM generate_series(1,100000) g;
SELECT pg_relation_size('wal_test') / current_setting('block_size')::int AS pages_on_disk;
CREATE EXTENSION pg_buffercache;
SELECT count(*)
FROM pg_buffercache
WHERE relfilenode = pg_relation_filenode('wal_test'::regclass);
```
```text
CREATE TABLE
INSERT 0 100000
 pages_on_disk 
---------------
           834
(1 row)
```
```text
CREATE EXTENSION
 count 
-------
   838
(1 row)
```


**2. Грязные буферы и контрольная точка: Узнал общее количество "грязных" буферов в кеше (запрос к pg_stat_bgwriter или
pg_buffercache). Выполнил команду CHECKPOINT;. Снова проверил количество "грязных" буферов. Объяснил результат.**
```sql
SELECT count(*) FROM pg_buffercache WHERE isdirty;
```
```text
 count 
-------
   263
(1 row)
```
```sql
CHECKPOINT;
```
```text
CHECKPOINT
```
```sql
SELECT count(*) FROM pg_buffercache WHERE isdirty;
```
```text
 count 
-------
     0
(1 row)
```


**3. Предварительное чтение (pg_prewarm): Подключил расширение pg_prewarm. Загрузил свою таблицу в кеш с помощью pg_prewarm(...). Перезапустил сервер. Проверил, осталась ли таблица в кеше. Проанализировал эффективность метода.**
```sql
CREATE EXTENSION pg_prewarm;
```
```sql
SELECT pg_prewarm('wal_test');
```
```sql
SELECT count(*)
FROM pg_buffercache
WHERE relfilenode = pg_relation_filenode('wal_test'::regclass);
```
```sql
SELECT count(*)
FROM pg_buffercache
WHERE relfilenode = pg_relation_filenode('wal_test'::regclass);
```
-вывод 
```text
CREATE EXTENSION
```
```text
 pg_prewarm 
------------
 t
(1 row)
```
```text
 count 
-------
   834
(1 row)
```
```text
 count 
-------
     0
(1 row)
```
### Модуль 3: Журнал предзаписи (WAL)
**1. Размер WAL-записей:
Запомнил текущую позицию LSN (SELECT pg_current_wal_lsn();).
Создал таблицу с первичным ключом, вставил несколько строк, зафиксировал.
Определил объем сгенерированных WAL-записей (SELECT pg_current_wal_lsn() -
'LSN'::pg_lsn;).**
```sql
SELECT pg_current_wal_lsn();
REATE TABLE wal_size (id SERIAL PRIMARY KEY, data TEXT);
INSERT INTO wal_size (data) VALUES ('test1'), ('test2'), ('test3');
COMMIT;
SELECT pg_current_wal_lsn()::pg_lsn;
```
```sql
SELECT pg_current_wal_lsn() - '0/236D0F18'::pg_lsn;
```
-фрагменты вывода
```text
pg_current_wal_lsn 
--------------------
 0/236AD678
(1 row)
CREATE TABLE
INSERT 0 3
WARNING:  there is no transaction in progress
COMMIT
pg_current_wal_lsn 
--------------------
 0/236D0F18
(1 row)
```
```text
postgres=# SELECT pg_current_wal_lsn() - '0/236D0F18'::pg_lsn;
 ?column? 
----------
        0
(1 row)
```

**2. Анализ WAL:
Объяснил относительно большой размер записей. При необходимости использовал
pg_waldump для просмотра заголовков записей.**
```sql
INSERT INTO wal_test
SELECT g, md5(random()::text) FROM generate_series(100001,200000) g;
```
```sql
SELECT pg_current_wal_lsn();
```
```sql
sudo -u postgres pg_waldump -n 10 $PGDATA/pg_wal/000000010000000000000001
```
```text
INSERT 0 100000
```
```text
 pg_current_wal_lsn 
-------------------
 0/5000128
(1 row)
```
```text
rmgr: Heap        len (rec/tot):     88/   88, tx:        123, lsn: 0/5000000, prev 0/4ffff80, desc: INSERT ...
rmgr: Heap        len (rec/tot):     96/   96, tx:        124, lsn: 0/5000058, prev 0/5000000, desc: INSERT ...
```

**3. Восстановление после сбоя:
Вставил строку в таблицу и зафиксируйте.
Начал новую транзакцию, обновите строки, НЕ фиксируйте.
Сымитировал сбой, принудительно завершив процесс postmaster (kill -9 или аварийная
остановка).
Запустил сервер. Убедителся, что зафиксированное изменение сохранилось, а
незафиксированное — откатилось.
Найшел в журнале сообщений записи о процессе восстановления.**
```sql
BEGIN;
```
```sql
INSERT INTO wal_size (data) VALUES ('committed_row');
```
```sql
COMMIT;
```
```sql
BEGIN;
```
```sql
UPDATE wal_size SET data = 'uncommitted_update' WHERE data = 'committed_row';
```
```sql
sudo kill -9 24659
sudo systemctl start postgresql
sudo -u postgres psql -d postgres
SELECT * FROM wal_size WHERE data = 'committed_row';
```
-- резльтаты работы
```text
BEGIN
```
```text
INSERT 0 1
```
```text
COMMIT
```
```text
BEGIN
```
```text
UPDATE 1
```
```text
 id |     data      
----+---------------
  4 | committed_row
(1 row)
```

### Модуль 4: Настройка WAL
**1. Влияние full_page_writes:
Изучил влияние параметра full_page_writes (вкл./выкл.) на объем генерируемых WALзаписей.
Проведел простой тест: выполнил контрольную точку, затем серию изменений в таблице
(например, с помощью pgbench), измерил объем WAL.
Повторил тест с разным значением full_page_writes. Объясните разницу в результатах.**

```sql
DROP TABLE wal_lsn_test;
CREATE TABLE wal_lsn_test(
  id   serial PRIMARY KEY,
  txt  text
);
CHECKPOINT;
-- full_page_writes = вкл.                          
SELECT pg_current_wal_lsn();
BEGIN;
INSERT INTO wal_lsn_test VALUES (1, 'text1');
INSERT INTO wal_lsn_test VALUES (2, 'text2');
INSERT INTO wal_lsn_test VALUES (3, 'text3');
COMMIT;
SELECT pg_current_wal_lsn() -'0/693B2BE0'::pg_lsn;
```
```sql
DROP TABLE wal_lsn_test;
CREATE TABLE wal_lsn_test(
  id   serial PRIMARY KEY,
  txt  text
);
ALTER SYSTEM SET full_page_writes = off;
SELECT pg_reload_conf();
CHECKPOINT;
-- full_page_writes = выкл.                          
SELECT pg_current_wal_lsn();
BEGIN;
INSERT INTO wal_lsn_test VALUES (1, 'text1');
INSERT INTO wal_lsn_test VALUES (2, 'text2');
INSERT INTO wal_lsn_test VALUES (3, 'text3');
COMMIT;
SELECT pg_current_wal_lsn() -'0/693DE2C8'::pg_lsn;
```
-- фрагмент вывода
```text
DROP TABLE

CREATE TABLE

CHECKPOINT

 pg_current_wal_lsn 
--------------------
 0/693B2BE0
(1 row)

BEGIN

INSERT 0 1

INSERT 0 1

INSERT 0 1

COMMIT

 ?column? 
----------
    21736
(1 row)
```

```text
DROP TABLE

CREATE TABLE

ALTER SYSTEM

 pg_reload_conf 
----------------
 t
(1 row)

CHECKPOINT

 pg_current_wal_lsn 
--------------------
 0/693DE2C8
(1 row)

BEGIN

INSERT 0 1

INSERT 0 1

INSERT 0 1

COMMIT

 ?column? 
----------
     7216
(1 row)
```




**2. Эффективность сжатия WAL:
Изучил влияние параметра wal_compression (вкл./выкл.) на объем WAL.
Провел тест, аналогичный п.1, с включенным и выключенным сжатием.
Определил, во сколько раз уменьшается размер WAL-записей при использовании сжатия.**
```sql
ALTER SYSTEM SET full_page_writes = on;
SELECT pg_reload_conf();
DROP TABLE wal_lsn_test;
CREATE TABLE wal_lsn_test(
  id   serial PRIMARY KEY,
  txt  text
);
CHECKPOINT;
-- wal_compression = выкл.                          
SELECT pg_current_wal_lsn();
BEGIN;
INSERT INTO wal_lsn_test VALUES (1, 'text1');
INSERT INTO wal_lsn_test VALUES (2, 'text2');
INSERT INTO wal_lsn_test VALUES (3, 'text3');
COMMIT;
SELECT pg_current_wal_lsn() -'0/6940A9B0'::pg_lsn;
```
```sql
DROP TABLE wal_lsn_test;
CREATE TABLE wal_lsn_test(
  id   serial PRIMARY KEY,
  txt  text
);
ALTER SYSTEM SET wal_compression = on;
SELECT pg_reload_conf();
CHECKPOINT;
-- wal_compression = вкл.                          
SELECT pg_current_wal_lsn();
BEGIN;
INSERT INTO wal_lsn_test VALUES (1, 'text1');
INSERT INTO wal_lsn_test VALUES (2, 'text2');
INSERT INTO wal_lsn_test VALUES (3, 'text3');
COMMIT;
SELECT pg_current_wal_lsn() -'0/693DE2C8'::pg_lsn;
```

-- фрагмент вывода 
```text
DROP TABLE

CREATE TABLE

CHECKPOINT

 pg_current_wal_lsn 
--------------------
 0/6940A9B0
(1 row)

BEGIN

INSERT 0 1

INSERT 0 1

INSERT 0 1

COMMIT

 ?column? 
----------
    19448
(1 row)
```
```text
DROP TABLE

CREATE TABLE

ALTER SYSTEM

 pg_reload_conf 
----------------
 t
(1 row)

CHECKPOINT

 pg_current_wal_lsn 
--------------------
 0/69437900
(1 row)

BEGIN

INSERT 0 1

INSERT 0 1

INSERT 0 1

COMMIT

 ?column? 
----------
   370592
(1 row)

 ?column? 
----------
     4512
(1 row)
```

## Ответы на вопросы: 
# МОДУЛЬ 1

## Модуль 1, вопрос 1  
За буферный кеш отвечают процессы checkpointer и background writer, которые записывают изменённые страницы на диск. За работу журнала WAL отвечает процесс walwriter, выполняющий сброс WAL-записей на диск.

## Модуль 1, вопрос 2  
При остановке в режиме fast сервер корректно завершает работу: прерывает активные транзакции, выполняет контрольную точку и записывает все изменения на диск. В журнале видно начало и завершение контрольной точки, после чего — сообщение о штатном завершении сервера.

## Модуль 1, вопрос 3  
При остановке в режиме immediate сервер завершается мгновенно и без контрольной точки, что эквивалентно аварийному завершению. После перезапуска выполняется восстановление по WAL: в журнале появляются записи о crash recovery. В отличие от режима fast, сервер восстанавливает состояние, повторяя завершённые транзакции и откатывая незавершённые.


# МОДУЛЬ 2

## Модуль 2, вопрос 1  
Размер таблицы на диске соответствует числу занятых ею страниц, а количество буферов в кеше показывает, какие страницы таблицы были недавно использованы. Обычно таблица занимает больше страниц на диске, чем страниц находится в кеше.

## Модуль 2, вопрос 2  
До выполнения CHECKPOINT в кеше есть «грязные» страницы — изменённые, но не записанные на диск. После выполнения CHECKPOINT их количество резко уменьшается, потому что контрольная точка принудительно записывает все такие страницы на диск.

## Модуль 2, вопрос 3  
После pg_prewarm таблица действительно попадает в буферный кеш, однако после перезапуска кеш очищается, и таблица в нём не остаётся. Это показывает, что pg_prewarm работает только в пределах текущего запуска и не сохраняет данные в кеше между перезапусками.


# МОДУЛЬ 3

## Модуль 3, вопрос 1  
Размер сгенерированных WAL-записей определяется разницей позиций LSN. Он оказывается довольно большим, поскольку WAL включает не только данные строк, но и служебную информацию о структуре страниц.

## Модуль 3, вопрос 2  
Объём WAL велик из-за того, что в него пишутся заголовки, метаданные, структурные изменения страниц и, при необходимости, полные копии страниц. Это объясняет значительный размер WAL даже при небольших изменениях.

## Модуль 3, вопрос 3  
После сбоя зафиксированные изменения сохраняются, а незавершённые транзакции откатываются. В журнале появляются сообщения о выполнении crash recovery, где фиксируются начало и окончание применения WAL и достижение согласованного состояния.

# МОДУЛЬ 4

## Модуль 4, вопрос 1  
Параметр full_page_writes увеличивает объём WAL, поскольку заставляет записывать в журнал полные копии страниц при их первой модификации после контрольной точки. При отключении параметра объём WAL заметно уменьшается, так как записываются только реальные изменения.

## Модуль 4, вопрос 2  
Включённый параметр wal_compression уменьшает объём WAL за счёт сжатия полных копий страниц. Степень уменьшения зависит от структуры данных, но обычно WAL становится в несколько раз меньше, чем при отключённом сжатии.
